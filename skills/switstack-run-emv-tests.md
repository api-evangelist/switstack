---
name: Run EMV tests with Swittest
description: >-
  Discover test suites, validate a custom suite, run a test selection while consuming the Server-Sent Events stream, and
  parse the resulting EMV TLV and Eval+ logs — the Swittest managed test-automation flow.
api: openapi/switstack-swittest-openapi.yml
operations:
  - list_test_suites
  - get_test_suite
  - get_test
  - get_test_config
  - get_test_vcard
  - verify_test_suite
  - verify_test
  - verify_bin_scope
  - verify_capk_scope
  - verify_cr_scope
  - verify_emv_scope
  - run_tests
  - run_custom_test
  - list_config_files
  - load_config_file
  - parse_log
  - parse_tlv
  - parse_tag
  - list_tags
generated: '2026-08-17'
method: generated
source: >-
  openapi/switstack-swittest-openapi.yml, https://docs.switstack.io/swittest/setup/,
  https://docs.switstack.io/swittest/cli/, https://docs.switstack.io/swittest/architecture/
---

# Run EMV tests with Swittest

Swittest is Switstack's managed EMV functional test-automation service. It drives a real device under test: the
**Swittest L3** Android app runs on the device, connects out to a Swittest server instance, and this API orchestrates the
runs. Swittest relies on Switcloud for card-processing data (configuration and payments).

## Before you start

- Authenticate (`switstack-authenticate`) with a **Full** role — the `Data` role can only read suites, tests, configs
  and vcards, not run anything.
- Your Swittest host is **instance-specific**. Switstack hosts separate Swittest instances per customer/partner and the
  hostname arrives with your access details; there is no shared public base URL. Do not guess one.
- The device under test must have the Swittest L3 app installed (sideload or ADB) and connected — check the connection
  status at the bottom of the app screen, and the host/port on its Settings screen.

## Steps

1. **Discover.** `GET /api/suites` (`list_test_suites`), then `GET /api/suites/{test_suite_name_or_index}`
   (`get_test_suite`). Suites and tests are addressable by **name or index** interchangeably.
2. **Inspect a test before running it.** `GET /api/tests/{suite}/{test}` (`get_test`), plus
   `get_test_config` for its configuration and `get_test_vcard` for its virtual card. Reading the config first is what
   tells you whether the test's pre-conditions match your device.
3. **Browse the config files.** `GET /api/configs/list` (`list_config_files`) and `GET /api/configs/load`
   (`load_config_file`, takes a `path` query parameter) to read Switstack's config documents.
4. **Validate before you run.** This API gives you a full pre-flight and you should use all of it:
   - `POST /api/verify/test-suite` (`verify_test_suite`) and `POST /api/verify/test` (`verify_test`) — check a custom
     suite/test's content and format.
   - `POST /api/verify/bin` (`verify_bin_scope`), `/api/verify/capk` (`verify_capk_scope`), `/api/verify/cr`
     (`verify_cr_scope`), `/api/verify/emv` (`verify_emv_scope`) — check each scope document.
   A malformed suite that fails mid-run wastes a bench session; verification is cheap.
5. **Run.** `POST /api/tests/{test_suite_name_or_index}/{test_selection}` (`run_tests`) with a `RunTestSchema` body.
   `test_selection` accepts:
   - a single test name
   - an index: `1036`
   - a list of indexes: `0, 99, 1036`
   - a range: `0-5`
   - a list of ranges: `0-5, 15-40`
   - a mix: `0-5, 8, 9, 15-40`
   - `all`
6. **Consume the stream.** The 200 response is `text/event-stream` — Server-Sent Events with `data`, `event`, `id` and
   `retry` fields, streaming execution status in real time. Choose a `verbose` level deliberately, because level 3 is
   very large:
   - **0** — status and errors only
   - **1** — level 0 plus payment data and log data sets
   - **2** — level 1 plus parsed authorization TLV in payment data
   - **3** — level 2 plus parsed `DF8129`, `DF8115` and `DF8116` tags
   Read pass/fail from `TestStatusEnum` in the streamed `TestResultSchema`; per-test failure detail arrives in
   `ErrorIndicationSchema`. Do not treat the HTTP 200 as a pass.
7. **Ad-hoc test.** `POST /api/tests` (`run_custom_test`) with a `RunCustomTestSchema` for a test that is not in a suite.
8. **Post-mortem.** `POST /api/parse/log` (`parse_log`) parses a supported Eval+ log file; `POST /api/parse/tlv`
   (`parse_tlv`) parses a TLV string; `POST /api/parse/tag` (`parse_tag`) and `POST /api/parse/list-tags` (`list_tags`)
   name and enumerate supported tags. These are also the tools for making sense of a Switcloud
   `Payment.trd`/`authorization`/`completion` blob (see `switstack-create-a-payment`).

## Automatic vs manual — a documented caveat

Switstack warns that the "Automatic" page is *"primarily made for developers doing regression testing. It is not meant to
be used by labs during debug or certification sessions."* For a certification session a lab tester is expected to create
the test manually, configure its pre-conditions, run it and verify each post-condition by hand. If you are automating
against `run_tests` with `all`, you are in the regression lane, not the certification lane — double-check
pre-conditions before trusting a green result.

## Error handling

Only **422** is declared, with `{"detail":[{"loc","msg","type","input","ctx"}]}` (Swittest's envelope carries `input`
and `ctx` in addition to the Switcloud fields). The usual causes are a malformed `test_selection` expression, an unknown
suite name/index, or a bad `path` on `load_config_file`. 401/403/404 are undeclared but real. Test *failures* are not
HTTP errors — they are in-band.

## CLI equivalent

Everything above is also reachable from `swittest`, the first-party Typer CLI (`settings`, `log`, `tlv`, `tag`,
`verify`, `configs`, `suites`, `tests`, `probe`) — see `cli/switstack-cli.yml`. It installs from the Switstack Nexus,
not PyPI, and needs access details from support.
