---
name: Consume Findigs decision webhooks and reconcile state
description: >-
  Receive the nine Findigs webhook events, then call the matching read operation to fetch the object's
  current state. This is the intended integration pattern — Findigs webhooks are deliberately terse.
api: openapi/findigs-client-api-openapi.yml
operations:
  - get_group_groups__group_id__get
  - get_application_applications__application_id__get
  - get_groups_groups__get
generated: '2026-08-12'
method: generated
source: https://docs.getfindigs.com/ + asyncapi/findigs-webhooks.yml
---

# Consume Findigs decision webhooks and reconcile state

## The pattern

Findigs webhooks carry **only** an object id, an event name and a timestamp. They are a nudge, not a
payload. On every event you call the corresponding `GET` to fetch current state.

```json
{
  "object_id": "45b7476b-262c-4b6c-91da-7d9b5ba83e5a",
  "event_name": "object_type.event_name",
  "event_detail": { "event_at": "2022-11-17T17:51:57.628894" }
}
```

## Setup

There is **no webhook-management endpoint** in the API. You give Findigs a URL that accepts an HTTP
`POST`; they configure it. Ask `integrations@findigs.com`.

## Respond correctly or you will be retried

Return **HTTP 200** as soon as you have durably queued the event. Anything else counts as a failure,
and Findigs retries **up to 9 times over the 24 hours** after the first failure. Do the read call
asynchronously — do not hold the webhook response open while you fetch.

**No signature or shared secret is documented.** Treat the endpoint as unauthenticated input: use an
unguessable path, allowlist by source if you can, and never trust the payload as proof of state.
Validate by reading back from the API, which is the pattern anyway.

## Event → read operation

| `event_name` | Object | Then call |
|---|---|---|
| `applications.started` | application | `get_application_applications__application_id__get` |
| `applications.submitted` | application | `get_application_applications__application_id__get` |
| `groups.ready_for_review` | group | `get_group_groups__group_id__get` |
| `groups.approved` | group | `get_group_groups__group_id__get` |
| `groups.declined` | group | `get_group_groups__group_id__get` |
| `groups.cancelled` | group | `get_group_groups__group_id__get` |
| `groups.decision_reversed` | group | `get_group_groups__group_id__get` |
| `groups.custom_status_changed` | group | `get_group_groups__group_id__get` |
| `groups.assigned_user_changed` | group | `get_group_groups__group_id__get` |

Pass `object_id` as the `group_id` / `application_id` path parameter with your `X-API-KEY` header.

## Reading the result

- `Group.status` is the decision: `incomplete`, `submitted`, `passed`, `pending_review`, `declined`,
  `approved`, `onboarded`, `cancelled`.
- `Group.workflow_status` is the operator's own custom status — that is what
  `groups.custom_status_changed` refers to, not `status`.
- `Group.decline_details` holds decline reasoning, but it is an **untyped object** with no enumerated
  vocabulary. Do not build adverse-action logic that assumes a fixed shape.
- `Group.applications[]` embeds the full `Application` objects, so one group read usually saves you
  the per-application calls.

## Ordering and gaps

Delivery ordering is **not documented** and there is no sequence number — `groups.decision_reversed`
can land out of order relative to `groups.approved`. Treat the API read as the source of truth and the
event as a trigger only. Because retries span 24 hours, an event can also arrive long after the fact;
compare `event_detail.event_at` against the object's `updated_at` before acting.

To backfill anything you dropped, sweep `get_groups_groups__get` with
`updated_at__gte` set to your last confirmed checkpoint, paging with `page`/`size` (max 100).
