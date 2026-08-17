---
name: Configure EMV for a Switcloud terminal
description: >-
  Assemble the EMV Level 2 configuration a POI fetches at runtime — kernel parameter sets, CAPKs, revocation entries
  and BIN tables — and bundle them into the POIConfig that create_payment references. Step 2 of the Switcloud
  getting-started flow.
api: openapi/switstack-switcloud-openapi.yml
operations:
  - create_emv
  - create_emv_list
  - add_emv_to_emv_list
  - create_emv_config
  - create_capk
  - create_capk_list
  - add_capk_to_capk_list
  - create_cr
  - create_cr_list
  - add_cr_to_cr_list
  - create_bin
  - create_bin_list
  - add_bin_to_bin_list
  - create_bin_config
  - add_bin_list_to_bin_config
  - create_poi_config
  - get_poi_config
  - list_poi_configs
generated: '2026-08-17'
method: generated
source: >-
  openapi/switstack-switcloud-openapi.yml, https://docs.switstack.io/switcloud/getting_started/,
  https://docs.switstack.io/switcloud/key_concepts/, https://docs.switstack.io/switcloud/architecture/
---

# Configure EMV for a Switcloud terminal

Switcloud centralizes what is normally baked into terminal firmware: CAPKs, BIN tables, revocation lists and kernel
parameters are defined once in the cloud and **fetched by the device at runtime**, so configuration can change without
a firmware update. This skill builds that configuration.

## The shape of it

Everything follows the same three-level pattern — **item -> list -> config** — and the four families converge on one
`POIConfig`:

```
EMV      -> EMVList (nominal + failsafe) -> EMVConfig ┐
CAPK     -> CAPKList ─────────────────────────────────┤
CR       -> CRList ───────────────────────────────────┼-> POIConfig
BIN      -> BINList -> BINConfig ─────────────────────┘
```

The docs are explicit about what is mandatory: *"A POIConfig object must contain at least an EMVConfig (and its
sub-elements). CAPKs, CRs, and BINs are optional and can be configured later."*

## Before you start

Authenticate as an **Organization Admin** (`switstack-authenticate`) and have an estate in place
(`switstack-build-the-estate`). As everywhere on this API, there is no idempotency key — list-then-create rather than
blind-retrying.

## Steps

1. **Kernel parameter sets.** `POST /api/config/emvs` (`create_emv`) once per parameter set. Two enums matter:
   `technology_type` is `CONTACT` or `CONTACTLESS`; `transaction_type` is an EMV code from
   `00, 01, 02, 09, 17, 20, 21, 93`.
2. **Group them.** `POST /api/config/emv-lists` (`create_emv_list`), then attach each EMV with
   `POST /api/config/emv-lists/{id}/emv/{emv_id}` (`add_emv_to_emv_list`).
3. **Build the EMVConfig.** `POST /api/config/emv-configs` (`create_emv_config`) with `emv_nominal_list_id` and
   `emv_failsafe_list_id`. Both point at EMVLists. The failsafe list is the fallback parameter set — populate it; it is
   the difference between a degraded terminal and a dead one.
4. **CAPKs (offline data authentication).** `POST /api/config/capks` (`create_capk`) — `hash_type` is `UNDEFINED`,
   `SHA1` or `SHA256`; `algorithm_type` is `RSA`. Group with `create_capk_list` +
   `add_capk_to_capk_list`. Required for ODA (SDA/DDA/CDA/XDA), so in practice not optional for a production terminal.
5. **Revocation entries.** `POST /api/config/crs` (`create_cr`), then `create_cr_list` + `add_cr_to_cr_list`.
6. **BIN tables.** `POST /api/config/bins` (`create_bin`) -> `create_bin_list` + `add_bin_to_bin_list` ->
   `create_bin_config` + `add_bin_list_to_bin_config`. Note this family has one extra level than the others: BINLists
   attach to a BINConfig, and it is the **BINConfig** the POIConfig references.
7. **Bundle into a POIConfig.** `POST /api/config/poi-configs` (`create_poi_config`) with `emv_config_id` (required in
   practice) plus `bin_config_id`, `capk_list_id` and `cr_list_id` as available.
8. **Verify the whole bundle in one call.** `GET /api/config/poi-configs/{id}?with_related=true` (`get_poi_config`) —
   with expansion on, the response inlines `bin_config`, `capk_list`, `cr_list` and `emv_config` so you can confirm the
   graph resolved before a terminal ever fetches it. Do this every time; a POIConfig with a dangling `*_id` looks fine
   without expansion.

## Operating it

- Configuration is versioned by object, not by a version field: create a new POIConfig and repoint rather than mutating
  a live one, so an in-flight payment keeps the config it started with (`Payment.poi_config_id` records which one was
  used).
- `list_poi_configs` supports `page`/`size`/`search`/`order_by`/`organization_id` like every collection here.
- `DELETE` on any of these accepts `cascade_delete`. Do not cascade-delete a config family that a POIConfig still
  references.

## Error handling

Only **422** is declared. In this flow it is nearly always a dangling or malformed reference — `emv_config_id`,
`capk_list_id`, `cr_list_id` and `bin_config_id` are `format: uuid`, so a non-uuid string fails validation here rather
than at fetch time. The join endpoints (`add_*_to_*`) take two path ids; a wrong order is the second most common 422.

## Next

`switstack-create-a-payment` — the transaction flow that consumes `poi_id` + `poi_config_id`.
