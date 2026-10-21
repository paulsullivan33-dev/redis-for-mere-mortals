# Not Your Database

Every chapter so far has added a
caveat, and they collect into one
sentence worth reading before you
build on Redis: **Redis is a superb
fast layer and a reluctant system of
record.** This chapter is the caveats,
gathered and stated as plainly as the
lab allows.

## What the lab itself proved

These aren't quotations from anyone's
documentation — they're the results
in the transcripts:

- **Restart amnesia is a
  configuration, not a bug.** Chapter
  8, part 3: with persistence off, a
  restart returned `DBSIZE 0`. With
  RDB, the loss window on a crash is
  "everything since the last
  snapshot". With AOF at its default,
  it's about a second. Choose per
  workload, in writing, before the
  data matters.
- **Memory is the ceiling, and the
  ceiling has a bouncer.** Chapter 9:
  at `maxmemory`, either writes fail
  (`noeviction`) or old keys vanish
  (`evicted_keys: 20` after 20 writes).
  A store whose size is bounded by
  RAM cannot be where the *only*
  copy of a growing archive lives.
- **No queries.** There is no `WHERE`.
  You find data by knowing its key,
  or by walking the keyspace with
  SCAN (chapter 7) — which means
  Redis answers "fetch this" orders of
  magnitude better than "find all
  members who joined before March and
  have more than two loans". That
  sentence is one SQL line.
- **Expiry is deletion.** Chapter 3's
  hold disappeared without a trace or
  a confirmation. Any key with a TTL
  is data you have agreed to lose on
  schedule.

None of that is an argument against
Redis. It's the argument for the
usual shape: a database holds the
truth; Redis holds the *hot copy*,
the counters, the queue, the cache —
everything whose loss is an
inconvenience rather than a loss.

## Which store when

The shelf's neighbour is [SQLite for
Mere Mortals](https://paulsullivan33-dev.github.io/sqlite-for-mere-mortals/):
an embedded relational database in a
file, no server at all. Between them,
most small systems are covered:

| The need | Redis | SQLite |
|---|---|---|
| counters, rate limits | `INCR` — one atomic step | a transaction, per click |
| cache with expiry | native TTL | roll your own timestamps |
| a work queue | lists + `BRPOP` | polling a table |
| leaderboards | sorted sets | `ORDER BY` on an index (fine, less instant) |
| data that must survive anything | only with AOF, and care | yes — that's the job |
| questions not planned in advance | no | SQL |
| runs inside your process | no, it's a server | yes, it's a library |

The pattern in real systems is both:
SQLite (or PostgreSQL, or files) as
the record, Redis in front for the
reads and writes that happen a
thousand times a minute.

## The license, and Valkey

Stated as facts, because the names
now matter when you install things:

- Redis was BSD-licensed for most of
  its life. In **March 2024**, from
  version 7.4 onward, Redis Ltd.
  relicensed it under a dual license —
  the Redis Source Available License
  (RSALv2) and the Server Side Public
  License (SSPLv1). It is *source
  available*, and no longer open
  source in the OSI sense. For
  running your own Redis, nothing in
  this guide changes; the license
  bites people *offering Redis as a
  service*.
- **Valkey** is the community fork of
  the last BSD code (Redis 7.2.4),
  started the same month, hosted by
  the Linux Foundation, and backed by
  several of the large cloud vendors.
  It is BSD-licensed and, at this
  writing, command-compatible with
  everything in this guide.
- The lab's binary is genuine Redis
  **7.0.15**, the Ubuntu package —
  which predates the relicensed
  versions entirely. What *your*
  package manager installs under the
  name "redis" varies by distribution
  and by year now (some ship Valkey
  behind the same package name);
  `redis-server --version` on your
  machine is the only answer that
  counts, which is chapter 1's first
  command for a reason.

## Named, not run

Three larger topics, placed honestly:

- **Replication** — one server copies
  another, for read scale and
  failover standby.
- **Sentinel** — watchers that
  promote a replica when the primary
  dies, so failover is automatic.
- **Cluster** — sharding the keyspace
  across many servers, when one
  machine's memory and thread are
  genuinely not enough.

Each deserves its own lab with
several instances; none fits the
one-throwaway-instance discipline of
this guide, and pretending otherwise
would produce the worst kind of
chapter — commands quoted from
someone else's run. If you outgrow
one instance, you'll meet them in the
project documentation with real
motivation, which is the right time.
