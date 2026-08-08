---
name: Quote and place a Barogo Gorela delivery order
description: >-
  Check serviceability and fare, then accept an order into the Gorela delivery-brokerage
  platform, using the fixed-fare (고정 요금 | 의무 수행) intake. Covers both the address-based and
  store-based binding modes and the pickup-time contract the merchant must be shown.
api: openapi/barogo-gorela-openapi.yml
operations:
  - checkDeliveryPossible
  - createOrder
  - getOrder
generated: '2026-08-06'
method: generated
source: https://developer.gorelas.com/api-docs-md/request-1.md
---

# Quote and place a Gorela order

Gorela brokers between your platform (the *order agency*) and the delivery agencies that
dispatch riders. You never talk to a rider. You submit an order; Gorela creates one or more
deliveries against it.

## Before you start

- Base URL: `https://staging-api-interlocker.gorelas.com` while building,
  `https://api-interlocker.gorelas.com` in production. Test and live are separated by host only.
- Every request: `Authorization: Bearer {API_Key}`.
- If you are using **store-based** binding, the store must already be mapped — see the
  `barogo-map-a-store.md` skill. Address-based binding needs no mapping.

## Step 1 — quote before you accept (`checkDeliveryPossible`)

`POST /api/delivery-possible`

The reference makes this mandatory: **주문 접수 전 필히 해당 API를 호출**. It tells you whether the
address is serviceable and what the delivery will cost. Calling it first is what stops you
accepting an order you cannot fulfil, and it lets you show the customer a fare before checkout.

For the flexible-fare model, use `checkFlexibleDeliveryPossible`
(`POST /api/flexible/delivery-possible`) instead — there you propose the fare and riders opt in.

## Step 2 — accept the order (`createOrder`)

`POST /api/orders`

Choose the request-body variant that matches your integration (the spec models these as a
`oneOf`, each branch titled with its source document):

- 고정 요금 | 의무 수행 (주소 기반) — you send pickup and drop addresses inline.
- 고정 요금 | 의무 수행 (상점 기반) — you send `orderAgencyStoreId`; the store must be mapped.
- ACCEPTED_ORDER — the merchant already accepted the order in your system and the Barogo store
  program will request the delivery.

Rules that will bite you if you skip them:

- `orderAgencyOrderId` is **your** key and must be unique. A repeat returns
  `409 CONFLICT / DUPLICATED_ID`. There is no idempotency key — see the failure section below.
- `pickupWishAt` is what the merchant *wants*. The response's **`pickupExpectedAt` is what
  Gorela actually expects**, and it moves. The docs require you to surface `pickupExpectedAt`
  to the merchant; hiding it is called out as a direct cause of operational disputes.
- Prices: `totalPayPrice` is gross product value (before discounts) and can differ from
  `actualPayPrice`, what the customer really paid. Send both honestly.
- Products: a base product with different options is a **different** product. Compute
  `product.totalPrice = (product.unitPrice + (option.unitPrice * option.quantity)) * product.quantity`.
- A customer-paid delivery tip is a product with type `DELIVERY_TIP`, not a separate field.
- Contactless delivery requires `isUntact: true`. A memo string does **not** guarantee it.
- Pass the customer's instructions in `ordererOrderMemo` — door codes, gate routing. Omitting
  them is a documented cause of failed drops and returned goods.
- Reservation orders (`isReservation`) surface to riders a configurable number of minutes
  before the slot; the default is 60 and changing it needs agreement with Barogo.

## Step 3 — confirm state (`getOrder`)

`GET /api/orders/{orderAgencyOrderId}`

Use this to reconcile, not to track. **Do not poll for progress** — the reference warns that
indiscriminate polling can get your requests restricted. Progress arrives on callbacks; see
`barogo-receive-delivery-callbacks.md`.

## Reading the response

Success is `{"statusCode": 200, "data": {...}}`. Two traps:

1. **Business rejections come back as HTTP 200** with `data.isSuccess: false` and a `reason`
   enum. Branch on `isSuccess`, not on the status code.
2. **HTTP 207** means partial success across delivery agencies — read the `errors[]` array and
   treat the un-listed agencies as unknown, not as failed.

Errors use `{"statusCode": <int>, "error": {"category", "errorCode", "message"}}`. Branch on
`errorCode`. The full registry is in `errors/barogo-problem-types.yml`.

## Failure handling — the one thing to get right

Gorela publishes **no idempotency contract**. If `createOrder` times out you do not know
whether the order landed, and retrying risks either a duplicate order or a
`409 DUPLICATED_ID` that tells you nothing about the first attempt's outcome.

The safe sequence is:

1. Set your client timeout to at least **12000 ms** (the documented recommendation).
2. On timeout or 5xx, **wait at least 10000 ms** — immediate retry is explicitly discouraged.
3. Then call `getOrder` with your `orderAgencyOrderId`. If it exists, the order landed: stop.
4. Only if it does not exist, retry `createOrder`.

Never treat `409 DUPLICATED_ID` as a failure to create — it usually means your first attempt
succeeded.
