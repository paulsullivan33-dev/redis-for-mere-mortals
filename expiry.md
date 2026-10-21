# Expiry

A key in Redis can carry a deadline.
When the deadline passes, the server
removes the key — not when someone
next looks, though it will also be
gone by then, but actively, on its
own schedule. Expiry is the feature
that makes Redis safe for the data
that is only true *for a while*: a
hold, a session, a cache entry, a
rate-limit window.

The lab proves it the only convincing
way: by waiting.

## A key with five seconds to live

```
$ redis-cli -p 6399 SET seedlib:hold:copy-441 "Ada Osei" EX 5
OK

$ redis-cli -p 6399 TTL seedlib:hold:copy-441
5

$ redis-cli -p 6399 PTTL seedlib:hold:copy-441
4989
```

`EX 5` sets the value and its deadline
in one command. `TTL` reports the
remaining seconds; `PTTL` reports
remaining *milli*seconds — 4,989 of
them when the lab asked, because a
little time had already passed. Then:

```
$ sleep 6

$ redis-cli -p 6399 GET seedlib:hold:copy-441


$ redis-cli -p 6399 EXISTS seedlib:hold:copy-441
0
```

Six seconds later the hold is gone —
the blank `GET` and the 0 say so — and
nothing deleted it. There is no trash
can and no undo; expiry is a deletion
the server performs on schedule.

**Check:** set your own 5-second key
and race it: `GET` before the deadline
answers the value, after it, the
blank.

## TTL's three special answers

`TTL` has a small vocabulary of
negative numbers, and all three
appeared in the lab:

```
$ redis-cli -p 6399 TTL seedlib:hold:copy-441
-2

$ redis-cli -p 6399 TTL seedlib:stock:beans
-1
```

- **a positive number** — seconds left
- **−1** — the key exists and has *no*
  expiry (the beans counter will live
  until deleted or the server forgets
  it some harder way; see chapter 8)
- **−2** — the key does not exist

Memorise which is which before you
write a script that branches on TTL;
−1 and −2 read similar at a glance
and mean opposite situations.

## Adding and removing deadlines

`EXPIRE` puts a deadline on a key that
already exists (answering 1 if it
did), and `PERSIST` takes one off:

```
$ redis-cli -p 6399 SET seedlib:hold:copy-207 "Ruth Adler" EX 60
OK

$ redis-cli -p 6399 TTL seedlib:hold:copy-207
60

$ redis-cli -p 6399 PERSIST seedlib:hold:copy-207
1

$ redis-cli -p 6399 TTL seedlib:hold:copy-207
-1

$ redis-cli -p 6399 EXPIRE seedlib:hold:copy-207 120
1

$ redis-cli -p 6399 TTL seedlib:hold:copy-207
120
```

One caution, from the semantics rather
than the lab: setting a *new value*
with plain `SET` clears any deadline
the key had. A program that refreshes
a cached value and forgets to re-apply
the expiry has silently made the entry
permanent. (`SET` has a `KEEPTTL`
option for exactly this; look it up
when you need it.)

## Where expiry earns its keep

- **Holds and reservations** — like
  the library's: claimed now, released
  automatically if never collected.
- **Sessions** — a login token that
  should die in an hour dies in an
  hour, not whenever someone remembers
  to sweep.
- **Cache entries** — chapter 2's
  cached page with an `EX 300` would
  refresh itself by absence: when it
  expires, the next requester rebuilds
  and re-caches it.
- **Rate limiting** — a counter key
  (`INCR`, chapter 2) whose window is
  its TTL: when the key dies, the
  window resets.

Every one of those is "data whose
staleness is a bug" — expiry converts
a cleanup chore into a property of
the data itself.
