---
name: Map a merchant store into Barogo Gorela
description: >-
  Link your own store key to a Gorela store so store-based ordering works, then check the
  store's delivery zones, prepaid balance and the delivery agencies' current dispatch state
  before you let orders flow.
api: openapi/barogo-gorela-openapi.yml
operations:
  - listStores
  - createStoreMapping
  - getStoreMapping
  - pauseStoreMapping
  - getStoreAreas
  - getStoreDepositInfo
  - getDeliveryAgencyConditions
generated: '2026-08-06'
method: generated
source: https://developer.gorelas.com/api-docs-md/request-12.md
---

# Map a store

Store-based ordering requires a **1:1 mapping** between your `orderAgencyStoreId` and Gorela's
`storeId`. Without it, order intake fails with `400 NONE_ORDER_AGENCY_MAPPING` or
`400 NONE_DELIVERY_AGENCY_MAPPING`.

Address-based ordering skips all of this — you pass pickup and drop addresses on each order.
Use address-based when you do not manage a persistent merchant relationship.

## Step 1 — find the Gorela store (`listStores`)

`GET /api/stores`

Search for the merchant location you want to map. You need Gorela's `storeId`.

## Step 2 — create the mapping (`createStoreMapping`)

`POST /api/store-mapping`

Binds your `orderAgencyStoreId` to that `storeId`. One-to-one — a partner store key maps to
exactly one Gorela store.

## Step 3 — verify (`getStoreMapping`)

`GET /api/store-mapping/{orderAgencyStoreId}`

Returns the mappings held against your store key. Check this before your first order rather
than discovering the gap as a 400 at intake.

## Pausing (`pauseStoreMapping`)

`PUT /api/store-mapping/pause`

Suspends a mapping. A mapping whose state is paused counts as absent for order intake, which is
what `NONE_DELIVERY_AGENCY_MAPPING` means when every mapped agency is stopped.

## Before you switch a store on

- **`getStoreAreas`** — `GET /api/order-agency-stores/{orderAgencyStoreId}/areas`. Returns the
  store's deliverable zones, excluded zones and surcharge zones. Check a drop address against
  these, or use `checkDeliveryPossible` per order and let Gorela decide.
- **`getStoreDepositInfo`** — `GET /api/deposit-infos`. The store's prepaid balance (Cash,
  Money) per delivery agency, plus the virtual account for topping it up. A store that runs its
  balance to zero stops getting deliveries. Some stores have no virtual account issued, in which
  case account fields are absent — handle that, don't assume they are present.
- **`getDeliveryAgencyConditions`** — `GET /api/delivery-conditions`. Current dispatch state
  across every connected delivery agency.

## Handle 207 on these reads

`getStoreDepositInfo` and `getDeliveryAgencyConditions` fan out across delivery agencies and
can answer **HTTP 207** — partial success. The `data[]` holds what came back; `errors[]` names
each agency that failed, with the agency in `error.info`. Do not render a 207 as a complete
picture; show the missing agencies as unknown.

## Production mapping is not self-serve

Staging mapping you can do yourself with these operations. **Production store mapping at
service launch is arranged with your Barogo sales/operations owner** — 운영 환경 상점 매핑 준비
과정은 법인 영업/운영 담당자에게 문의. Plan for that lead time; it is not an API call.
