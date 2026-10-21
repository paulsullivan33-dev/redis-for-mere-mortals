# Sets and Sorted Sets

Two structures that answer questions
about *membership*. A **set** holds
unique strings in no order — a roster.
A **sorted set** holds unique strings
too, but each carries a **score**, and
the order is the score — a
leaderboard.

## Sets: who is in the club

```
$ redis-cli -p 6399 SADD club:seed-savers "Ada Osei" "Ruth Adler" "Tom Okafor" "Priya Nair"
4

$ redis-cli -p 6399 SADD club:tool-library "Ruth Adler" "Tom Okafor" "Sam Whitfield"
3

$ redis-cli -p 6399 SADD club:seed-savers "Ada Osei"
0

$ redis-cli -p 6399 SCARD club:seed-savers
4

$ redis-cli -p 6399 SISMEMBER club:seed-savers "Priya Nair"
1

$ redis-cli -p 6399 SISMEMBER club:seed-savers "Sam Whitfield"
0
```

Adding Ada a second time answers 0:
the set already had her, and nothing
changed. A set is where "have we seen
this before?" gets a cheap, exact
answer — `SCARD` counts, `SISMEMBER`
asks after one name.

Sets earn their place when the
question involves two of them:

```
$ redis-cli -p 6399 SINTER club:seed-savers club:tool-library
Ruth Adler
Tom Okafor

$ redis-cli -p 6399 SUNION club:seed-savers club:tool-library
Ada Osei
Priya Nair
Tom Okafor
Sam Whitfield
Ruth Adler

$ redis-cli -p 6399 SDIFF club:seed-savers club:tool-library
Ada Osei
Priya Nair
```

`SINTER` is "in both", `SUNION` "in
either" (five names, no duplicates),
`SDIFF` "in the first but not the
second". Notice `SUNION`'s order is
nothing like alphabetical — a set has
**no order**, and any meaning you read
into the listing order is yours, not
the server's. `SMEMBERS` prints a set
the same unpromising way; read sets
with membership questions, not by
listing them.

**Check:** both clubs together hold
exactly five distinct people, and the
intersection is exactly two.

## Sorted sets: the leaderboard

Every member of a sorted set carries
a number. The community garden logs
season volunteer hours:

```
$ redis-cli -p 6399 ZADD garden:season-hours 12.5 "Ada Osei" 9 "Tom Okafor" 21.25 "Ruth Adler" 4.5 "Priya Nair" 15 "Sam Whitfield"
5

$ redis-cli -p 6399 ZRANGE garden:season-hours 0 -1 WITHSCORES
Priya Nair
4.5
Tom Okafor
9
Ada Osei
12.5
Sam Whitfield
15
Ruth Adler
21.25
```

`ZRANGE` walks lowest score first;
`WITHSCORES` interleaves the numbers.
For a leaderboard you almost always
want the other direction:

```
$ redis-cli -p 6399 ZREVRANGE garden:season-hours 0 2 WITHSCORES
Ruth Adler
21.25
Sam Whitfield
15
Ada Osei
12.5

$ redis-cli -p 6399 ZREVRANK garden:season-hours "Ada Osei"
2

$ redis-cli -p 6399 ZSCORE garden:season-hours "Ruth Adler"
21.25
```

`ZREVRANGE 0 2` is the top three.
`ZREVRANK` answers a member's place,
counting from **0** — Ada's 2 means
third. And scores move, which is the
point of the structure:

```
$ redis-cli -p 6399 ZINCRBY garden:season-hours 2 "Priya Nair"
6.5
```

**Check:** Priya's score is 6.5 — and
the top three are unchanged, because
6.5 doesn't reach Ada's 12.5. A
leaderboard is a query (`ZREVRANGE`)
over a structure the server keeps
sorted on every write, which is why
"top 10, updated live" costs a
keystroke here and a scan in a
relational table.
