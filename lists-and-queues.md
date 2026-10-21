# Lists and Queues

A **list** is an ordered run of
strings under one key — ordered by
insertion, with a head and a tail you
can both push to and pop from. With
several producers and consumers that
one structure becomes Redis's most
famous job: the **queue**.

The rule that makes a queue out of a
list is: *push at one end, pop from
the other*. The lab pushes at the
head (`LPUSH`) and serves from the
tail (`RPOP`), which makes the line
first-in-first-out.

## The door queue

Saturday morning at the seed library,
three people join the line for the
seed cabinet:

```
$ redis-cli -p 6399 LPUSH door-queue "Ada Osei"
1

$ redis-cli -p 6399 LPUSH door-queue "Ruth Adler"
2

$ redis-cli -p 6399 LPUSH door-queue "Tom Okafor"
3

$ redis-cli -p 6399 LRANGE door-queue 0 -1
Tom Okafor
Ruth Adler
Ada Osei

$ redis-cli -p 6399 LLEN door-queue
3
```

Each `LPUSH` answers with the length
after the push. `LRANGE 0 -1` reads
the whole list head-first — which is
why it looks backwards: the head is
the *most recent* arrival, Tom. The
tail is Ada, who arrived first, and
the tail is where service happens:

```
$ redis-cli -p 6399 RPOP door-queue
Ada Osei

$ redis-cli -p 6399 LRANGE door-queue 0 -1
Tom Okafor
Ruth Adler
```

**Check:** Ada is served first,
although she is last in the `LRANGE`
listing. If your line ever serves Tom
first, the ends are crossed — pushes
and pops are meeting at the same end,
which is a stack, not a queue.

(Push and pop from *both* ends are all
real commands — `RPUSH`, `LPOP` — and
there are jobs for each combination.
The queue is just the polite one.)

## BRPOP: the pop that waits

A worker process sitting on a queue
shouldn't spin asking "anything
yet?" — Redis has the blocking form:

```
$ redis-cli -p 6399 BRPOP door-queue 1
door-queue
Ruth Adler
```

The `1` is a timeout in seconds.
Because Ruth was already waiting in
line, `BRPOP` answered at once — with
a two-part reply: the queue's name,
then the value. Draining Tom the same
way leaves the queue empty, and then
the timeout shows its other face:

```
$ redis-cli -p 6399 BRPOP door-queue 1


$ redis-cli -p 6399 LLEN door-queue
0
```

The command blocked for the full
second it was allowed, then answered
with *nothing* — the blank lines are
the empty reply. A timeout of **0**
means wait forever, and that is the
normal setting for a real worker: it
sleeps inside the server's answer
until work exists, using no CPU at
all, and wakes the moment a producer
pushes. That is the entire trick
behind "background jobs" in a great
many web applications — a list, some
workers parked in `BRPOP`, and
producers who never need to know the
workers exist.

## Queue, honestly

A list queue is simple and, for most
small systems, enough. Its limits,
stated plainly: a popped job is
*gone* — if the worker dies mid-job,
nothing returns the job to the line
(the polite pattern is to move jobs
to a "processing" list and remove
them when done; Redis's own answer at
scale is **Streams**, a heavier
structure with acknowledgements,
named here and not taught). And a
list holds strings — a job is usually
a small JSON document or an ID, with
the real data kept elsewhere.
