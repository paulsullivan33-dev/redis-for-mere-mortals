# Memory and Eviction

An in-memory store has exactly one
hard wall: the size of memory. Redis
lets you set the wall yourself with
`maxmemory`, and — the interesting
decision — tell it what to do when
writes arrive at a full server. That
decision is the **eviction policy**.
The lab ran both halves of it on a
separate instance with a 5 MB ceiling,
small enough to hit in seconds.

## The ceiling, and the default refusal

```
$ redis-cli -p 6399 CONFIG GET maxmemory
maxmemory
5242880

$ redis-cli -p 6399 CONFIG GET maxmemory-policy
maxmemory-policy
noeviction

$ redis-cli -p 6399 INFO memory | grep -E '^(used_memory_human|maxmemory_human)'
used_memory_human:645.15K
maxmemory_human:5.00M
```

An empty server already uses 645 KB
of its 5 MB on itself — structures,
buffers, the machinery. `maxmemory 0`,
the lab's main setting, means "no
ceiling but the machine's".

Now the fill: sixty writes of a
100 KB value each, under the default
policy, `noeviction`:

```
$ for i in $(seq 1 60); do redis-cli -p 6399 SET fill:$i "$BIG"; done
OK
... (37 more OKs, then:)
OOM command not allowed when used memory > 'maxmemory'.
... (21 more refusals)

$ redis-cli -p 6399 DBSIZE
38

$ redis-cli -p 6399 INFO stats | grep '^evicted_keys'
evicted_keys:0
```

The count tells the story: 38 writes
landed (each a little over 100 KB,
plus the server's own 645 KB), and
then the server started *refusing* —
22 `OOM` errors, one per remaining
write. `noeviction` means exactly
what it says: nothing is thrown away,
and new writes that need memory fail
instead. (Reads and deletes still
work at the wall; only growth is
refused.) Usage levelled at 4.86 MB
of the 5.00 MB — the server keeps a
little headroom rather than spending
the last byte.

**Check:** the arithmetic. 38 keys in,
evicted_keys still 0 — under
noeviction the failure is pushed back
onto *your writes*, where you can see
it. That is the right default for
data you care about, and a terrible
one for a cache, where a failed write
is a disaster and a forgotten entry
is a shrug.

## Eviction switched on

`CONFIG SET` changes a running
server. Turning the policy to
`allkeys-lru` — when space is needed,
evict the **least recently used** key,
any key — and writing the same twenty
keys again:

```
$ redis-cli -p 6399 CONFIG SET maxmemory-policy allkeys-lru
OK

$ for i in $(seq 61 80); do redis-cli -p 6399 SET fill:$i "$BIG"; done
OK
... (all twenty)

$ redis-cli -p 6399 INFO stats | grep '^evicted_keys'
evicted_keys:20

$ redis-cli -p 6399 DBSIZE
38

$ redis-cli -p 6399 INFO memory | grep -E '^(used_memory_human|maxmemory_human)'
used_memory_human:4.88M
maxmemory_human:5.00M
```

Every write succeeded — and exactly
20 old keys are gone, one eviction
per new arrival, the population
steady at 38, memory steady at the
wall. The eviction counter in `INFO
stats` is the instrument to watch on
a real server: climbing
`evicted_keys` on an instance you
believe is a database means the
server is quietly deleting your data
to stay alive. On a cache it's the
sound of the thing working.

The policy family, briefly:
`allkeys-*` can evict anything;
`volatile-*` may only evict keys that
have an expiry set (safer neighbours
for important keys, useless if
nothing expires); `-lru`, `-lfu`
(least *frequently* used), `-random`,
and `-ttl` (shortest-lived first)
pick the victim. The choice is a
statement about which data is
expendable.

## CONFIG SET forgets; CONFIG REWRITE remembers

Everything above was runtime-only.
Proof, the restarting way — set a
third policy, read it back, restart,
read again:

```
$ redis-cli -p 6399 CONFIG SET maxmemory-policy allkeys-random
OK

$ redis-cli -p 6399 CONFIG GET maxmemory-policy
maxmemory-policy
allkeys-random

$ redis-cli -p 6399 SHUTDOWN NOSAVE
$ redis-server ~/workspace/redis-lab/redis-evict.conf

$ redis-cli -p 6399 CONFIG GET maxmemory-policy
maxmemory-policy
noeviction
```

Gone. The restart re-read the config
*file*, which still says noeviction.
To make a runtime change survive,
`CONFIG REWRITE` writes the running
values back into the file the
instance started from:

```
$ redis-cli -p 6399 CONFIG SET maxmemory-policy allkeys-lru
OK

$ redis-cli -p 6399 CONFIG REWRITE
OK

$ grep -E '^maxmemory-policy' ~/workspace/redis-lab/redis-evict.conf
maxmemory-policy allkeys-lru

$ redis-cli -p 6399 SHUTDOWN NOSAVE
$ redis-server ~/workspace/redis-lab/redis-evict.conf

$ redis-cli -p 6399 CONFIG GET maxmemory-policy
maxmemory-policy
allkeys-lru
```

**Check:** after the rewrite and
restart, the policy answers
`allkeys-lru` — carried by the file,
not by memory. Keep the two verbs
separate in your head: `CONFIG SET`
is a conversation with the running
server; `CONFIG REWRITE` is editing
its config file by proxy. Plenty of
production settings — `maxmemory`,
`appendonly`, save points — are
commonly SET at runtime during an
incident and just as commonly lost
at the next restart because nobody
ran the second command.
