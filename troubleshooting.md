# Troubleshooting

Symptom → cause → fix, built from the
failures the lab produced on purpose
and the ones it produced by being a
lab. When a symptom involves the
server's state, the fix column starts
with asking the server — `INFO`,
`CONFIG GET` — before changing
anything.

## Could not connect

```
$ redis-cli -p 6399 PING
Could not connect to Redis at 127.0.0.1:6399: Connection refused
```

**Cause:** nothing is listening on
that port — the instance isn't
running, or you're asking the wrong
port. (`redis-cli` without `-p` asks
6379, which in this guide is
deliberately nobody.)

**Fix:** check for a listener
(`ss -ltn`), start the instance from
its config file, and read the log
file the config names — a failed
start almost always explains itself
there (a missing directory, a bad
config line, another instance
already bound to the port).

## Everything vanished after a restart

**Cause:** persistence, chapter 8.
Either it was off (`save ""`,
`appendonly no` — and chapter 8's
control experiment ended at
`DBSIZE 0`), or it was on and the
restart read a different `dir` than
the old process wrote to.

**Fix:** on the *running* instance,
`CONFIG GET save`, `CONFIG GET
appendonly`, and `CONFIG GET dir` —
the three answers say what the next
restart will do. Then read
`INFO persistence` for the last
save's time and status. Prevention
is chapter 8's verdict table, applied
before the data matters.

## OOM command not allowed

```
OOM command not allowed when used memory > 'maxmemory'.
```

**Cause:** the server is at its
`maxmemory` ceiling and the policy
is `noeviction` — chapter 9's
22-refusal run.

**Fix:** three legitimate shapes.
Raise the ceiling (`CONFIG SET
maxmemory`, and REWRITE it if it must
survive), free space deliberately
(expire or delete keys), or accept
eviction and set a policy —
`CONFIG GET maxmemory-policy` says
what's in force. What not to do:
restart and hope. The data refills
to the same wall.

## evicted_keys keeps climbing on data I care about

**Cause:** an eviction policy is on
and the dataset wants more room than
`maxmemory` gives it. The counter is
in `INFO stats`; chapter 9 watched
it go 0 → 20 while exactly 20 keys
were quietly deleted.

**Fix:** this instance is acting as
a cache while holding records. Give
it memory, or move the records to an
instance (or a database) whose policy
is `noeviction`. There is no setting
where eviction happens "only to
unimportant keys" unless the
important ones are elsewhere or
everything expendable carries the
expiry that `volatile-*` policies
limit eviction to.

## NOAUTH Authentication required

**Cause:** the server has a
`requirepass` and this client didn't
present it — chapter 11's exact
error, including after a password was
set at runtime and half the team
wasn't told.

**Fix:** `redis-cli -a <password>`
(or `REDISCLI_AUTH` in the
environment). If *nobody* knows the
password: whoever administers the
server can read or reset it through
the config file and a restart — and
an unexpected `requirepass` on a
server you own is worth one paranoid
minute: check `CONFIG GET bind` too,
and ask how it got there.

## WRONGPASS

```
WRONGPASS invalid username-password pair or user is disabled.
```

**Cause:** the password was wrong —
or, on Redis 6+, the username was
(`AUTH` alone means user `default`).
Typos in passwords pasted with
trailing spaces are the classic.

**Fix:** retype it, check for the
user, move on. It is not a lockout;
Redis counts nothing and forgives
retries.

## A queue's worker never wakes

**Cause:** ends crossed (chapter 5):
producers `LPUSH` and the worker
`BLPOP`s from the *same* end, so it
competes with the newest arrival
instead of taking the oldest — or
the worker blocks on a differently
named key than producers write
(check with `LLEN` on both spellings
while a job is pushed by hand).

**Fix:** one end in, other end out,
same key name, proven by pushing a
known string and watching the worker
take it.

## KEYS is slow / the server "hangs" briefly

**Cause:** someone ran `KEYS` (or
another everything-at-once command)
against a large keyspace on a
single-threaded server — chapter 7's
rule, broken.

**Fix:** find the caller (the
`MONITOR` command will show live
traffic on a quiet moment, and
`INFO commandstats` shows which
commands accumulate time) and move
it to `SCAN`. On the instance itself,
`SLOWLOG GET` names recent slow
commands with their durations.

## The save or the journal file isn't where I look

**Cause:** `dir` is a *server*
setting, not a shell one — the
instance writes `dump.rdb` and
`appendonlydir` into the directory
from its config, which for the lab
was under `redis-lab/`, not the
directory you happened to `cd` into.

**Fix:** `CONFIG GET dir` and
`CONFIG GET dbfilename` on the
running server; look there.
