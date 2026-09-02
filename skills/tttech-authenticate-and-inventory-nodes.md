---
name: Authenticate to a Nerve Management System and inventory the node estate
description: >-
  Log in to a Nerve Management System, handle multi-factor authentication, then list and filter the
  registered edge nodes and the labels that target them. This is the entry point for every other Nerve
  skill — nothing else works until the session exists.
api: openapi/tttech-nerve-management-system-openapi.yml
operations:
  - mfa_is_enabled
  - login
  - mfa_validate
  - get_nodes_filtered
  - get_node_by_serial_number
  - get_labels
  - get_label_group
  - logout
generated: '2026-09-01'
method: generated
source: openapi/tttech-nerve-management-system-openapi.yml + https://docs.nerve.cloud/developer_guide/ms-api/
---

# Authenticate and inventory the node estate

The base URL is the customer's own Management System host. TTTech's published specification names
`https://trynerve1.nerve.cloud` (the hosted trial system); a production system is a different host, and
the docs tell you to replace the `servers[]` URL accordingly. There is no global api.nerve.cloud.

## 1. Check whether MFA is on

`mfa_is_enabled` (`GET /auth/mfa/is-enabled`) is unauthenticated and tells you whether this Management
System requires a second factor. Branch on it before you send credentials.

## 2. Log in

`login` (`POST /auth/login`) exchanges username and password for a session. The Management System's
global security requirement is the `sessionId` header — send it on every subsequent call. A `cookie`
header is accepted as an alternative, and a subset of operations accept HTTP basic or bearer.

If MFA is enabled, `login` does not complete the session on its own: follow it with `mfa_validate`
(`POST /auth/mfa/validate`), which validates the code and logs the user in.

## 3. Respect brute-force protection

Nerve counts failed logins. Five consecutive failures block that user from the same IP for 30 minutes;
100 failures from one IP block the address for 24 hours. Exhaustion returns **429** with an
`{errors: [...]}` body and **no `Retry-After` header** — you must back off on your own clock. From Nerve
3.1.0 an operator can retune these through Node DNA (`maxFailedLoginAttemptsBeforeLockout`,
`loginAttemptWindowMinutes`, `lockoutDurationMinutes`), so never hard-code the 5/30 numbers as a
guarantee. Never retry a failed login in a loop.

## 4. List nodes

`get_nodes_filtered` (`GET /nerve/nodes/filtered/list`) is the paged, filterable estate view — it takes
`page`, `limit`, `filterBy` and `order`. Page through it; do not assume one response is the whole estate.

`get_node_by_serial_number` (`GET /nerve/node`) fetches a single node. **Serial numbers must be sent in
all capital letters** — the documentation states this explicitly for every function that takes one, and
the Management System schema pins the pattern to `([A-Z0-9]){12}`.

## 5. Read the labels

`get_labels` (`GET /nerve/labels/list`) and `get_label_group` (`GET /nerve/labels/group`, labels grouped
by key) give you the key/value tags attached to nodes. Labels are how deployments are targeted, so this
is the vocabulary you need before deploying anything.

## 6. Log out

`logout` (`GET /auth/logout`) ends the session. Session lifetime, concurrent sessions per user and
inactivity logout are operator-configured through Node DNA from 3.1.0 onward.

## Errors

Management System errors are `application/json` with `{errorCode, message}`, where `errorCode` is a
translation key for the Nerve UI — not RFC 9457 problem details. See
`errors/tttech-problem-types.yml`. There is **no idempotency key** in this API and **no request-id
header**; traceability comes from the Management System audit log.
