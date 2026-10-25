# Redis for Mere Mortals

**Redis** is a data store that keeps
everything in memory and answers in
microseconds: counters, queues, caches,
leaderboards, and little records, all
behind a few dozen plain commands you
type at it. Programs use it as the fast
layer in front of slower storage — the
session store, the cache, the work
queue, the scoreboard — anywhere being
*quick* matters more than being
*complete*.

That trade is this guide's spine. Redis
is fast because it lives in RAM, and
everything in RAM is one power cut away
from gone unless the server is told,
carefully, how to write things down.
Chapter 8 runs that experiment three
ways on the record: one restart where
the data comes back from a snapshot,
one where it comes back from a journal,
and one where it simply isn't there.
Read those results before you decide
what to trust Redis with.

Everything here is practised on a
throwaway lab instance: Redis
**7.0.15**, installed from the Ubuntu
package, started from the config in
`examples/` on the nonstandard port
**6399** with its own scratch directory,
so it never touches a real install on
the standard port. The lab's data is a
fictional community seed library —
stock counters, member records, a door
queue, club rosters, a garden-hours
leaderboard. Every command quoted was
run against that instance and captured;
`criteria.md` keeps the ledger,
including the eviction counts, the
persistence file sizes, and what was
only described, not run.

## What this guide covers

- [**What Redis is**](what-redis-is.md) —
  an in-memory store, where it fits
  (and the SQLite question), installing
  it, and starting the lab instance
- [**Keys and strings**](keys-and-strings.md) —
  SET, GET, counters with INCR, key
  naming, EXISTS and DEL
- [**Expiry**](expiry.md) — keys that
  remove themselves: EX, TTL, PERSIST,
  and an expiry watched in real time
- [**Hashes**](hashes.md) — one key, a
  whole record: the member card
- [**Lists and queues**](lists-and-queues.md) —
  LPUSH plus RPOP as a first-in-first-out
  line, and BRPOP, the pop that waits
- [**Sets and sorted sets**](sets-and-sorted-sets.md) —
  unique members, clubs in common, and
  a leaderboard with scores
- [**KEYS or SCAN**](keys-or-scan.md) —
  finding keys without stopping the
  server: 5,000 planted keys, both
  commands timed
- [**Persistence**](persistence.md) —
  the centerpiece: RDB snapshots, the
  AOF journal, and persistence off,
  each restarted and read back
- [**Memory and eviction**](memory-and-eviction.md) —
  a 5 MB ceiling, writes refused under
  `noeviction`, an eviction counter
  climbing under LRU, and CONFIG SET
  versus CONFIG REWRITE
- [**Pub/sub and transactions**](pubsub-and-transactions.md) —
  messages to whoever is listening,
  MULTI/EXEC, and a WATCH that aborts
- [**Security basics**](security.md) —
  loopback binding, protected mode,
  a password set at runtime, and why
  port 6379 stays off the internet
- [**Not your database**](not-your-database.md) —
  the honest framing: durability
  limits, the 2024 licensing change,
  Valkey, and which store when
- [**Troubleshooting**](troubleshooting.md) —
  symptom → fix for the failures the
  lab produced, and their neighbours
- [**Glossary**](glossary.md) — the
  terms, collected in one place

## What it doesn't cover

Replication, Sentinel, and Cluster —
running several Redis servers as one
system — are named in chapter 12 and
left there: the lab is one instance,
deliberately. Modules (search, JSON,
and friends), Redis as a vector store,
and the managed services (ElastiCache,
MemoryStore, Azure Cache) are likewise
named, not taught. Programming against
Redis from Python, Node, or anything
else belongs to those languages'
guides; here every command is typed
at `redis-cli`, which is where
understanding starts anyway.

## How to use this guide

Read chapters 1–3 in order; they stand
up the lab and the two ideas everything
else leans on (keys are typed values,
and keys can expire). After that,
chapters stand alone — go to the data
type your problem is shaped like. Keep
the lab instance on its own port and
in its own directory, and when a
chapter says **Check:**, run the
command: in this guide the checks are
the content, and most of them read a
number back out of the server and
compare it with what the chapter
promised.

If your data has to survive and be
queryable in complicated ways, you
probably want a database first and
Redis in front of it — the sibling
guide [SQLite for Mere
Mortals](https://paulsullivan33-dev.github.io/sqlite-for-mere-mortals/)
covers the embedded end of that
choice, and chapter 12 here draws the
line between the two.

---

## Part of the series

This guide is one of the [For Mere Mortals](https://paulsullivan33-dev.github.io/mere-mortals/) guides: plain-language guides to the tools programmers and makers use.

Related guides:

- [PostgreSQL for Mere Mortals](https://paulsullivan33-dev.github.io/postgresql-for-mere-mortals/)
- [Docker for Mere Mortals](https://paulsullivan33-dev.github.io/docker-for-mere-mortals/)
- [SQLite for Mere Mortals](https://paulsullivan33-dev.github.io/sqlite-for-mere-mortals/)

## License

The writing in this guide is licensed under [CC BY 4.0](https://github.com/paulsullivan33-dev/redis-for-mere-mortals/blob/main/LICENSE). Code examples and lab scripts are licensed under the [MIT License](https://github.com/paulsullivan33-dev/redis-for-mere-mortals/blob/main/LICENSE-CODE).
