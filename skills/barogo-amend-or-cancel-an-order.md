---
name: Amend or cancel a Barogo Gorela order in flight
description: >-
  Change a live order's pickup time, drop address, payment and products, memos or contacts —
  each within its own delivery-state window — raise the fare on an unassigned flexible order,
  signal that goods are ready, and cancel with the fee and refusal rules the docs set out.
api: openapi/barogo-gorela-openapi.yml
operations:
  - updateOrderPickupWishAt
  - updateOrderDropInfo
  - updateOrderPaymentInfo
  - updateOrderMemoInfo
  - updateOrderPhoneInfo
  - updateFlexibleOrderDeliveryPrice
  - markOrderPrepareComplete
  - cancelOrder
  - getOrder
generated: '2026-08-06'
method: generated
source: https://developer.gorelas.com/api-docs-md/request-8.md
---

# Amend or cancel a live order

Every amendment on this API has a **state window** — a delivery status past which it is
refused. Check the current delivery status before you offer the action in your UI, or you will
generate support traffic for buttons that cannot work.

## The state windows

| Operation | Path | Allowed until |
|---|---|---|
| `updateOrderPickupWishAt` | `PUT /api/orders/{orderAgencyOrderId}/pickup-wish-at` | delivery becomes `ALLOCATED` (배차) |
| `updateOrderDropInfo` | `PUT /api/orders/{orderAgencyOrderId}/drop-info` | delivery becomes `PICKUP_FINISHED` |
| `updateOrderPaymentInfo` | `PUT /api/orders/{orderAgencyOrderId}/payment-info` | delivery reaches a terminal state |
| `updateOrderMemoInfo` | `PUT /api/orders/{orderAgencyOrderId}/memo-info` | — |
| `updateOrderPhoneInfo` | `PUT /api/orders/{orderAgencyOrderId}/phone-info` | — |
| `updateFlexibleOrderDeliveryPrice` | `PUT /api/flexible/orders/{orderAgencyOrderId}/delivery-price` | before assignment, flexible-fare orders only |
| `markOrderPrepareComplete` | `POST /api/orders/{orderAgencyOrderId}/prepare-complete` | before `PICKUP_FINISHED`, address-based orders, once only |

## Amendments can change the fare

`updateOrderDropInfo` and `updateOrderPaymentInfo` can both move the delivery price. The docs
are explicit: **read the response and apply the changed fare** — 수정 요청에 성공한 경우 배달 요금이
변경되었을 수 있습니다. If you ignore the response you will bill the merchant the pre-amendment
figure. You may also receive an `onDeliveryChargeChanged` callback afterwards.

## Rescuing an unassigned flexible order

Flexible-fare orders (유연 요금 | 자율 수행) are opt-in for riders. If the fare is unattractive
nobody takes it and the order **auto-cancels** after a period. `updateFlexibleOrderDeliveryPrice`
raises the fare, but only **before assignment**. If your dashboard shows an order sitting
unassigned, that is the lever — and it has an expiry.

## Telling the rider goods are ready

`markOrderPrepareComplete` signals to the rider that the goods are ready for pickup. Address-based
orders only. It is single-use: a second call is rejected with
`ALREADY_COMPLETED_ORDER_PRODUCT_PREPARE`.

Note this operation answers **HTTP 200 with `data.isSuccess: false`** on rejection, alongside a
`reason` of `ALREADY_COMPLETED_ORDER_PRODUCT_PREPARE`, `INVALID_STATUS` or `ETC`. Branch on
`isSuccess`.

## Cancelling (`cancelOrder`)

`PUT /api/orders/{orderAgencyOrderId}/status/cancel`

Two things the docs insist on:

1. **Cancellation can be refused.** Operational policy decides, not your request. Handle a
   rejection as a normal outcome, not an error.
2. **A cancellation fee may apply**, scaling with how far the delivery has progressed
   (`cancelFee` on the delivery). Show that to the merchant before they confirm.

An order that failed intake cannot be cancelled — there is nothing to cancel.

Because one order can carry N deliveries, an order-level cancellation is not final until every
child delivery is terminal. Wait for `onOrderStatusChanged`, not for the cancel response, before
you tell the merchant it is done.

## After any amendment

Amendments are not idempotent and carry no idempotency key. On a timeout, wait at least 10s,
then call `getOrder` and compare the field you tried to change before retrying.
