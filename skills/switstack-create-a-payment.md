---
name: Create and reconcile a Switcloud payment
description: >-
  Run the Switcloud card-present transaction flow from the backend side — create a Payment, hand its ID to the on-device
  application, then retrieve the completed transaction data and its log data set for reconciliation, reporting and
  submission to the acquirer.
api: openapi/switstack-switcloud-openapi.yml
operations:
  - create_payment
  - get_payment
  - list_payments
  - partial_update_payment
  - get_log_data_set
  - list_log_data_sets
generated: '2026-08-17'
method: generated
source: >-
  openapi/switstack-switcloud-openapi.yml, https://docs.switstack.io/switcloud/processing_payments/,
  https://docs.switstack.io/switcloud/getting_started/
---

# Create and reconcile a Switcloud payment

Switcloud splits a card-present transaction in two: your **backend** orchestrates via this API, and the **payment
application on the device** performs the card interaction through the Switcloud Client SDK. This skill covers only the
backend half. The card leg is not callable over HTTP and must not be simulated.

Switstack's documented sequence:

```
Backend  -> Switcloud : Create Payment
Switcloud-> Backend   : Payment ID
Backend  -> Device app: Initiate payment (with Payment ID)
Device   -> Switcloud : Payment data (encrypted)
Device   -> Backend   : Payment complete
Backend  -> Switcloud : Get Payment data
```

## Before you start

- Authenticate (`switstack-authenticate`) as a **basic or machine user**. This is a role trap: an Organization Admin
  *cannot* create payments, and a basic user is read-only on the estate. You need a different identity here than the one
  that built the estate and configuration.
- You need a `poi_id` (`switstack-build-the-estate`) and a `poi_config_id`
  (`switstack-configure-emv-for-a-terminal`). Both are **required** on create.

## Steps

1. **Create the payment.** `POST /api/payment/payments` (`create_payment`) with at least `poi_id` and `poi_config_id`.
   The response `PaymentReadSchema` carries `id` — the Payment ID that coordinates state across backend, device app and
   Switcloud Client — plus `state`, `outcome_status` and `log_data_set_id`.
2. **Hand the Payment ID to the device application.** Out of band, over your own channel. The device app passes it to
   the Switcloud Client, which fetches the POIConfig, drives the EMV L2 stack locally, and uploads the encrypted payment
   data back to Switcloud.
3. **Wait for the device to report completion.** There is **no webhook and no callback** on this API — Switcloud
   publishes no event surface at all. Your device application notifies your backend, or you poll `get_payment`. If you
   poll, back off: no rate limits are published, so treat the service as unmetered-but-fragile rather than free.
4. **Retrieve the transaction.** `GET /api/payment/payments/{id}` (`get_payment`). Check **two** fields, not one:
   - `state`: `UNDEFINED` -> `CREATED` -> `READY` -> `INITIATED` -> `REQUESTED` -> `COMPLETED`. Anything short of
     `COMPLETED` means the device leg did not finish.
   - `outcome_status`: `APPROVED`, `DECLINED`, `ONLINE_REQUEST`, `END_APPLICATION`, `SELECT_NEXT`,
     `TRY_ANOTHER_INTERFACE`, `TRY_AGAIN`, `UNDEFINED`, `NA`. These are **EMV terminal outcomes**, not issuer decline
     codes — `TRY_ANOTHER_INTERFACE` means ask for contact instead of contactless; `TRY_AGAIN` means re-present the
     card. A declined transaction arrives as HTTP **200**.
   - EMV data arrives as TLV blobs in `trd`, `authorization` and `completion`; `pin_block` is nullable. Pass these to
     your acquirer or gateway; do not attempt to interpret them without an EMV parser (Swittest's `parse_tlv` will name
     the tags — see `switstack-run-emv-tests`).
5. **Pull the log data set for anything that went wrong.** `GET /api/payment/logs/{log_data_set_id}`
   (`get_log_data_set`) returns `meta_data`, `telemetry`, `config`, `trd`, `all_tags`, `apdus`, `trace` and `signals`.
   This is the only transaction-level diagnostic surface Switcloud offers and it is where an EMV failure is actually
   explained.
6. **Reconcile in bulk.** `GET /api/payment/payments` (`list_payments`) filters on `poi_id` and `outcome_status`, and
   pages with `page`/`size` returning `{items,total,page,size,pages}`.

## Idempotency — read this before you add retries

**There is no idempotency key on this API.** No `Idempotency-Key` parameter is declared on `create_payment` or anywhere
else, and no retry-safety policy is documented. A timed-out or 5xx `create_payment` that you retry will create a
**second Payment** against the same terminal.

Mitigate on your side:
- Generate your own correlation id, store it against the Payment ID you get back, and never re-issue a create for a
  correlation id you have already recorded — even if the response was lost.
- On a lost response, do **not** retry. Call `list_payments` filtered by `poi_id` and reconcile by recency before
  deciding anything.
- There is no metadata field on Payment to stash your correlation id in, so it has to live in your own store.

## Immutability

Payments have no `DELETE` — they are append-only. `partial_update_payment` (PATCH) exists but Super Admins see Payment
data as read-only; treat updates as a narrow correction path, not part of the happy flow. LogDataSets have no create and
no delete: they are created implicitly with the Payment.

## Error handling

Only **422** is declared, with `{"detail":[{"loc","msg","type"}]}` — usually a missing `poi_id`/`poi_config_id` or a
malformed uuid. Undeclared but real: **401** (expired token — refresh once), **403** (you are using an admin identity
that cannot create payments), **404** (unknown Payment ID). None of these should be blind-retried. And remember that the
transaction failing is **not** an HTTP error; it is `outcome_status` in a 200 body.
