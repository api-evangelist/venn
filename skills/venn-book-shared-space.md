---
name: venn-book-shared-space
description: >-
  Check availability and book a time slot in a Venn shared space, then cancel it —
  the one flow in the Venn tenant GraphQL API that has a named, first-class
  reversal operation.
api: Venn Tenant GraphQL API
endpoint: https://api.venn.city/production/graphql
transport: graphql
operations:
  - Query.sharedSpaces
  - Query.getTimeSlotsForDates
  - Query.checkIfTimeSlotsAvailable
  - Mutation.bookTimeSlots
  - Query.myBookedTimeSlots
  - Mutation.cancelBooking
generated: '2026-09-02'
method: generated
source: graphql/venn-tenant.graphql
---

# Book (and cancel) a Venn shared space

Every operation, argument and type below was verified against
`graphql/venn-tenant.graphql` (introspected 2026-09-02).

## Reversibility — read this before you write

`Mutation.cancelBooking(calendarBookingId: String!)` exists and returns a
`CancelCalendarBookingResult`, so this action **can** be taken back. Venn publishes
**no window** for it — no documentation states how late a booking may be cancelled,
and none is assumed here. Treat the window as unknown and confirm with the operator
before booking on someone's behalf. See `conventions/venn-conventions.yml`
(`reversibility`).

There is **no idempotency key** on this flow. `bookTimeSlots` retried after a
timeout may create a second booking. If a call does not return, query
`myBookedTimeSlots` before retrying — do not blind-retry a write.

## Steps

### 1. Find the shared space

```graphql
query Spaces($hoodId: ID!) {
  sharedSpaces(first: 25, where: { deleted: false, hood: { id: $hoodId } }) {
    id
    name
  }
}
```

### 2. Read the real slots for a date range

`getTimeSlotsForDates(resource: CalendarBookableInput!, startTime: DateTime!, endTime: DateTime!)`
returns `[CalendarTimeSlot]`. Never construct slot boundaries yourself — take them
from this response.

### 3. Confirm availability immediately before writing

```graphql
query Check($resource: CalendarBookableInput, $slots: [CalendarTimeSlotInput!]!) {
  checkIfTimeSlotsAvailable(resource: $resource, slots: $slots) {
    __typename
  }
}
```

### 4. Book

```graphql
mutation Book($resource: CalendarBookableInput, $slots: [CalendarTimeSlotInput!]) {
  bookTimeSlots(resource: $resource, slots: $slots) {
    id
  }
}
```

`bookOnBehalf` exists for an operator booking for a resident. Use it only when you
genuinely hold that authority.

### 5. Verify, then cancel if needed

```graphql
query Mine($resource: CalendarBookableInput!, $startDate: DateTime, $endDate: DateTime) {
  myBookedTimeSlots(resource: $resource, startDate: $startDate, endDate: $endDate) {
    id
  }
}

mutation Cancel($id: String!) {
  cancelBooking(calendarBookingId: $id) {
    __typename
  }
}
```

`cancelBooking` takes the booking id as a `String!`, not an `ID!` — a detail that
will fail validation if you assume otherwise.

## The generic CRUD path

`createCalendarBooking(data: CalendarBookingCreateInput!)` and
`deleteCalendarBooking` also exist (they are the OpenCRUD-generated pair on the
entity). Prefer `bookTimeSlots` / `cancelBooking`: those are the intent-shaped
operations and they are the ones that run Venn's availability logic.

## Errors

Codes and the error envelope: `errors/venn-error-codes.yml`. All errors return
HTTP 200 with `data: null` and a code in `errors[].extensions.code` — never branch
on the HTTP status.
