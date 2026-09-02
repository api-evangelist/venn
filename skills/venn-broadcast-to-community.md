---
name: venn-broadcast-to-community
description: >-
  Send a broadcast message to a Venn community, using the one part of the Venn
  tenant GraphQL API that supports an idempotency key — and understanding exactly
  where that protection stops.
api: Venn Tenant GraphQL API
endpoint: https://api.venn.city/production/graphql
transport: graphql
operations:
  - Query.hoods
  - Query.getBroadcastAudienceUserCount
  - Mutation.createBroadcast
  - Mutation.createNotification
  - Query.broadcasts
  - Mutation.deleteBroadcast
generated: '2026-09-02'
method: generated
source: graphql/venn-tenant.graphql
---

# Broadcast to a Venn community

Every operation, field and input name below was verified against
`graphql/venn-tenant.graphql` (introspected 2026-09-02).

## This is a high-consequence write

A broadcast reaches real residents. It is not reversible in the sense that matters:
`deleteBroadcast` exists and is a soft delete (`deleted` / `deletedAt`), but it does
**not** unsend messages already delivered. Confirm the audience size before sending.

## Steps

### 1. Size the audience first

`Query.getBroadcastAudienceUserCount` exists precisely so a caller can check reach
before committing. Call it. If the number surprises you, stop and ask.

### 2. Draft rather than send

`BroadcastCreateInput` carries `isDraft` and `sendStatus`. Create with
`isDraft: true`, have a human read it, then flip it. `scheduledBroadcast` is
available for a timed send.

```graphql
mutation Draft($data: BroadcastCreateInput!) {
  createBroadcast(data: $data) {
    id
    isDraft
    sendStatus
    hoodName
    stats { __typename }
  }
}
```

`BroadcastCreateInput` fields (exact, from the schema): `audienceRaw`, `audiences`,
`broadcastAttachments`, `category`, `content`, `contentRaw`, `createdByUserId`,
`dateSent`, `deleted`, `deletedAt`, `hoodIds`, `hoodName`, `isDraft`, `medium`,
`notifications`, `portfolioId`, `scheduledBroadcast`, `sendStatus`, `sourceType`.

### 3. Where idempotency actually applies

`Notification` — and only `Notification` — carries an `idempotencyKey: String`.
It is present on `NotificationCreateInput`, `NotificationUpdateInput` and
`SendNotificationToUserParams`.

So:

- A per-user notification send **can** be safely de-duplicated. Set
  `idempotencyKey` to a value you can regenerate deterministically, and a retry
  after a timeout will not double-deliver.
- `createBroadcast` has **no** idempotency key. A retried broadcast is a second
  broadcast. If a `createBroadcast` call does not return, query
  `Query.broadcasts(first: 5, orderBy: createdAt_DESC, where: {hoodName: ...})`
  and look before you retry.

This asymmetry is the single most important thing to know about writing to this
API. It is documented in `conventions/venn-conventions.yml` under `idempotency`,
and it is why this profile does **not** claim API-wide idempotency support.

### 4. Confirm

```graphql
query Recent($hoodName: String!) {
  broadcasts(first: 5, orderBy: createdAt_DESC, where: { hoodName: $hoodName, deleted: false }) {
    id
    sendStatus
    dateSent
    isDraft
  }
}
```

## Errors

All errors return HTTP 200; read `errors[].extensions.code`. `VennUnknownError`
from the `persistency` subgraph is what an unauthenticated or unauthorized call
gets — it does not distinguish "not allowed" from "server fault", so do not treat it
as automatically retryable. See `errors/venn-error-codes.yml`.
