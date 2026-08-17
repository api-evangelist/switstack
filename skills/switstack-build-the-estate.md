---
name: Build a Switcloud merchant estate
description: >-
  Create the Merchant -> Store -> POI hierarchy that both the payment backend and the on-device payment application
  depend on. This is step 1 of the Switcloud getting-started flow and must complete before any EMV configuration or
  payment work.
api: openapi/switstack-switcloud-openapi.yml
operations:
  - create_merchant
  - create_store
  - create_poi
  - list_merchants
  - list_stores
  - list_pois
  - get_poi
  - partial_update_poi
generated: '2026-08-17'
method: generated
source: >-
  openapi/switstack-switcloud-openapi.yml, https://docs.switstack.io/switcloud/getting_started/,
  https://docs.switstack.io/switcloud/key_concepts/
---

# Build a Switcloud merchant estate

The estate is the hierarchy `Organization -> Merchant -> Store -> POI`. Switstack calls it the Business Object Model
(BOM); it replaces a traditional terminal management system, and it is what makes multi-tenant use cases (payment
facilitators, enterprise fleets) work.

## Before you start

- Authenticate first — see `switstack-authenticate`. You need an **Organization Admin** identity: the docs state an org
  admin "can manage users, merchants, stores, and POI configurations but cannot create payments".
- There is **no Organization resource** in the API. Organizations are provisioned by Switstack out of band, and
  `organization_id` is required on every read schema. Read it off any existing object, or off your first create.
- Do not run this without a plan for idempotency. No `Idempotency-Key` parameter exists on this API, so a retried
  create makes a duplicate. Before each create, list with `search` and reconcile.

## Steps

1. **Create the merchant.** `POST /api/bom/merchants` (`create_merchant`). `MerchantReadSchema` requires `name`, and
   returns `id` and `organization_id`.
2. **Create the store.** `POST /api/bom/stores` (`create_store`). Required: `name`, `merchant_id` (from step 1), and
   `address`. Omitting `merchant_id` is the most common 422 here.
3. **Register the POI.** `POST /api/bom/pois` (`create_poi`). Required: `name`, `identifier`, `serial_number`,
   `store_id`. Optional: `description`, `brand`, `state`.
   - `brand` is an enum: `UNDEFINED`, `SUNMI`, `CASTLES`, `PAX`, `SAMSUNG`, `GOOGLE`.
   - `state` is the device lifecycle: `UNDEFINED`, `PROVISIONED`, `OPERATED`, `SUSPENDED`, `PHASED_OUT`. Create as
     `PROVISIONED` and move to `OPERATED` when the device is live.
   - `identifier` and `serial_number` are both required and distinct — `serial_number` is the hardware serial;
     `identifier` is your POI identifier.
4. **Verify.** `GET /api/bom/pois` (`list_pois`) filtered by `store_id`, `brand` or `state`, or `GET
   /api/bom/pois/{id}` (`get_poi`). Add `with_related=true` to see related objects inline.
5. **Move a device through its lifecycle.** `PATCH /api/bom/pois/{id}` (`partial_update_poi`) with only the fields you
   are changing — e.g. `{"state": "SUSPENDED"}`. Prefer PATCH over `update_poi` (PUT), which replaces the object.

## Reading collections

Every list operation on this API pages the same way and takes the same helpers:

- `page` + `size`; the response is `{items, total, page, size, pages}`. There is no cursor and no `Link` header —
  compute the next request from `pages`.
- `organization_id` selects the tenant, `search` is free-text, `order_by` sorts.
- `with_related=true` expands nested objects instead of returning bare `*_id` fields.

## Deleting

`DELETE` is available on merchants, stores and POIs, and takes `cascade_delete`. Treat `cascade_delete=true` as
destructive: it removes children with the parent. Prefer setting a POI's `state` to `PHASED_OUT` over deleting it, so
past payments keep a resolvable `poi_id`.

## Error handling

Only **422** is declared, with `{"detail":[{"loc","msg","type"}]}`. In this flow it almost always means a missing
required foreign key (`merchant_id` on a store, `store_id` on a POI) or a malformed uuid. 404 on an unknown `{id}` and
403 on a role boundary are undeclared but real — handle both without retrying.

## Next

`switstack-configure-emv-for-a-terminal` — the EMV configuration a POI needs before it can transact.
