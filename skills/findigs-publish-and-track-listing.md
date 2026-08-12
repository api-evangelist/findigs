---
name: Publish a Findigs listing and track its applications
description: >-
  Create a rental listing in Findigs, then follow the applications and application groups that arrive
  against it through to a decision. Covers the only write path the Findigs Client API exposes.
api: openapi/findigs-client-api-openapi.yml
operations:
  - post_listing_listings__post
  - get_listing_listings__listing_id__get
  - patch_listing_listings__listing_id__patch
  - get_groups_groups__get
  - get_applications_applications__get
generated: '2026-08-12'
method: generated
source: openapi/findigs-client-api-openapi.yml + https://docs.getfindigs.com/
---

# Publish a Findigs listing and track its applications

## Before you start

- **Access is not self-serve.** Findigs states the Client API is not available to new clients. Keys are
  issued by the Findigs team; contact `integrations@findigs.com`. Verify you have a key before doing
  anything else.
- **Base URLs.** Sandbox `https://api.sandbox.findigs.com`, production `https://api.client.findigs.com`.
  There is no key prefix telling the two apart — track which key belongs to which host yourself.
- **Auth.** Send the scoped key as the `X-API-KEY` request header on every call. A missing or invalid
  key returns `403 {"detail":"Not authenticated"}`, which is **not** declared in the spec.
- **This API returns regulated data.** Application and group responses carry FCRA consumer report data
  and applicant PII. Do not log, cache or forward response bodies without checking the handling rules
  that apply to you.

## 1. Create the listing

Call `post_listing_listings__post` (`POST /listings/`). The body is `ListingCreate` and all four fields
are required:

- `rent_amount` — number, minimum 0, rent in dollars
- `date_available` — date, first move-in date
- `status` — one of `LISTED` (appears in search, accepts applications), `LOCKED` (hidden from search,
  still accepts applications), `CLOSED` (neither)
- `address` — `address_line_1`, `city`, `state` (exactly 2 letters), `postal_code` (`12345` or
  `12345-6789`); `address_line_2` optional

Keep the returned `id` (uuid) and `url` — `url` is the applicant-facing apply link you hand to renters.

> **There is no idempotency key.** A retried `POST /listings/` creates a second listing. Persist the
> returned `id` before you retry anything, and reconcile with `get_listings_listings__get` rather than
> re-posting blind.

> **In the sandbox only,** posting a listing automatically generates applications and groups against it,
> so you can exercise the read paths immediately.

## 2. Adjust the listing as leasing progresses

Call `patch_listing_listings__listing_id__patch` (`PATCH /listings/{listing_id}`) with `ListingUpdate`.
Every field is optional and only the ones you send are changed. The normal lifecycle move is
`LISTED` → `LOCKED` once you have enough applicants in flight, then `CLOSED` on signature.

Read back with `get_listing_listings__listing_id__get`.

## 3. Watch the groups, not the applications

A **group** is the decision unit — the set of people applying together. Poll
`get_groups_groups__get` (`GET /groups/`) filtered by your listing:

- `listing_ids` — repeated uuid parameter
- `status` — repeated, from `incomplete`, `submitted`, `passed`, `pending_review`, `declined`,
  `approved`, `onboarded`, `cancelled`
- `updated_at__gte` / `updated_at__lte` — date-time window; use `updated_at__gte` with your last poll
  timestamp so you only pull what moved
- `page` (min 1, default 1) and `size` (min 1, **max 100**, default 50)

Read `total` and `pages` off the `Page` envelope and walk `page` until you have them all.

## 4. Drill into individual applications

Call `get_applications_applications__get` (`GET /applications/`) to list, or
`get_application_applications__application_id__get` for one.

> **Hard constraint:** on `GET /applications/` the `groups` and `listings` filters are **mutually
> exclusive**. Sending both returns `400`. Pick one.

Pass `embed_bi_data=true` on the single-application and single-group reads only if you actually need
the `bi_data` object; it is untyped in the contract, so do not depend on its shape.

## Error handling

- `422` — Pydantic validation. Read `detail[].loc` to find the offending field. Usual causes: a
  malformed uuid, `size` over 100, `page` under 1, a `status` outside the enum, a `state` that is not
  two letters, or a `postal_code` failing the pattern.
- `403` — bad or missing `X-API-KEY`.
- `400` — both `groups` and `listings` sent to `GET /applications/`.
- **No rate limits are published** and no `429` is declared. Back off conservatively on your own
  schedule; there is no `Retry-After` to read.
