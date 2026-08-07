---
name: pagination-join-safety
description: Checklist for paginated list queries (LIMIT/OFFSET, page/items, TypeORM find/findAndCount, or any raw SQL) that join or load a one-to-many relation. Catches the class of bug where the join multiplies rows and silently corrupts page counts, page boundaries, or item order. Trigger when writing a new paginated list endpoint/repository/raw query that joins another table, when reviewing/investigating an existing paginated query (raw SQL or ORM), or when a list API returns fewer/more items than the page size, or duplicate/missing rows across pages.
---

# Pagination + one-to-many join safety

This is a **SQL/relational-algebra issue, not an ORM-specific one**. It applies
identically to raw SQL, TypeORM, Prisma, Knex, or anything else that generates
`LIMIT`/`OFFSET` on a joined query — the root cause is how JOIN and LIMIT
interact at the database level, so the fix is the same regardless of what
generated the query.

## The core pitfall

`LIMIT`/`OFFSET` (or ORM equivalents: `take`/`skip`, `page`/`items`) apply to
the **row count of the query as executed** — not to the count of distinct
parent entities you intend to paginate.

If a query joins a one-to-many child table (e.g. `transactions` LEFT JOIN
`transaction_files`, where one transaction can have 0, 1, or many files) and
`LIMIT` is applied to that joined result set, a parent with multiple matching
child rows consumes more than one "slot" in the limit — you can get **fewer
distinct parents than the page size**, and `OFFSET` on the next page can
**skip or duplicate** parents depending on how many child rows preceded them.

This bug is silent: it doesn't error, it just quietly returns wrong pages.
It gets worse under filters/searches that change which child rows match, so a
count that "happened to work" in one environment can break in another.

## Diagnostic questions to ask before shipping (or when reviewing) a paginated query

1. Does this query `LEFT JOIN`/`INNER JOIN` any table where the parent-to-child
   cardinality can be greater than 1?
2. Is `LIMIT`/`OFFSET` (or TypeORM `take`/`skip`) applied in the **same**
   query as that join?
3. If yes to both — the query is very likely wrong. Pick one of the safe
   patterns below.

If the join is guaranteed 1:1 in practice (e.g. a `LEFT JOIN` to a table with
a unique constraint on the join key), this isn't an issue — but say so
explicitly in a comment, because "guaranteed 1:1" is exactly the kind of
assumption that quietly stops being true later.

## Safe pattern 1 — dedupe the child to 1 row before joining (single query)

Use a subquery with `ROW_NUMBER() OVER (PARTITION BY <parent_key> ORDER BY
<tiebreaker>)`, filtered to `rn = 1`, and join *that* — guaranteeing at most
one child row per parent before the join, so `LIMIT`/`OFFSET` on the outer
query is safe. This works the same whether it's hand-written raw SQL or built
through a query builder.

Raw SQL:
```sql
SELECT t.*, latest_payment.paid_at
FROM transactions t
LEFT JOIN (
  SELECT p.*, ROW_NUMBER() OVER (
    PARTITION BY p.transaction_id ORDER BY p.paid_at DESC
  ) AS rn
  FROM payments p
) latest_payment
  ON t.id = latest_payment.transaction_id AND latest_payment.rn = 1
ORDER BY t.created_at DESC
LIMIT 25 OFFSET 0;
```

TypeORM `SelectQueryBuilder` (same idea — see `CancellationRequestRepository`'s
`_getLatestPaymentSubQuery`/`_getLatestCancellationHistorySubQuery` for the
full working pattern):
```ts
private _getLatestPaymentSubQuery() {
  return (subQuery: SelectQueryBuilder<PaymentEntity>) => {
    return subQuery
      .select([
        'payment_sub.id',
        'payment_sub.policy_id',
        'payment_sub.paid_at',
        `ROW_NUMBER() OVER (
           PARTITION BY payment_sub.policy_id
           ORDER BY payment_sub.paid_at DESC
         ) AS rn`,
      ])
      .from(PaymentEntity, 'payment_sub');
  };
}
// .leftJoin(this._getLatestPaymentSubQuery(), 'latest_payment',
//   'req.policy_id = latest_payment.policy_id AND latest_payment.rn = 1')
// .limit(limit).offset(offset)   -- safe: at most 1 row per parent already
```

Best when: you want one DB round-trip and the "pick one child" logic (latest,
highest priority, etc.) is naturally expressible as an `ORDER BY` inside the
window function.

## Safe pattern 2 — paginate the parent alone, then batch-fetch children by ID

Two queries. Query 1 paginates the parent with no join at all, so
`LIMIT`/`OFFSET` is trivially correct. Query 2 fetches all matching child rows
for exactly that page's parent IDs (`WHERE parent_id IN (...)`) — no `LIMIT`
needed there since it's already bounded by the ID list. Merge in application
code.

```ts
// Query 1 — plain, no join, pagination is guaranteed correct
const transactions = await this.find({
  where, order, take: limit, skip: offset,
});

// Query 2 — bounded by IN(...), not by LIMIT, so cardinality can't corrupt it
const ids = transactions.map(t => t.id);
const files = await this.eStampFileRepository.find({
  where: { eStampTransactionId: In(ids), type: EStampFileType.PayInSlip },
  order: { createdAt: 'DESC' },
});

// Merge in the service layer: group `files` by eStampTransactionId,
// take the first (already ordered DESC = latest) per transaction.
```

Best when: the join logic is simple (one type filter, "pick latest"), and
readability matters more than saving one round-trip — the extra query costs
nothing meaningful for a page of ≤100 rows.

## Unsafe pattern to avoid

```ts
// BAD: plain join + LIMIT in the same query, child is one-to-many
this.createQueryBuilder('t')
  .leftJoinAndSelect('t.files', 'files')  // one-to-many, unfiltered
  .take(25)
  .skip(0)
  .getMany();
// A transaction with 3 files can eat 3 "slots" of the LIMIT budget.
// TypeORM's leftJoinAndSelect + take/skip on a to-many relation has this
// exact footgun even when going through the entity-hydrating API — this
// isn't a raw-SQL-only problem, so always ask the diagnostic questions
// above regardless of which API generated the query.
```

```sql
-- BAD: same bug in hand-written SQL
SELECT t.*, f.media_id
FROM transactions t
LEFT JOIN files f ON f.transaction_id = t.id
LIMIT 25 OFFSET 0;
```

## Checklist

- [ ] Identify every join in the paginated query (raw SQL or ORM); confirm cardinality (1:1 vs 1:many) for each
- [ ] For any 1:many join, confirm it is either deduped via `ROW_NUMBER()`/`DISTINCT ON` *before* `LIMIT`, or split into a separate non-limited query
- [ ] Never combine `LIMIT`/`OFFSET` (or `.take()/.skip()`, `leftJoinAndSelect`/`relations`) with a to-many join in the same query
- [ ] `countList()`/count query uses the *same* base filters as `list()` but never needs the child join at all unless the join affects `WHERE` (e.g. filtering by a child column)
- [ ] If reviewing an existing query: reproduce with a parent row that has >1 matching child rows and confirm the page size/count is still correct