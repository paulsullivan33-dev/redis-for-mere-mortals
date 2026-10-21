# Keys and Strings

Everything in Redis is reached by a
**key**: a name, made of characters,
that stands for one value. The
simplest value is a **string** — a run
of text or bytes, up to 512 MB, though
you will meet few past a few kilobytes.
Strings carry more of the real world
than you'd expect: a cached page, a
session token, a name — and, because a
string can hold digits, every counter
you've ever seen on a website.

The lab's subject is a fictional
community seed library. Its keys are
named the way working Redis keys are
named everywhere: **colon-separated**,
most general first — application,
kind, identifier:

```
seedlib:stock:tomato
seedlib:stock:beans
seedlib:cache:events-page
```

Colons are a convention, not a rule
of the server — Redis neither knows
nor cares — but the convention is what
keeps 10,000 keys readable, and it
groups a family under one prefix,
which chapter 7 leans on.

## Setting and getting

```
$ redis-cli -p 6399 SET seedlib:stock:tomato "Cherokee Purple"
OK

$ redis-cli -p 6399 SET seedlib:stock:beans 40
OK

$ redis-cli -p 6399 GET seedlib:stock:tomato
Cherokee Purple

$ redis-cli -p 6399 GET seedlib:stock:beans
40
```

`SET` answers `OK`; `GET` answers the
value, exactly as stored. `SET` on an
existing key replaces the value
without ceremony — there is no "row
already exists" to argue with.

## Counters are strings that hold numbers

```
$ redis-cli -p 6399 INCR seedlib:stock:beans
41

$ redis-cli -p 6399 INCRBY seedlib:stock:beans 10
51

$ redis-cli -p 6399 DECR seedlib:stock:beans
50

$ redis-cli -p 6399 GET seedlib:stock:beans
50
```

`INCR` adds one, `INCRBY` adds an
amount, `DECR` subtracts one, and
each answers with the *new* value. The
operation is **atomic** — the server
does it in one indivisible step — which
is why counters live here: a hundred
programs incrementing the same key
cannot lose each other's updates the
way read-add-write in application code
can. There is no separate counter
type; ask the server what the key is:

```
$ redis-cli -p 6399 TYPE seedlib:stock:beans
string
```

The flip side arrives the first time
the digits aren't digits:

```
$ redis-cli -p 6399 INCR seedlib:stock:tomato
ERR value is not an integer or out of range
```

**Check:** `INCR` on the tomato key
fails with exactly that error, and the
tomato value is unharmed. Redis types
are not column constraints — the check
happens when a command meets a value
it can't use.

## A string can be a page

Counters suggest numbers, but the
classic string job is the **cache
entry**: something expensive rendered
once, then served verbatim:

```
$ redis-cli -p 6399 SET seedlib:cache:events-page "<h1>Seed Library Events</h1>"
OK

$ redis-cli -p 6399 STRLEN seedlib:cache:events-page
28
```

`STRLEN` reports the length without
fetching the value — the first of
many commands that answer a question
*about* a value. Cached entries are
also the natural home of expiry, which
is the whole next chapter: a cache
that never expires is just a second
database you're afraid to clear.

## Asking after keys

```
$ redis-cli -p 6399 EXISTS seedlib:stock:tomato
1

$ redis-cli -p 6399 EXISTS seedlib:stock:corn
0

$ redis-cli -p 6399 SET seedlib:tmp:scratch "to be deleted"
OK

$ redis-cli -p 6399 DEL seedlib:tmp:scratch
1

$ redis-cli -p 6399 EXISTS seedlib:tmp:scratch
0

$ redis-cli -p 6399 DBSIZE
3
```

`EXISTS` answers 1 or 0. `DEL` removes
a key and answers with how many keys
it actually removed — deleting a ghost
answers 0, which makes scripts honest.
And `GET` on a key that isn't there
answers with nothing at all: in the
interactive client you'll see `(nil)`;
in the one-shot form used here, an
empty line. Remember that blank —
chapter 3 and chapter 8 both end with
keys answering it.

**Check:** `DBSIZE` counts the keys in
the current database (tomato, beans,
the cached page — 3). Deleting a key
twice answers 1, then 0.
