---
name: Authenticate against a Switstack API
description: >-
  Obtain, use, refresh and revoke an OAuth 2.0 bearer token for the Switcloud API or the Swittest API. Do this first —
  every operation on both APIs except the three /auth/* endpoints requires a bearer token.
api: openapi/switstack-switcloud-openapi.yml
operations:
  - token
  - refresh_token
  - revoke_token
generated: '2026-08-17'
method: generated
source: >-
  openapi/switstack-switcloud-openapi.yml, openapi/switstack-swittest-openapi.yml,
  https://docs.switstack.io/switcloud/security_authentication/
---

# Authenticate against a Switstack API

Both Switstack APIs use the same `OAuth2PasswordBearer` scheme, so this skill applies to either one. Only the host
differs.

## Before you start

You cannot self-serve credentials. You need an **active organization** and an **active user** provisioned by Switstack;
Switcloud additionally distinguishes an admin user from a basic or machine user. Requests go to `contact@switstack.io`.
Do not attempt to register — there is no signup endpoint.

Hosts are per-tenant. The Switcloud docs name `https://switcloud.switstack.io`; Swittest runs as separate instances and
its host arrives with your access details. Use the host you were given, not a guessed one.

## Steps

1. **Get a token.** `POST /auth/token` (`token`) with an `OAuth2Form` body.
   - Password grant: `grant_type=password`, `username` (an email address), `password`.
   - Client-credentials grant (machine users, Switcloud): `grant_type=client_credentials`, `client_id`,
     `client_secret`. The `Oauth2GrantType` enum accepts exactly `password` and `client_credentials`.
   - The response is a `TokenSchema`: `access_token`, `token_type` (`bearer`), `expires_in` (3600 on Switcloud),
     and optionally `refresh_token` + `refresh_expires_in`.
2. **Send the token.** Every subsequent call carries `Authorization: Bearer <access_token>`. The security requirement
   is declared per operation, so treat *all* non-`/auth/*` operations as authenticated.
3. **Refresh before expiry.** `POST /auth/refresh-token` (`refresh_token`) exchanges the refresh token for a new access
   token. Refresh proactively — the access token is short-lived (one hour) and no operation on either API is
   idempotent, so a mid-flow 401 followed by a blind retry is unsafe (see `switstack-create-a-payment`).
4. **Revoke on teardown.** `POST /auth/revoke-token` (`revoke_token`) invalidates a token. Do this when a session or
   worker ends rather than letting tokens sit until expiry.

## What you are authorized to do

Authorization is **role-based, not scope-based** — the OAuth flow declares an empty `scopes` map, so the token carries
no scope list to inspect. Know your role before you call:

**Switcloud**
- *Super Admin* — CRUD on all models; Payment data is read-only.
- *Organization Admin* — CRUD on their own organization's estate and configuration; cannot see other organizations'
  data and **cannot create payments**.
- *Simple user (human or machine)* — read-only on their organization's data; **can create and update Payments**.

**Swittest**
- *Data* — read test suites, tests, configs and vcards.
- *Full* — full use of Swittest.

The split matters: the estate/config half and the payment half of Switcloud are deliberately different roles, so a
single credential usually cannot do both. Plan two identities.

## Error handling

The only declared failure is **HTTP 422** with `{"detail":[{"loc","msg","type"}]}`. 401 and 403 are *not* declared even
though every secured operation can return them, so handle them defensively:

- 422 on `/auth/token` — a malformed form body. Check `detail[].loc` for the offending field.
- 401 — treat as an expired or invalid token: refresh once, then re-authenticate. Do not loop.
- 403 — a role boundary, not a transient error. Do not retry; use the correct identity.

Every request and response is `application/json` except the `/auth/token` form body. Do not log `access_token`,
`refresh_token`, `client_secret` or `password`.
