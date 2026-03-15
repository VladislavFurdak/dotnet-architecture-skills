# API design patterns

Collection endpoints need filtering, sorting, field selection, and pagination. This file covers the practical options and trade-offs for each.

## Filtering

Filtering narrows a collection to the needed subset.

### Simple parametric

```
GET /orders?status=paid&customerId=123
```

Works well for flat, exact-match filters.

### Operators in parameter names

```
GET /orders?createdAt.gte=2025-01-01&createdAt.lt=2025-02-01
GET /products?price.gte=100&price.lte=500
```

Readable and extensible. Keep operator naming consistent across the API.

### Lists of values

```
GET /orders?status=paid,shipped
```

Or repeated parameters:

```
GET /orders?status=paid&status=shipped
```

Pick one format and document it.

### Filtering rules

- validate all filter parameters
- constrain allowed fields and operators
- ensure filters hit indexes (or at least don't cause full scans)
- do not build a filtering DSL so complex that only one person understands it

## Sorting

```
GET /orders?sort=-createdAt
GET /products?sort=price,name
```

Minus = descending. No prefix = ascending.

### Rules

- allow sorting only on a documented set of fields
- define a stable tie-breaker (usually `id`) to guarantee deterministic order
- do not allow arbitrary sorting on unindexed fields
- document the default sort when `sort` is omitted

## Field selection

Field selection lets the client request only the fields it needs.

```
GET /users?fields=id,name,email
GET /orders?fields=id,status,total
```

For related entities, use a separate `include` parameter:

```
GET /orders/42?include=items,payment
```

### Rules

- keep it flat (`fields=...`) — don't build nested grammar
- use `include=` for related sub-resources
- if field selection starts requiring conditions or modes, consider GraphQL instead

## Pagination

Pagination protects the system from oversized responses and gives the client a predictable way to traverse a collection.

There is no universal approach — choose based on data characteristics and navigation pattern.

### Offset pagination

```
GET /products?limit=20&offset=40
```

Skip first N, return next M.

| Pros | Cons |
|---|---|
| Simple to implement and explain | Large offsets are expensive (DB skips rows) |
| Maps to classic table UI | Unstable on changing data (duplicates/gaps) |

Best for: catalogs, reference data, admin panels — calm collections with moderate depth.

### Page-based pagination

```
GET /products?page=3&pageSize=20
```

Usually translates to offset internally. Inherits the same trade-offs.

Contract must define:
- page numbering: 0-based or 1-based
- max `pageSize`
- out-of-bounds behavior
- response metadata: `totalPages`, `totalItems`, `hasNextPage`

Best for: business UIs with numbered page controls.

### Cursor pagination

```
GET /orders?limit=20&cursor=eyJjcmVhdGVk...
```

Cursor is opaque to the client. Contains internal position info.

| Pros | Cons |
|---|---|
| Stable on live data | No random page jumps |
| Efficient at any depth | Less intuitive for humans |
| Natural for infinite scroll | More complex to implement |

Response includes `nextCursor` / `prevCursor`.

Best for: feeds, chats, order streams, infinite scroll — any live, growing collection.

### Keyset pagination

```
GET /orders?limit=20&afterCreatedAt=2026-03-01T00:00:00Z&afterId=42
```

Builds a DB condition like `WHERE (created_at, id) > (?, ?) ORDER BY created_at, id LIMIT 20`.

Two fields needed because timestamps alone are not unique — `id` acts as a tie-breaker.

| Pros | Cons |
|---|---|
| Index-friendly, fast at any depth | Requires stable, indexed sort order |
| No row skipping | Harder for clients to use directly |

Often wrapped inside cursor pagination: client sees an opaque cursor, server uses keyset conditions internally.

Best for: event journals, logs, transactions, large tables with natural temporal order.

### Time-based pagination

```
GET /events?limit=100&after=2026-03-01T00:00:00Z
GET /logs?before=2026-03-07T12:00:00Z&limit=200
```

Special case where the traversal axis is time itself.

Pitfalls:
- timestamps are not always unique — need a second key (`id`)
- clock skew in distributed systems can cause out-of-order records
- no random page jumps

Best for: logs, telemetry, audit trails, event feeds, time series.

### Pagination choice summary

| Approach | Random jump | Live data stability | Deep traversal cost | Implementation complexity |
|---|---|---|---|---|
| Offset | Yes | Low | High | Low |
| Page-based | Yes | Low | High | Low |
| Cursor | No | High | Low | Medium |
| Keyset | No | High | Low | Medium |
| Time-based | No | Medium | Low | Medium |

### Pagination response contract

Regardless of the approach, the response should include enough metadata for the client to navigate:

```json
{
  "data": [...],
  "pagination": {
    "totalItems": 1042,
    "hasNextPage": true,
    "nextCursor": "..."
  }
}
```

What to include depends on the approach:
- offset/page: `totalItems`, `totalPages`, `hasNextPage`, `hasPreviousPage`
- cursor: `nextCursor`, `prevCursor`, `hasNextPage`
- keyset/time: `nextCursor` or explicit key values, `hasMore`
