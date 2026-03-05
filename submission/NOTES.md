# n8n Incident Notification Workflow

## Overview

This workflow processes incident events, normalizes and validates the payload, and sends notifications to Slack and Office365 using offline mock APIs.

The workflow also implements retry logic, idempotency, and failure persistence.

---

## How to Run

1. Start the mock servers:

```
npm install
npm run mocks
```

Slack mock:

```
http://localhost:4010/chat.postMessage
```

Microsoft mock:

```
http://localhost:4020/me/sendMail
```

2. Start n8n locally.

3. Import `workflow.json`.

4. Trigger the webhook:

```
POST /webhook/incident
```

Example test command:

```
curl -X POST http://localhost:5678/webhook/incident \
-H "Content-Type: application/json" \
-d @fixtures/incidents/INC-10001.json
```

---

## Validation

The workflow validates required fields:

* incidentId
* severity
* title
* createdAt

If any field is missing the workflow throws a clear error.

---

## Normalization

Severity is mapped to numeric priority:

P1 → 1
P2 → 2
P3 → 3
P4 → 4

---

## dedupeKey Formula

The dedupeKey is generated as:

```
dedupeKey = incidentId + "-" + createdAt
```

This ensures replayed events are detected and notifications are not sent twice.

---

## Message Formatting

A clean human-readable message is created and used for both Slack and Email notifications.

The description field is truncated to 240 characters.

---

## Retry and Backoff

Retries occur when the API returns:

* HTTP 429
* HTTP 5xx

Configuration:

* Max attempts: 5
* Linear backoff using a wait node

Other 4xx errors are not retried.

---

## Idempotency / Deduplication

Workflow static storage is used to store processed dedupeKeys.

If the same event is replayed the workflow stops early and does not send notifications again.

---

## Error Handling

If Slack or Office365 continues failing after all retry attempts:

1. The workflow routes to an error branch. (Both workflows must be active, and the error workflow is triggered automatically when a real execution error happens.)
2. A failure record is persisted locally.

Failure record includes:

* incidentId
* dedupeKey
* retryCount
* timestamp
* failed service
