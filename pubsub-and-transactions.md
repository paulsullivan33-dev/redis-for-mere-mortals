# Pub/Sub and Transactions

Two features that aren't about storing
at all. **Pub/sub** is Redis as a
postman: clients subscribe to channels,
anyone can publish to a channel, and
whoever is listening gets the message.
**Transactions** are Redis keeping a
promise about a *group* of commands.

## Publishing to whoever is listening

The lab ran two clients. One
subscribed and waited (its output
saved to a file):

```
$ redis-cli -p 6399 SUBSCRIBE seedlib-news > pubsub-capture.txt 2>&1 &
$ sleep 1
```

The other published. `PUBLISH` answers
with the number of subscribers the
message was delivered to:

```
$ redis-cli -p 6399 PUBLISH seedlib-news "Spring seed swap: Saturday 10am, community hall"
1

$ redis-cli -p 6399 PUBLISH seedlib-news "Tomato seedlings ready for pickup at the desk"
1

$ redis-cli -p 6399 PUBSUB NUMSUB seedlib-news
seedlib-news
1
```

And the listening client's file,
read afterwards:

```
$ cat ~/workspace/redis-lab/pubsub-capture.txt
subscribe
seedlib-news
1
message
seedlib-news
Spring seed swap: Saturday 10am, community hall
message
seedlib-news
Tomato seedlings ready for pickup at the desk
```

One subscription confirmation and
both messages, in order. Now the
honest half — a channel nobody hears:

```
$ redis-cli -p 6399 PUBLISH seedlib-gossip "unheard"
0
```

Zero. Pub/sub is **fire and forget**:
nothing is stored, nothing is queued
for later, a subscriber that connects
a second after publication missed the
message forever. That sounds like a
flaw and is, deliberately, the design
— pub/sub is for "tell whoever is
listening *right now*" (presence,
live updates, chat fan-out), not for
work that must not be lost. Work that
must not be lost belongs in chapter
5's lists, where it waits to be
picked up instead of evaporating.

**Check:** `PUBLISH` to a channel
with no subscribers answers 0 and
the message is unrecoverable — the
number *is* the delivery report.

## Transactions: MULTI and EXEC

A transaction groups commands so they
run together, with nothing from
another client interleaved. `MULTI`
starts collecting; every command
after it is answered `QUEUED` instead
of being run; `EXEC` runs the whole
queue:

```
$ redis-cli -p 6399 <<'EOF'
MULTI
SET seedlib:tx:counter 100
INCR seedlib:tx:counter
INCR seedlib:tx:counter
EXEC
EOF
OK
QUEUED
QUEUED
QUEUED
OK
101
102

$ redis-cli -p 6399 GET seedlib:tx:counter
102
```

(The commands were piped into one
`redis-cli` session — a transaction
lives on one connection, so the
one-shot form used elsewhere in this
guide can't hold one open.) EXEC's
reply is the queue's replies in
order: the SET's `OK`, then the two
new counter values.

One honesty note: a Redis transaction
is *not* a database transaction.
There's no rollback — if one command
in the queue fails at EXEC time (say
an `INCR` on words), the others still
run. What MULTI buys is *atomicity of
the group*: all of it, in order, with
nobody else's commands in between.

## WATCH: the optimistic lock

Atomicity of a group doesn't cover
the read-decide-write pattern: "read
the register, then set it based on
what I read" breaks if someone else
writes in between. `WATCH` covers it:
watch a key, and if it changes before
your `EXEC`, the transaction aborts
instead of clobbering.

The lab held a watching session open
on a fifo so a second client could
meddle mid-transaction:

```
(watching session)  WATCH seedlib:watch:register   → OK
(second client)     INCR seedlib:watch:register    → 11
(watching session)  MULTI                          → OK
(watching session)  SET seedlib:watch:register 999 → QUEUED
(watching session)  EXEC                           → (nothing)
```

The watching session's own capture
shows the replies in a row — `OK`,
`OK`, `QUEUED`, then a blank line
where EXEC's answer should be:

```
$ redis-cli -p 6399 GET seedlib:watch:register
11
```

The blank is the verdict: EXEC
returned **nil**, the transaction was
discarded, and the register still
holds the other client's 11 — not
the 999 the watcher tried to base on
its stale read of 10. The pattern is
called optimistic because it assumes
collisions are rare: no lock is held
while you think; if you lose the
race, you read the new value and
retry. For counters and registers
under real contention, chapter 2's
atomic `INCR` is usually the better
answer — WATCH is for when the
decision genuinely depends on the
value read.
