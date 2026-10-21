# Persistence

This is the chapter the whole guide
leans on, so it is run the strictest
way: the same writes, three different
configurations, a full shutdown and
restart each time, and the read-back
as the only verdict.

First, the mental model. Redis keeps
data in memory. **Persistence** is the
arrangement for what exists after the
process stops — crash, upgrade, power
cut, tidy shutdown, all the same to
the data. Redis offers two mechanisms,
and a third option: neither.

- **RDB** — snapshots. Every so often
  (or on demand), the whole dataset is
  written to one file, `dump.rdb`.
- **AOF** — a journal. Every write is
  appended to a log; on restart, the
  log is replayed.
- **Off** — memory only. Restart is a
  blank server.

The lab's main instance read back its
own settings in chapter 1: RDB save
points in force, AOF off. Each
experiment below gets a fresh
instance, its own directory and
config variant, on the same lab port,
one at a time.

## Part 1: RDB — the data comes back from a snapshot

The same three writes each time:

```
$ redis-cli -p 6399 SET seedlib:stock:tomato "Cherokee Purple"
OK

$ redis-cli -p 6399 SET seedlib:stock:beans 40
OK

$ redis-cli -p 6399 INCR visit:counter
1
... (twice more: 2, 3)
```

The instance's directory, before any
save happens:

```
$ ls -la ~/workspace/redis-lab/data-rdb/
total 0
```

Empty. The data exists only in the
running process. `BGSAVE` takes the
snapshot on demand (the BG is
"background": the server forks, the
child writes the file, service never
pauses visibly):

```
$ redis-cli -p 6399 BGSAVE
Background saving started

$ ls -la ~/workspace/redis-lab/data-rdb/
total 8
...
-rw-rw---- 1 root nogroup 172 Oct 21 01:16 dump.rdb
```

172 bytes holds all three keys. Now
the honest test — stop the server
completely, start it again on the
same directory:

```
$ redis-cli -p 6399 SHUTDOWN NOSAVE
$ redis-server ~/workspace/redis-lab/redis-rdb.conf

$ redis-cli -p 6399 GET seedlib:stock:tomato
Cherokee Purple

$ redis-cli -p 6399 GET seedlib:stock:beans
40

$ redis-cli -p 6399 GET visit:counter
3

$ redis-cli -p 6399 DBSIZE
3
```

Data back, all of it. (The `NOSAVE`
on SHUTDOWN matters here: it proves
the rescue came from the file written
by `BGSAVE`, not from a save quietly
taken on the way down. A plain
shutdown of the lab's main instance
in chapter 11 behaves the same way.)

What RDB costs you is the gap: a
snapshot is a moment. The automatic
save points from the config —
`900 1 300 10 60 10000` reads as "save
if 1 key changed in 900 seconds, or
10 in 300, or 10,000 in 60" — mean a
crash between snapshots loses
everything written since the last
one. RDB answers "restart, upgrade,
planned stop" perfectly, and "sudden
death" with *up to the interval*.

## Part 2: AOF — the data comes back from a journal

This variant clears the save points
entirely (`save ""`) and turns the
journal on — so anything that
survives can only have come from the
AOF:

```
$ redis-cli -p 6399 CONFIG GET appendonly
appendonly
yes

$ redis-cli -p 6399 CONFIG GET save
save

```

Note the empty `save`. Same writes,
then a look at the directory *before*
any restart:

```
$ ls -la ~/workspace/redis-lab/data-aof/appendonlydir/
...
-rw-rw---- 1 root nogroup  89 Oct 21 01:16 appendonly.aof.1.base.rdb
-rw-r----- 1 root nogroup 234 Oct 21 01:16 appendonly.aof.1.incr.aof
-rw-r----- 1 root nogroup  88 Oct 21 01:16 appendonly.aof.manifest
```

Modern AOF is a small directory, not
one file: a *base* (a snapshot in RDB
format, here 89 bytes — the starting
point), an *incremental* journal
(234 bytes — the five writes from
this experiment, as commands), and a
manifest keeping the order. Restart:

```
$ redis-cli -p 6399 SHUTDOWN NOSAVE
$ redis-server ~/workspace/redis-lab/redis-aof.conf

$ redis-cli -p 6399 GET seedlib:stock:tomato
Cherokee Purple

$ redis-cli -p 6399 GET visit:counter
3

$ redis-cli -p 6399 DBSIZE
3
```

Data back again — replayed from the
journal. The gap is now tiny: the
default `appendfsync everysec` policy
flushes the journal once a second, so
a sudden death loses at most about a
second of writes. In exchange the
journal does more disk work than an
occasional snapshot, and the
base-plus-increment dance (called a
*rewrite*, done automatically) exists
to keep it from growing forever.

Most real deployments run AOF, RDB,
or both together — with both, a
restart loads the AOF, as the fuller
record. The choice is not "safe or
unsafe"; it's which gap you accept.

## Part 3: persistence off — the control

No save points, no AOF. Same writes.
The directory *while the server holds
the data*:

```
$ ls -la ~/workspace/redis-lab/data-none/
total 0
```

Nothing, anywhere. Restart:

```
$ redis-cli -p 6399 SHUTDOWN NOSAVE
$ redis-server ~/workspace/redis-lab/redis-none.conf

$ redis-cli -p 6399 GET seedlib:stock:tomato


$ redis-cli -p 6399 GET visit:counter


$ redis-cli -p 6399 DBSIZE
0
```

The blank `GET`s are chapter 2's
"not there", and now they mean it
permanently. Nothing failed; nothing
errored. The server did exactly what
it was configured to do, which was:
remember nothing.

## The verdict table

| Configuration | After restart | What a crash loses |
|---|---|---|
| RDB (snapshots) | data back, from the last snapshot | everything since the last snapshot |
| AOF (journal) | data back, replayed | ~1 second of writes (default fsync) |
| off | **gone, DBSIZE 0** | everything |

**Check:** run the third configuration
yourself before you trust the table.
The point of this chapter is not the
table; it's that "Redis lost my data"
is almost always "Redis was never
asked to keep it" — and the asking is
two lines of configuration you can
read back with `CONFIG GET` at any
time.
