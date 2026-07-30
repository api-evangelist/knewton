---
name: Onboard a learner and get an adaptive recommendation
description: Create an anonymized Knewton account, register the learner into a learning instance, assign a scoped goal, and retrieve the personalized module recommendation for that goal.
api: https://api.knewton.com/v0
docs: https://dev.knewton.com/implementation/getting-started/
generated: '2026-07-19'
method: generated
source: https://dev.knewton.com/reference/
operations:
  - POST /oauth/token
  - POST /accounts
  - POST /learning-instances
  - POST /registrations
  - POST /learning-instances/{li_id}/scoped-goals
  - PUT /learning-instances/{li_id}/scoped-goals/{goal_id}/registrations/{reg_id}
  - GET /registrations/{reg_id}/recommendation
---

# Onboard a learner and get an adaptive recommendation

Knewton publishes no OpenAPI definition, so every step below is grounded in the published
reference at https://dev.knewton.com/reference/ using the documented HTTP method and path.

## Before you start

- You need a partner `api_key` and `api_secret`, provisioned by a Knewton representative.
  There is no self-service sign-up.
- Point non-production work at the sandbox host `https://dev-api.knewton.com/v0`. Production
  is `https://api.knewton.com/v0` (global) or `https://api.knewton.ie/v0` (EMEA).
- You need an ingested knowledge graph. `graph_id` is a UUID or a `gref-` prefixed string.
- Store every UUID Knewton returns. Account ID, learning instance ID, registration ID, goal
  ID and recommendation ID are all needed by later calls.

## Steps

### 1. Get an access token

`POST /oauth/token` with `Authorization: Basic <base64(api_key:api_secret)>` and
`grant_type=client_credentials`. This is the only endpoint that uses Basic auth; the partner
admin user must make the call.

To bind the token to a specific learner, pass `scope=<external_user_id>`. On Knewton the
`scope` parameter is **not** a permission scope — it names the account the tokens are
associated with.

The response carries `access_token`, `refresh_token`, `expires_in` and `expires_at`. Tokens
are long lived (`expires_in` is on the order of 180 days). Send `Authorization: Bearer
<access_token>` on every other call.

### 2. Create the learner account

`POST /accounts` with `external_user_id`. The ID must be unique across all accounts you
create, and **must be completely anonymized — never put PII in it**.

When you are testing, send the header `X-Knewton-Test: true` on this call so the account is
marked as a test account. A learning instance becomes a test instance if the first account
registered to it is a test account.

Keep the returned `id` (UUID).

### 3. Create the learning instance

`POST /learning-instances` with `graph_id` and `name` (optionally `start_date` and
`end_date`). One graph per learning instance. Requires the `create_learning_instance`
entitlement. Keep the returned `id` as `li_id`.

### 4. Register the learner

`POST /registrations` with `account_id`, `learning_instance_id` and `type` set to `learner`
(or `instructor`). Keep the returned `id` as `reg_id`.

### 5. Create and assign a goal

`POST /learning-instances/{li_id}/scoped-goals` with `name`, `targets`, `topics`, `scope`,
`config`, `completion_criteria` and `adaptive_behavior`. Keep the returned `id` as `goal_id`.

Assign it with `PUT /learning-instances/{li_id}/scoped-goals/{goal_id}/registrations/{reg_id}`.
For many learners at once use
`PUT /learning-instances/{li_id}/scoped-goals/{goal_id}/registrations` with `action`,
`registration_type` and `registration_ids`, and read `success` and `failure` in the response.

Note `id`, `config.analytics_enabled`, `config.assign_to`, `targets.completion_behavior` and
`adaptive_behavior` cannot be changed on a later `PUT`.

### 6. Get the recommendation

`GET /registrations/{reg_id}/recommendation?goal_id={goal_id}`.

Optional query parameters: `continued_recommendations=true` to keep getting recommendations
after the goal is complete, and `justifications=true` to get the pedagogical rationale for
the top module.

The response gives `recommendation_id`, `module_ids` (up to 10, in preference order),
`status` (`unfocused`, `in progress`, `ready`) and optionally `stuck`. Render the modules in
the order returned. Ignore `focus_state`; it is deprecated.

**This endpoint is the tightest rate limit on the platform: 15 requests per minute per
token.** Request a recommendation when the learner needs the next activity, not on a poll.

## Rules

- **Rate limits.** On HTTP 429, read `X-Knewton-Rate-Exceeded` (`ip-limited`,
  `token-limited`, `partner-limited`). Knewton has processed no part of the request, so a
  plain resend is safe — back off exponentially. There is no `Retry-After` header.
- **No idempotency.** Knewton documents no idempotency key. Do not blind-retry writes on
  timeouts; re-read state first.
- **Errors.** Failures return `application/json` with `message` and `error_id`. Quote the
  `error_id` to Knewton support. Do not retry 4xx other than 401 (refresh the token) and 429.
- **Entity limits.** 5000 goals per learning instance, 200 goals assigned per registration
  (50 when `config.analytics_enabled` is true), 500 registrations per account, 256 characters
  on any character field. Breaches surface as HTTP 422.
