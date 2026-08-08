---
name: Receive and reconcile Barogo Gorela delivery callbacks
description: >-
  Implement the Gorela → order-agency webhook receiver: which callbacks are mandatory, the
  3-second / 3-retry delivery contract, the order-to-N-deliveries state machine, and why
  payment and cash-receipt callbacks are cumulative rather than incremental.
api: openapi/barogo-gorela-callbacks-openapi.yml
operations:
  - onOrderStatusChanged
  - onDeliveryStatusChanged
  - onDeliveryPickupExpectedAtChanged
  - onDeliveryChargeChanged
  - onDeliveryCardPayments
  - onDeliveryCashReceipts
  - getOrder
generated: '2026-08-06'
method: generated
source: https://developer.gorelas.com/api-docs-md/index.md
---

# Receive Gorela callbacks

Callbacks, not polling, are how you learn what happened to an order. You host a base URL;
Gorela appends the documented path and POSTs JSON.

## The delivery contract

- You have **3 seconds** to respond. Slower is a timeout.
- On timeout Gorela retries **3 times, at 2s / 18s / 50s**. After that the event is gone.
- Therefore: **acknowledge first, process asynchronously.** Write the payload to a queue and
  return 200 immediately. Anything that touches your database synchronously will eventually
  breach 3 seconds under load and you will silently lose events.
- Gorela sends **every** callback it generates regardless of what you have implemented.
  Answer **404** on the paths you have not built — that is the documented signal, not an error.
- Ordering is **not** guaranteed between callbacks. Do not infer sequence from arrival order;
  use the `changedAt` / `createdAt` timestamps on the payload.
- Callbacks are **unauthenticated**. No signature, no shared secret, no mTLS is published. Treat
  the body as a hint, and confirm anything financial with `getOrder`. If you need transport
  controls, request IP allow-listing from tech_poc@barogo.com.

## Implement these first

| Callback | Path | Why |
|---|---|---|
| `onOrderStatusChanged` | `POST /order/status` | Required. Terminal order state — completed or cancelled. |
| `onDeliveryPickupExpectedAtChanged` | `POST /delivery/pickup-expected-at` | Required. The revised pickup time; the merchant must see it. |
| `onDeliveryStatusChanged` | `POST /delivery/status` | Assignment, reassignment, arrival at pickup, pickup complete. |
| `onDeliveryChargeChanged` | `POST /delivery/delivery-charge-info` | The fare you quoted at intake has changed. Reflect it. |

Then, as your product needs them: `onDeliveryAccepted`, `onDeliveryDropNear`,
`onDeliveryDropExpectedAtChanged`, `onDeliveryDropInfoChanged`, `onOrderPaymentInfoChanged`,
`onDeliveryCardPayments`, `onDeliveryCashReceipts`, `onStoreDeliveryDisabled`,
`onStoreDeliveryEnabled`, `onStoreDepositCharged`, and the store / order-agency area
create-update-delete events.

## The state machine that actually matters

**One order fans out to N deliveries.** Order state and delivery state are tracked separately.
An order reaches a terminal state only when **every** child delivery is terminal — rejected,
completed or cancelled.

Model it that way. If you collapse order and delivery into one record you will report a
completed order while a re-dispatched delivery is still in flight, which is the most common
integration defect on this API.

Correlate on `orderAgencyOrderId` (your key) plus `orderId` (Gorela's), and keep deliveries in
a child table keyed by `orderId` + `deliveryAgencyId`.

## Payment and cash-receipt callbacks are cumulative

`onDeliveryCardPayments` and `onDeliveryCashReceipts` always carry the **entire** history for
that delivery, including cancellations — never a single delta.

Two payments produce: `[A]`, then `[A, B]`. A cancellation of the first produces:
`[A]`, then `[A, B]`, then `[A, cancel-A, B]`.

So **replace** your stored list on each callback; do not append. Appending double-counts revenue.

These also arrive with no ordering guarantee relative to the completion callback — the rider
takes payment around drop-off, not strictly after it. Do not gate payment recording on having
already seen completion.

## Batching

`onStoreDeliveryDisabled` / `onStoreDeliveryEnabled` chunk their store lists **20 stores per
notification**. A suspension affecting 100 stores arrives as 5 callbacks. Do not treat one
callback as the full set.

## Reconciliation

Because callbacks are unauthenticated, unordered and droppable after 3 failed retries, run a
low-frequency reconciliation pass with `getOrder` for orders that have been open longer than
expected. Do not turn that into polling — the reference explicitly restricts partners who poll
the order list aggressively.
