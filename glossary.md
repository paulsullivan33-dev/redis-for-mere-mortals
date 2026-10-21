# Glossary

**ACL** — Access control list.
Redis 6 and later: named users with
their own passwords and limits on
which keys and commands they may
touch. The grown-up version of
`requirepass`.

**AOF** — Append-only file. The
journal form of persistence: every
write is recorded, and a restart
replays the journal. In Redis 7 it
lives in an `appendonlydir`
directory — a base snapshot, an
incremental journal, and a manifest.

**atomic** — Done as one indivisible
step. Other clients' commands cannot
slip into the middle of an atomic
operation, which is what makes
`INCR` safe as a shared counter.

**bind** — The configuration naming
which network interfaces the server
answers on. The lab's `127.0.0.1
-::1` means "this machine only".

**blocking command** — A command
that waits for its answer instead of
replying at once. `BRPOP` with a
timeout is the guide's example: it
parks the client until a list has
something to give, or time runs out.

**client** — Anything connected to
the server sending commands:
`redis-cli`, or a library inside
your program.

**cursor** — SCAN's place-marker.
Each call returns a batch of keys
and the cursor for the next call;
a returned cursor of 0 ends the
walk.

**dataset / keyspace** — Everything
the server currently holds: all
keys, in all databases. `DBSIZE`
counts the current database's share.

**eviction** — Deleting keys to stay
under `maxmemory`, under the rules
of the eviction *policy*. The
counter `evicted_keys` in `INFO
stats` counts them.

**expiry** — A key's deadline. When
it passes, the server deletes the
key itself. Set with `EX`, `EXPIRE`,
and friends; read with `TTL`.

**fork** — How the server saves
without stopping: the process clones
itself, and the child writes the
snapshot while the parent keeps
serving. `BGSAVE`'s BG is this.

**hash** — A key holding field →
value pairs: one small record.

**journal** — See AOF.

**key** — The name every value is
stored and reached by. Keys are
strings; the colon-separated naming
(`app:kind:id`) is convention, not
server rules.

**list** — A key holding an ordered
run of strings, pushable and
poppable at both ends. Push at one
end and pop from the other and it's
a queue.

**LRU / LFU** — Eviction tastes.
Least *recently* used evicts what
hasn't been touched longest; least
*frequently* used evicts what is
seldom touched at all.

**maxmemory** — The server's memory
ceiling in bytes (0 = none). What
happens at the ceiling is the
eviction policy's decision.

**nil** — Redis's "nothing there"
reply: an empty answer. `redis-cli`
prints it as a blank line in one-shot
mode and `(nil)` interactively.

**noeviction** — The default
eviction policy: at `maxmemory`,
refuse new writes rather than delete
anything.

**persistence** — The arrangement
for what survives the process
stopping: RDB snapshots, AOF
journal, both, or — legitimately,
for pure caches — neither.

**protected mode** — The backstop:
a server with no password refuses
non-loopback connections even if its
bind would allow them.

**pub/sub** — Publish/subscribe
messaging. Fire-and-forget: a
message reaches the subscribers
connected at that moment, and is
not stored.

**RDB** — The snapshot form of
persistence: the whole dataset in
one file (`dump.rdb`), written on a
schedule or by `BGSAVE`.

**replication** — One server copying
another continuously, for read
scale and standby. Named in this
guide, not run.

**requirepass** — The single shared
password a server demands before
any other command (`AUTH`).

**SCAN** — The cursor-based way to
walk the keyspace in batches,
without blocking the server.
`KEYS`'s production-safe sibling.

**set** — A key holding unique
strings in no order: a roster, with
intersection/union/difference
commands.

**sorted set** — A key holding
unique strings, each with a
**score**; the order is the score,
so it reads as a leaderboard.

**TTL** — Time to live: the seconds
a key has left before its expiry.
Also used loosely for the deadline
itself.

**Valkey** — The BSD-licensed fork
of Redis 7.2.4, created in 2024
under the Linux Foundation after
Redis's relicensing. Command-
compatible with this guide's lab.
