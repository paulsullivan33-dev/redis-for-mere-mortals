# KEYS or SCAN

Sooner or later you want to ask "what
keys do I have that look like *this*?"
Redis offers two answers, and the
difference between them is one of the
few production rules worth memorising:
`KEYS` hands you everything at once;
`SCAN` walks you through it in
batches. The lab planted 5,000 keys
so both could be watched working:

```
$ python3 - <<'EOF' | redis-cli -p 6399 --pipe
...
EOF
All data transferred. Waiting for the last reply...
Last reply received from server.
errors: 0, replies: 5000

$ redis-cli -p 6399 DBSIZE
5007
```

(The generator writes Redis's wire
protocol directly into `redis-cli
--pipe`, which is the fast way to load
data in bulk — 5,000 `SET`s in one
stream, zero errors. The count checks
out: 5,000 new keys on top of the 7
the earlier chapters left behind.)

## KEYS: everything, in one breath

```
$ { time redis-cli -p 6399 KEYS 'bulk:*' | wc -l; } 2>&1
5000

real	0m0.009s
```

Nine milliseconds — nothing, at this
size. And that is exactly the trap.
`KEYS` builds the *complete* answer
before replying, and Redis serves
everyone from a single thread: while
one `KEYS` call assembles a list of
five *million* keys, every other
client waits its turn behind it. The
command is not slow because it is
badly written; it is slow at scale
because it is thorough in one go,
on a server that does one thing at
a time.

## SCAN: a cursor and small answers

`SCAN` trades the single answer for
an iterator — a **cursor** you carry
from call to call:

```
$ redis-cli -p 6399 SCAN 0 MATCH 'bulk:*' COUNT 5
1024
bulk:04393
bulk:01796
bulk:01140
bulk:03325
bulk:04743
bulk:01095
bulk:02230

$ redis-cli -p 6399 SCAN 1024 MATCH 'bulk:*' COUNT 5
4608
bulk:00850
bulk:00472
bulk:02953
bulk:04425
bulk:04349
```

Read the replies as: "start at 0;
here is the *next* cursor (1024) and
a batch of keys; call again with that
cursor; a cursor of 0 means the walk
is finished." Each call does a small,
bounded piece of work and returns, so
other clients are never locked out
for long.

`COUNT 5` is a **hint**, not a
promise — the first call asked for 5
and returned 7 keys. The server
examines roughly that many buckets
and reports whatever matches it finds
in them, which can also mean a batch
of *none* with a nonzero cursor. A
correct SCAN loop must therefore keep
going until the cursor returns to 0,
and must tolerate empty batches and
the occasional duplicate (a full
walk guarantees finding every key
that existed throughout the walk —
it does not guarantee seeing each one
exactly once).

`redis-cli` will do the whole walk
for you:

```
$ { time redis-cli -p 6399 --scan --pattern 'bulk:*' | wc -l; } 2>&1
5000

real	0m0.028s
```

Slower than `KEYS` here — 28 ms
against 9 — because it makes many
small calls instead of one big one.
That is the correct trade on a live
server: your key listing costs a
little more, and nobody else's
request ever waits behind it.

## The rule

- **`KEYS`** in a lab, in a script
  against a disposable instance, in a
  debugging session where you *know*
  the keyspace is small — anywhere the
  blocking can't hurt anyone.
- **`SCAN`** everywhere else. Any
  code that runs against a production
  instance uses SCAN, full stop; most
  client libraries expose it directly,
  often named `scan_iter` or similar.

And the cleanup, to leave the lab as
it was found:

```
$ redis-cli -p 6399 FLUSHDB
OK

$ redis-cli -p 6399 DBSIZE
0
```

**Check:** your walk counts 5,000
keys both ways. If `--scan` reports a
slightly different total on a busier
instance, that's the semantics above
— keys arriving and expiring mid-walk
are counted as the walk finds them.
