# What Redis Is

Redis is a **data store**: a program
that holds data so other programs can
ask for it quickly. Two properties
explain everything else about it.

First, *all* of it lives in memory.
There is no disk in the read path at
all — a value you ask for is already
in RAM, which is why answers come back
in microseconds. Second, it speaks in
**commands**, not queries: you don't
write anything like SQL. You say `SET`,
`GET`, `INCR`, `LPUSH`, and each
command works on a **key** — a name —
holding one typed value.

That second property is the real
difference. A database table holds
rows, all the same shape. A Redis key
holds one *thing*: a string, a small
record, a list, a set, a scored
ranking. Chapters 2–7 are mostly a tour
of those shapes and the few jobs each
one is perfect for: the counter, the
cache entry, the queue, the roster,
the leaderboard.

Where it fits, honestly: in *front*
of slower storage, holding the data
that is asked for constantly or that
is allowed to be temporary — sessions,
cached pages, rate-limit counters,
work queues, scores. Chapter 12 is a
full chapter on what it should *not*
be trusted with, because holding
things in memory is also its limit:
memory is finite (chapter 9) and
forgetful unless told otherwise
(chapter 8).

A word about the name you'll meet in
chapter 12: in 2024 Redis changed its
license, and a fork named **Valkey**
split off under the Linux Foundation.
For everything in this guide the two
are interchangeable — same commands,
same lineage. The lab runs genuine
Redis, as its version banner says
below.

## Installing it

On Debian or Ubuntu, Redis is one
package. This also installs
`redis-cli`, the command-line client
every chapter uses:

```
$ apt-get install -y redis-server
...
redis-server is already the newest version (5:7.0.15-1ubuntu0.24.04.5).
```

(The build machine had already
installed it; on a fresh machine apt
fetches and installs it in the same
step. On macOS, `brew install redis`;
on Windows, Redis runs under WSL or
Docker — there is no native Windows
build from the project itself.)

```
$ redis-server --version
Redis server v=7.0.15 sha=00000000:0 malloc=jemalloc-5.3.0 bits=64 build=d064f9f48c901a92

$ redis-cli --version
redis-cli 7.0.15
```

**Check:** your version strings print.
The exact version matters less than
this guide's habit: every output
quoted was captured from *this*
version, and `criteria.md` says so.

## The system service — leave it alone

The package also installs a **system
service** definition, the setup a real
deployment would use:

```
$ systemctl status redis-server --no-pager
○ redis-server.service - Advanced key-value store
     Loaded: loaded (/usr/lib/systemd/system/redis-server.service; enabled; preset: enabled)
     Active: inactive (dead)
```

```
$ grep -E '^(port|bind|protected-mode|daemonize)' /etc/redis/redis.conf
bind 127.0.0.1 -::1
protected-mode yes
port 6379
daemonize yes
```

On the build machine the package's
service was never started (the
sandbox's policy blocked it, and this
guide doesn't start it either). That
standard configuration — port **6379** —
belongs to real installs, so the lab
never touches it. The lab is a
*separate, throwaway instance* with:

- its own config file, shipped as
  `examples/redis-lab.conf`
- a nonstandard port, **6399**
- its own scratch directory for any
  files it writes
- loopback-only binding, like the
  package default

If you already run Redis for real on a
machine, build the lab on a different
one, or keep the port separation this
guide uses — two instances, two ports,
two directories never meet.

## Starting the lab instance

One command, naming the config file.
The config says `daemonize yes`, so
the server starts in the background
and the command returns at once:

```
$ redis-server ~/workspace/redis-lab/redis-lab.conf

$ redis-cli -p 6399 PING
PONG
```

`PING` is Redis's smallest command:
the server answers `PONG`. Everything
in this guide is commands like it,
sent by `redis-cli` — the `-p 6399`
says which port (without it, the
client assumes 6379 and would talk to
somebody else's server, or nobody).

The server's own log (the config names
a log file in the lab folder) confirms
what it's ready for:

```
$ tail -3 ~/workspace/redis-lab/redis-lab.log
...
* Ready to accept connections
```

**Check:** `PING` answers `PONG`. If
you get `Could not connect`, the
instance isn't running — start it
again and read the log; the usual
causes are a typo'd path in the config
or another lab instance already
holding the port.

## What the server says about itself

`INFO` is the server's self-report, in
named sections. Two of them, at the
very start of the lab:

```
$ redis-cli -p 6399 INFO server
# Server
redis_version:7.0.15
...
redis_mode:standalone
os:Linux 7.0.0-39-generic x86_64
...
process_id:21798
...
tcp_port:6399
...
config_file:/home/hatch/workspace/redis-lab/redis-lab.conf
```

```
$ redis-cli -p 6399 INFO keyspace
# Keyspace
```

An empty keyspace section is the
correct report for a new instance:
no databases, no keys. Later chapters
read other `INFO` sections — `memory`
and `stats` in chapter 9 carry the
eviction story — and the habit to
build is: when you want to know
something about the server, ask the
server.

## Reading the configuration back

Config files are intentions; `CONFIG
GET` is the truth in force. The lab
instance, read back:

```
$ redis-cli -p 6399 CONFIG GET port
port
6399

$ redis-cli -p 6399 CONFIG GET bind
bind
127.0.0.1 -::1

$ redis-cli -p 6399 CONFIG GET dir
dir
/home/hatch/workspace/redis-lab/data-main

$ redis-cli -p 6399 CONFIG GET maxmemory-policy
maxmemory-policy
noeviction

$ redis-cli -p 6399 CONFIG GET save
save
900 1 300 10 60 10000

$ redis-cli -p 6399 CONFIG GET appendonly
appendonly
no

$ redis-cli -p 6399 DBSIZE
0
```

Three of those answers — the `save`
points, `appendonly no`, and
`noeviction` — are whole chapters
later (8 and 9). For now, notice the
shape: `CONFIG GET` answers with the
setting's name and value on separate
lines, and it accepts patterns, so
`CONFIG GET maxmemory*` fetches the
whole memory family at once.

**Check:** `DBSIZE` answers `0` and
`CONFIG GET port` answers `6399`. Your
lab is running, isolated, and empty —
the next chapter puts something in it.
