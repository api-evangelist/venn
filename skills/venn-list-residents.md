---
name: venn-list-residents
description: >-
  Page through the residents (users and members) of a Venn building or community
  over the Venn tenant GraphQL API, honouring Venn's mandatory cursor pagination.
api: Venn Tenant GraphQL API
endpoint: https://api.venn.city/production/graphql
transport: graphql
operations:
  - Query.usersConnection
  - Query.membersConnection
  - Query.buildings
  - Query.hoods
generated: '2026-09-02'
method: generated
source: graphql/venn-tenant.graphql
---

# List residents in a Venn building or community

Every field, argument and type named here was verified against the live schema in
`graphql/venn-tenant.graphql` (introspected 2026-09-02). Do not invent fields —
the schema carries no descriptions, so guessing is unusually easy and unusually
wrong.

## Before you start

- **Auth is required for data.** Introspection is anonymous, reads are not. Send
  `Authorization: Bearer <cognito-access-token>`. Without it the gateway answers
  HTTP 200 with `errors[0].extensions.code = "VennUnknownError"` and `data: null`.
  See `authentication/venn-authentication.yml`.
- **Pagination is mandatory.** Omitting `first` or `last` on a list field returns
  `extensions.code = "pagination_enforce_error"` with the message
  *"Cant fetch all data - must send query with first or last arguments"*. There is
  no way to fetch a whole collection in one call.
- **Send a `venn-request-id` header** so your call is traceable on both sides. It
  comes back on the response and is recorded on the `Audit` entity.

## Steps

### 1. Find the community or building

`Query.hoods` and `Query.buildings` both take
`(first, last, after, before, skip, orderBy, where)`.

```graphql
query FindBuilding($name: String!) {
  buildings(first: 10, where: { name_contains: $name }) {
    id
    name
  }
}
```

### 2. Page the residents

Use the `*Connection` form, not the bare list — it returns `pageInfo` and
`aggregate` so you get a total on the same round trip.

```graphql
query Residents($buildingId: ID!, $after: String) {
  usersConnection(
    first: 50
    after: $after
    orderBy: createdAt_DESC
    where: { deleted: false, computedBuilding: { id: $buildingId } }
  ) {
    aggregate { count }
    pageInfo { endCursor hasNextPage }
    edges {
      node {
        id
        firstName
        lastName
        email
        residentialStatus
        computedUnit { id }
      }
    }
  }
}
```

### 3. Loop

Repeat with `after: pageInfo.endCursor` while `pageInfo.hasNextPage` is true.
Do not raise `first` to avoid paging; the gateway is the one enforcing the budget.

## Filtering

`UserWhereInput` supports per-field operators — `_not`, `_in`, `_not_in`, `_lt`,
`_lte`, `_gt`, `_gte`, `_contains`, `_not_contains`, `_starts_with`, `_ends_with` —
plus `AND` / `OR` / `NOT`. Sorting uses the `UserOrderByInput` enum
(`<field>_ASC` / `<field>_DESC`).

## Soft delete

Every entity carries `deleted` and `deletedAt` and both are filterable. Records are
**not** removed by `deleteUser` — filter `where: { deleted: false }` unless you
deliberately want tombstones.

## Errors

| `extensions.code` | What to do |
|---|---|
| `pagination_enforce_error` | Add `first` or `last`. Not retryable as-is. |
| `VennUnknownError` | Authenticate, or you lack tenant access. Do not hammer. |
| `GRAPHQL_VALIDATION_FAILED` | Your document does not match the schema. Fix it; never retry. |

Full envelope and codes: `errors/venn-error-codes.yml`.
No rate-limit headers are returned — see `rate-limits/venn-rate-limits.yml` — so
back off on your own schedule rather than waiting for a `Retry-After`.
