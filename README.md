# Switstack

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **Not from the company, and here with a question?** You are welcome here — we would rather be the
> front line and point you the right way than have a good report go nowhere. What this repository
> can answer is narrow, though, so it is worth knowing who you are actually looking for:
>
> - **A question about how the API works, an account, billing, or a bug in the service** — that is
>   the company's own support, not us. We profile this API; we do not operate it and cannot see
>   your account.
> - **A bug in an open-source project we only catalog** — file it on that project's own repository.
>   This has happened with a real and correct bug report that reached us instead of the people who
>   could fix it, which helped nobody.
> - **Anything about this listing itself** — the description, the tags, the rating, a missing or
>   wrong artifact — is ours. Open an issue here.
> - **Not sure, or something general about API Evangelist or APIs.io** — open an issue on the
>   [APIs.io Inbox](https://github.com/api-search/inbox) and we will route it.
>
> **This repository contains no software, and we will never ask you to download anything.** There is
> no build, release, installer, or binary here — only text and machine-readable API descriptions, so
> there is nothing here that can be "corrupt" or need "repairing". Any issue, comment, or email
> claiming otherwise and offering a download link is not from us and is hostile. Do not follow the
> link; it is a lure. Report it to GitHub and, if you like, tell us at
> [info@apievangelist.com](mailto:info@apievangelist.com) so we can take it down.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

Switstack is a payment infrastructure company building software-defined EMV acceptance for physical retail. It ships
three products: **switstack moka**, a source-available EMV Level 2 kernel stack with brand Letters of Compliance;
**Switcloud**, a hosted "universal compliance infrastructure" that abstracts EMV L2 execution and centralizes estate
(organization/merchant/store/POI) and EMV configuration (CAPK, BIN, CRL, kernel parameter) management behind a REST API;
and **Swittest**, a managed EMV functional test-automation service for labs and developers certifying card-present
applications. The platform is terminal-agnostic and L2-stack agnostic — one L3 application runs across Android COTS
tap-to-pay devices and PCI-PTS terminals via a GLA adapter layer — and Switstack supplies pre-certified PCI MPoC
software and service components to shrink certification scope.

## What this profile holds

Two complete OpenAPI 3.1.0 documents, harvested from Switstack's own published API reference:

| API | Version | Operations | Reference |
|---|---|---|---|
| Switcloud API | 2.28.0 | 106 across 40 paths | https://docs.switstack.io/api/switcloud/api/switcloud-srv/ |
| Swittest API | 0.13.0 | 22 across 22 paths | https://docs.switstack.io/api/swittest/api/swittest-srv/ |

Alongside them: the authentication profile, API conventions, error catalogue, entity-relationship model, lifecycle,
conformance (web-API standards and the EMV/PCI MPoC scope Switstack publishes), packages read from Switstack's own
public Sonatype Nexus, the Swittest CLI command surface, sandbox and test-tooling, five Agent Skills and five Arazzo
workflows grounded in verified operationIds, plus probed records of what is **not** published (no pricing page, no
changelog, no status page, no terms or privacy policy, no `/.well-known/*` document, no agent card, and — despite the
homepage headline — no MCP server).

Source: portfolio company of [serena](https://github.com/api-evangelist/serena) — https://www.switstack.io/
