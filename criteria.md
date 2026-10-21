# Writing Criteria

Every guide in this repo is written to the same
standard. If you're adding or editing a guide —
human or AI — hold it to these.

1. **Plain words, real ideas.** Simple language,
   never dumbed-down concepts. Explain jargon on
   first use; collect terms in the glossary.
2. **No hype.** No marketing adjectives, no
   "revolutionary," no promises about what
   something will do for the reader.
3. **Honest about limits.** Say what the thing
   *can't* do. State uncertainty instead of
   smoothing it over. Validation notes include
   what remains unverified.
4. **No invented numbers.** Benchmarks, defaults,
   and specs are either measured or qualified with
   "depends on your setup." If it wasn't tested,
   say so.
5. **Checkable.** Commands are meant to be run,
   with "Check:" blocks showing what success looks
   like. Sources are linked, not asserted.
6. **Respect the reader's setup.** Call out
   Windows/Mac/Linux differences. Don't assume a
   GPU, a budget, or prior experience.
7. **Safety isn't skipped.** Real risks get
   stated plainly, not buried.
8. **Scoped.** Each guide says what it covers and
   points elsewhere for the rest. No guide
   pretends to be the whole subject.
9. **Structure over prose.** Tables, before/after
   examples, symptom → fix. Scannable beats essay.

## Verification notes for this guide

Standard 3 and 4 applied to this guide,
specifically. Redis **7.0.15** (Ubuntu package
`5:7.0.15-1ubuntu0.24.04.5`, `malloc=jemalloc-5.3.0`)
was installed fresh on the build machine with
`apt-get install -y redis-server`, and every
quoted output is a capture from it. Working
transcripts are kept with the build
(`redis-lab/transcripts/`, 14 numbered files).

**The lab.** One throwaway instance per phase,
each started from an explicit config file in
the lab folder, each on port **6399**,
loopback-bound, with its own data directory
under `redis-lab/`. The shipped configuration
(`examples/redis-lab.conf`) is the main
instance's, byte for byte. The package also
installed a system service
(`redis-server.service`, on the standard port
6379); the build sandbox's policy-rc.d refused
to start it at install time, and the guide
never started, stopped, or reconfigured it —
at the end of the build it stands exactly as
the package left it (enabled, inactive,
nothing listening on 6379), verified in
transcript 14. Lab instances were each shut
down inside their own transcripts; transcript
14 confirms port 6399 free and no lab
redis-server process remaining.

**The fixtures.** A fictional community seed
library: stock counters, member records, a
door queue, two club rosters, a garden-hours
leaderboard. Names (Ada Osei, Ruth Adler, Tom
Okafor, Priya Nair, Sam Whitfield) are
invented; the only credential is the lab's
invented runtime password, cleared again in
its own transcript.

**Verified by running, per chapter:**

- *Ch. 1:* the install (package already
  present; `apt-get` re-run captured saying
  so), `--version` for server and client,
  the package's system service status and its
  shipped `/etc/redis/redis.conf` bind/port
  lines, the lab instance started from
  `examples/redis-lab.conf`, `PING` → `PONG`,
  the startup log's "Ready to accept
  connections", `INFO server` and `INFO
  keyspace` (empty), and `CONFIG GET`
  read-backs of port, bind, dir,
  maxmemory-policy, save, and appendonly.
- *Ch. 2:* SET/GET on named keys; the
  counter sequence 40 → `INCR` 41 →
  `INCRBY` 51 → `DECR` 50; `TYPE` → string;
  `STRLEN` 28 on the cached page; `EXISTS`
  1/0; `DEL` answering 1 then `EXISTS` 0;
  `DBSIZE` 3; and `INCR` on a word-valued
  key failing with `ERR value is not an
  integer or out of range`.
- *Ch. 3:* a key set with `EX 5`, `TTL` 5,
  `PTTL` 4989, then a real `sleep 6` after
  which `GET` answers blank and `EXISTS` 0;
  TTL's −2 (gone) and −1 (no expiry);
  `PERSIST` removing a 60-second deadline
  (TTL → −1) and `EXPIRE` setting 120.
- *Ch. 4:* `HSET` answering 3 new fields,
  `HGETALL`, `HGET`, `TYPE` → hash,
  `HINCRBY` moving loans 2 → 3, a fourth
  field added (HSET 1), `HLEN` 4,
  `HEXISTS` 1, `HDEL` 1, `HEXISTS` 0.
- *Ch. 5:* three `LPUSH`es with lengths
  1/2/3, `LRANGE` head-first order (latest
  arrival at the head), `RPOP` serving the
  first arrival, `BRPOP` answering at once
  with its two-part reply — and, on the
  drained queue, blocking the full allowed
  second and answering blank, `LLEN` 0.
- *Ch. 6:* `SADD` 4 and 3, a duplicate add
  answering 0, `SCARD`, `SISMEMBER` 1/0,
  `SINTER` (exactly 2), `SUNION` (5,
  unordered as captured), `SDIFF`;
  `ZADD` 5 members, `ZRANGE ...
  WITHSCORES` ascending, `ZREVRANGE 0 2`
  the top three (21.25/15/12.5),
  `ZREVRANK` 2 (zero-based), `ZSCORE`,
  `ZINCRBY` moving 4.5 → 6.5 with the top
  three provably unchanged.
- *Ch. 7:* 5,000 keys loaded via
  `redis-cli --pipe` (generator and replies
  captured: `errors: 0, replies: 5000`),
  `DBSIZE` 5007; `KEYS` returning 5,000
  lines in 0.009 s; `SCAN 0 ... COUNT 5`
  returning cursor 1024 and *7* keys
  (COUNT is a hint, in the upward
  direction that run — the chapter says
  "smaller or larger" because both occur),
  the cursor followed to 4608; `--scan`
  completing the walk at 5,000 lines in
  0.028 s; `FLUSHDB` returning the
  keyspace to 0. Times are this
  keyspace on this machine, quoted as
  such; the production rule rests on
  Redis's single-threaded design, which
  is documented behaviour, stated as such.
- *Ch. 8 (the centerpiece):* three fresh
  instances, identical writes (two SETs,
  three INCRs), full `SHUTDOWN NOSAVE`
  and restart each time. **RDB:** empty
  directory before, `BGSAVE`, `dump.rdb`
  at 172 bytes, restart, all three keys
  back, `DBSIZE` 3. **AOF:** save points
  cleared (`CONFIG GET save` empty) so
  only the journal could restore;
  `appendonlydir` held base 89 B, incr
  234 B, manifest 88 B; restart, all
  three keys back, `DBSIZE` 3. **Off:**
  directory empty throughout; restart,
  `GET` blank on both probed keys,
  `DBSIZE` 0.
- *Ch. 9:* a 5 MB instance read back at
  5242880 bytes, 645.15K baseline use;
  sixty 100 KB writes under `noeviction`
  — 38 landed (`DBSIZE` 38), 22 refused
  with the `OOM` error verbatim, usage
  4.86M, `evicted_keys` 0; policy changed
  at runtime to `allkeys-lru`, twenty
  more writes all answered OK,
  `evicted_keys` 20, `DBSIZE` still 38,
  usage 4.88M; then the survival arc —
  a runtime `CONFIG SET` to
  `allkeys-random` read back, restarted,
  and gone (config file's `noeviction`
  in force), versus `CONFIG SET` +
  `CONFIG REWRITE`, the config file's
  line shown changed by grep, restarted,
  and in force.
- *Ch. 10:* two-client pub/sub — a
  background subscriber's captured file
  shows the subscribe confirmation and
  both published messages verbatim;
  `PUBLISH` answered 1, 1, and 0 (the
  third to a channel with no listeners);
  `PUBSUB NUMSUB` 1. MULTI/EXEC piped
  into one session: `OK`, three
  `QUEUED`, then EXEC's replies
  `OK`/101/102, counter read back at
  102. WATCH: a session held open on a
  fifo while a second client moved the
  watched key 10 → 11; the watcher's
  EXEC answered blank (nil) and the key
  read back 11, not the attempted 999.
- *Ch. 11:* `CONFIG GET requirepass`
  empty; runtime `CONFIG SET`; `PING`
  refused with `NOAUTH Authentication
  required.`; `AUTH wrong-password`
  refused with the `WRONGPASS` line
  verbatim; `-a` with the lab password
  answered `PONG`; the password then
  cleared (using it), `PING` answering
  again — and the instance shut down in
  the same transcript, its port refusing
  connections.
- *Ch. 13:* every symptom in the
  troubleshooting table is one the lab
  produced (connection refused, post-
  restart absence, OOM refusals,
  NOAUTH/WRONGPASS) or the direct
  neighbour of one it produced
  (crossed queue ends, wrong-`dir`
  lookups, slow KEYS); the fixes are
  the commands the lab itself used.

**Capture hygiene (declared trims):**
repeated homogeneous lines in three
captures are elided with `...` and a
count (the persistence INCRs, the
fill-loop OKs and refusals — refusals
counted from the transcript: 22); the
`$` command lines show shell quoting
restored where the capture harness's
echo dropped it (a quoted value in a
command line); no output line was
altered, and every number quoted in a
chapter was read off the transcripts,
not the harness scripts. Two
transcripts carry a stray empty
comment line from the harness's note
helper; no command output is affected.
One mid-build incident, for the record:
a `pkill -f` in the pub/sub harness
matched the supervising shell's own
command line and ended that shell
after its transcripts were complete;
both transcripts were verified whole
before the build continued.

**Documented-only (not run), and why:**

- *Replication, Sentinel, Cluster* —
  each needs several cooperating
  instances; the lab's discipline is
  one throwaway instance per phase.
  Named in chapter 12, with their jobs
  stated, and not a command quoted.
- *Streams* (acknowledged queues),
  *modules*, *managed services* — named
  where they belong, not taught.
- *TLS and per-user ACLs* — named in
  chapter 11 as the grown-up step;
  the lab's security story is bind,
  protected mode, and requirepass,
  all captured.
- *`appendfsync always`* — named as
  the paranoid end of the fsync
  choice; the lab ran the default
  `everysec`, and chapter 8 says the
  ~1-second figure as the documented
  consequence of that default.
- *Hash field expiry* (per-field TTL)
  — exists only in Redis 7.4+; the
  lab's 7.0.15 does not have it, and
  chapter 4 says so rather than
  demonstrating.
- *Installation on Windows/macOS* —
  described (WSL/Docker, Homebrew),
  not performed; the build machine
  installed the Ubuntu package.
- *The licensing facts* in chapter 12
  (March 2024 dual-license change,
  Valkey's BSD fork from 7.2.4 under
  the Linux Foundation) are stated as
  dated public facts; they are not
  lab-verifiable and are not presented
  as lab results.
