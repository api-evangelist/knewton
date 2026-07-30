---
name: Stream student events and track goal progress
description: Send graded, ungraded and recommendation-followed events for a Knewton registration, then read back goal status and progress and predicted scores.
api: https://api.knewton.com/v0
docs: https://dev.knewton.com/implementation/sending-events/
generated: '2026-07-19'
method: generated
source: https://dev.knewton.com/reference/
operations:
  - POST /registrations/{id}/graded-events
  - POST /registrations/{id}/ungraded-events
  - POST /registrations/{id}/recommendation-followed-events
  - POST /registrations/{id}/batch-events
  - POST /registrations/{reg_id}/metrics/status-and-progress
  - POST /learning-instances/{li_id}/metrics/status-and-progress
  - POST /accounts/metrics/predicted-score
---

# Stream student events and track goal progress

Grounded in the published reference at https://dev.knewton.com/reference/. Knewton publishes
no OpenAPI definition, so operations are documented HTTP method and path.

Events are the input that makes recommendations adapt. Send them as the learner works, then
read the metrics endpoints for instructor-facing analytics.

## Sending events

All event endpoints hang off a registration.

### Ungraded event

`POST /registrations/{id}/ungraded-events` — `module_id` (required),
`interaction_end_time` (required), `duration`, `is_complete`, `goal_id`.

Use for reading, watching, or any non-assessed interaction.

### Graded event

`POST /registrations/{id}/graded-events` — `module_id` (required),
`interaction_end_time` (required), `is_correct` (required), `duration`, `is_complete`,
`instance_hash`, `goal_id`.

`is_correct` is the signal that drives mastery estimation. Send `instance_hash` when the same
module can be rendered with different generated values so Knewton can tell attempts apart.

### Recommendation-followed event

`POST /registrations/{id}/recommendation-followed-events` — `module_id` (required),
`time_followed` (required), `recommendation_id` (required).

Send this when the learner acts on a module you surfaced from
`GET /registrations/{reg_id}/recommendation`, passing the `recommendation_id` you were
given. This closes the loop and lets Knewton attribute outcomes to its recommendations.

### Batch

`POST /registrations/{id}/batch-events` — `events` (required), `goal_id`,
`ignore_invalid_module_ids`.

- Maximum **500 events** per batch.
- **Events must be listed in reverse chronological order (most recent event first).**
- Accepts `graded-events`, `ungraded-events` and `recommendation-followed` types.
- A partial success returns HTTP **207**; parse the body to find which entries failed.

Do not use `POST /registrations/{id}/focus-events` — it is deprecated. Goals focus
automatically whenever a recommendation is requested for the goal.

## Reading progress

### One learner

`POST /registrations/{reg_id}/metrics/status-and-progress` with `goal_ids` (required).

Returns `query`, `column_headers` (the goal IDs) and `data` — a single-item array with the
registration's `status_and_progress`.

### A whole class

`POST /learning-instances/{li_id}/metrics/status-and-progress` with `goal_ids` (required),
plus `registration_offset` (default 0) and `registrations_per_page` (default 100, **max
100**).

Paginate by reading the `next` object off the response and feeding its `registration_offset`
and `registrations_per_page` into the following request. `next` is absent on the last page.

Each row carries `registration_id` and `status_and_progress` with `status`, `progress` (0-1),
`work_remaining` (0-30, or -1) and per-target objective and module progress.

Only goals created with `config.analytics_enabled: true` are analytics-eligible, and a
registration is capped at 50 such goals.

### Predicted score

`POST /accounts/metrics/predicted-score` with `account_ids`, `graph_ids`,
`grouped_learning_objectives` (each with `group_id` and `learning_objectives` carrying
`learning_objective_id` and optional `weight`) and `confidence_levels`. Optional:
`get_overall_score`, `get_learning_objective_scores`,
`learning_objective_coverage_requirement`.

Returns `account_scores` keyed by account ID, with `group_scores`,
`learning_objective_scores`, `overall_score` and `overall_learning_objective_coverage`.

## Rules

- **Rate limits.** Metrics endpoints are 225/min/token; predicted score is 140/min/partner
  and shared across every token you hold, so a fan-out job can starve interactive traffic.
  Batch reads and schedule them off peak.
- **No idempotency.** There is no idempotency key. A retried event post can double-count. On
  a timeout, prefer reading progress back over blindly resending.
- **429 is safe to resend.** Knewton has processed no part of a rate-limited request. Back
  off exponentially; there is no `Retry-After`.
- **Load testing.** In the sandbox, exceeding event thresholds for over 3 minutes makes
  Knewton return HTTP 204 for every event *without persisting or processing it*, and mock
  recommendations with random `recommendation_id` values, flagged by
  `X-Knewton-Rate-Exceeded: partner-soft-limited`. Do not treat those runs as functional
  verification.
- **PII.** Never place personally identifiable information in `external_user_id` or any
  identifier sent to Knewton.
