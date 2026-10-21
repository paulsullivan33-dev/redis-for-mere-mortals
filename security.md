# Security Basics

Redis was designed to run inside a
trusted network, behind other
software, and its security settings
all make sense once you accept that
starting point: Redis's own defences
are a fence, not a fortress, and the
fence only works if you don't open
the gate to the whole internet. This
chapter shows the fence as the lab
ran it — and every setting in it was
read back from the running server,
not just written in a file.

## Who can connect: bind and protected mode

From chapter 1, the running values:

```
$ redis-cli -p 6399 CONFIG GET bind
bind
127.0.0.1 -::1

$ redis-cli -p 6399 CONFIG GET protected-mode
protected-mode
yes
```

`bind` lists the network interfaces
the server answers on. `127.0.0.1`
and `-::1` are the loopbacks — this
server can only be reached *from this
machine*. The package's own
`/etc/redis/redis.conf` ships with
the same two values, plus
`protected-mode yes` as a backstop:
if a server has no password *and* is
somehow reachable from outside,
protected mode refuses the outside
connections anyway. Defence in depth,
two independent settings both saying
"locals only".

## The password: requirepass

A server with no password lets any
client that can reach it do
*anything* — read, write, `FLUSHALL`,
reconfigure. `requirepass` is the
single shared password. The lab set
one at runtime on the main instance
(an invented password, like every
credential in this guide):

```
$ redis-cli -p 6399 CONFIG GET requirepass
requirepass


$ redis-cli -p 6399 CONFIG SET requirepass garden-gate-421
OK

$ redis-cli -p 6399 PING
NOAUTH Authentication required.

$ redis-cli -p 6399 AUTH wrong-password
WRONGPASS invalid username-password pair or user is disabled.

$ redis-cli -p 6399 -a garden-gate-421 --no-auth-warning PING
PONG
```

The empty line after the first
`CONFIG GET` is the "no password"
state. After the `CONFIG SET`, every
unauthenticated command gets `NOAUTH`;
a wrong password is refused with
`WRONGPASS`; and the right one —
supplied by the client with `-a` —
works. (The `--no-auth-warning` flag
just silences redis-cli's complaint
about passing a password on a command
line, where `ps` can see it. In
scripts, prefer the `REDISCLI_AUTH`
environment variable, which keeps it
out of the process list.)

Removing it again took the password
itself, as it should:

```
$ redis-cli -p 6399 -a garden-gate-421 --no-auth-warning CONFIG SET requirepass ""
OK

$ redis-cli -p 6399 PING
PONG
```

Remember chapter 9's lesson while
you're here: that `CONFIG SET` lasted
exactly as long as the process did.
A password that must survive restarts
belongs in the config file (or needs
a `CONFIG REWRITE`).

## The rule that outranks the settings

All of the above is the fence. The
rule is simpler and has no command:

**Never expose a Redis port to the
internet.** Not 6379, not a
renumbered one — port scanners find
Redis by what it says, not by its
number, and an unauthenticated,
internet-facing Redis is routinely
compromised within hours of
appearing: it can write files
(`CONFIG SET dir` plus a snapshot is
a known road to writing SSH keys and
cron jobs, which is another reason
the CONFIG family deserves respect).
The safe shapes, in order:

1. **Loopback binding** (the lab's) —
   applications on the same machine
   only.
2. **A private network** — bind to
   the LAN or VPN interface, password
   set, and the host firewall agreeing
   (the sibling guide [UFW for Mere
   Mortals](https://paulsullivan33-dev.github.io/ufw-for-mere-mortals/)
   covers the firewall half).
3. **TLS and ACLs** — Redis 6 and
   later add per-user ACLs (named
   users with passwords and per-key,
   per-command limits) and optional
   TLS. That is the grown-up version
   of this chapter: worth it when
   several applications share one
   server and shouldn't share all of
   its keys. Named here, not run in
   the lab.

**Check:** your lab instance answers
`CONFIG GET bind` with loopback only,
and if you set the runtime password,
`PING` without it answers `NOAUTH`.
If either check surprises you on a
server you own, chapter 13 has the
symptom.
