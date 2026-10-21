# Hashes

A **hash** is one key holding a small
record: a set of **field → value**
pairs, like a single row of a table
with the table stripped away. It's
the shape for "an object with a few
named facts" — and it costs far less
memory than one key per fact, because
the fields live inside the key instead
of each carrying a key's overhead.

The seed library's member record:

```
$ redis-cli -p 6399 HSET member:1001 name "Ada Osei" joined 2024-03-14 loans 2
3

$ redis-cli -p 6399 HGETALL member:1001
name
Ada Osei
joined
2024-03-14
loans
2

$ redis-cli -p 6399 HGET member:1001 name
Ada Osei

$ redis-cli -p 6399 TYPE member:1001
hash
```

`HSET` answers with the number of
*new* fields it created (3 — setting
existing fields again answers 0).
`HGETALL` dumps field and value in
alternating lines; `HGET` fetches one
field by name.

## Working on fields

Fields behave like small strings with
their own commands:

```
$ redis-cli -p 6399 HINCRBY member:1001 loans 1
3

$ redis-cli -p 6399 HSET member:1001 phone "555-0142"
1

$ redis-cli -p 6399 HGETALL member:1001
name
Ada Osei
joined
2024-03-14
loans
3
phone
555-0142

$ redis-cli -p 6399 HLEN member:1001
4

$ redis-cli -p 6399 HEXISTS member:1001 phone
1

$ redis-cli -p 6399 HDEL member:1001 phone
1

$ redis-cli -p 6399 HEXISTS member:1001 phone
0
```

`HINCRBY` is chapter 2's counter
living inside a record — the loans
count went 2 → 3 without touching the
rest of the card. `HSET` returns 1 for
the genuinely new phone field, `HLEN`
counts fields, `HEXISTS` and `HDEL`
ask after and remove single fields.

**Check:** loans ends at 3, phone is
added and removed, and `HGETALL` in
the middle shows all four fields in
one answer — the whole record costs
one round trip, not four.

## Hash or separate keys?

The decision that matters:

- **A hash** when the fields are
  always read and written *together*
  as one object's facts, and when no
  field needs its own expiry — a hash
  expires as a whole key or not at
  all (in this version; per-field
  expiry exists only in much newer
  releases than the lab's 7.0, so
  don't plan on it).
- **Separate keys** when the pieces
  live independent lives: different
  lifetimes, different owners, or the
  value is large.

Member cards, product details, a
user's settings — hashes. Session
tokens, stock counters, cache pages —
keys. The same system usually wants
both, which is fine: they're all just
keys, and `TYPE` will always tell you
which one you're holding.
