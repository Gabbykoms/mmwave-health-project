# Backend Implementation Plan

Status: proposed implementation plan, based on the agreed Python/FastAPI stack and cloud deployment direction. No backend implementation exists yet.

## 1. Objective and scope

Build the persistent event and caregiver-response backend for the mmWave home safety prototype. The first working system uses simulated detections:

`simulated fall → confirmation → cancellation or timeout → caregiver SMS → acknowledgement → resolution`

This plan supports the wider architecture in [README.md](README.md) and maps implementation milestones to [the MVP backlog](planning/MHS-mvp-tickets.csv).

The initial scope is one home, a configured device, and a configured test caregiver. Public radar dataset replay and live radar integration will use the same detection contract later. ML training, raw radar processing, frontend implementation, automatic emergency-service calls, and preventive monitoring are outside this backend increment.

## 2. Agreed direction and proposed stack

| Component | Choice | Role |
| --- | --- | --- |
| Language and API | Python + FastAPI | HTTP API and generated OpenAPI contract |
| Notifications | Twilio Programmable Messaging; Voice in a later milestone | SMS first, then phone-call escalation |
| Validation | Pydantic | Validate requests, responses, and configuration |
| Persistence | PostgreSQL + SQLAlchemy | Store events, transitions, devices, and delivery work |
| Migrations | Alembic | Version database schema changes |
| Background processing | Separate Python worker | Process deadlines and notification retries |
| Tests | pytest | Unit, database integration, and API checks |
| Local environment | Docker Compose | Run API, worker, and PostgreSQL together |
| Deployment | Cloud server; provider undecided | Host API, worker, and database |

Use PostgreSQL locally as well as in the cloud so concurrency and transaction tests exercise the intended database. Start with one API service and one worker sharing application modules. A separate message broker is not needed for the first implementation: pending work is durable in PostgreSQL.

Build and test locally before internet deployment. Python/package versions, hosting provider, and authentication provider will be selected during setup and pinned or documented where applicable.

## 3. Edge/cloud responsibilities

```text
Edge device                              Cloud
Radar → preprocessing → ML
             ↓
       decision filtering ── HTTPS ──→ FastAPI → PostgreSQL
             ↓                            ↑          ↑
       local warning                 web UI       worker
       durable send buffer                           ↓
       device heartbeat                        caregiver SMS
```

### Edge

- Read radar observations, preprocess, run inference, and filter predictions into suspected falls.
- Send compact detection events rather than raw radar streams.
- Keep stable event IDs and original observation timestamps when retrying delivery.
- Eventually buffer unsent detections durably and provide a local warning without cloud access.
- Send periodic health updates independently of fall events.

The simulator initially substitutes for these functions; implementing durable edge buffering and offline local interaction is a later integration milestone.

### Cloud

- Own the authoritative event lifecycle, confirmation deadlines, and caregiver action timestamps.
- Persist events, transitions, notification work, and delivery outcomes.
- Serve confirmation and caregiver interfaces through the API.
- Process deadlines, SMS retries, and device health status.

Separating the backend reduces edge responsibilities, but inference may still dominate edge resource usage. Measure actual load during integration.

Cloud notification depends on connectivity. Buffered events do not provide immediate caregiver assistance while offline. The interface must distinguish an unavailable connection from a successful cancellation or acknowledgement.

## 4. Event lifecycle

```text
CONFIRMING ── cancel before deadline ──→ CANCELLED
    │
    └── deadline expires ──→ ALERTING ──→ ACKNOWLEDGED ──→ RESOLVED
```

Rules:

- An accepted fresh detection creates one `CONFIRMING` event with a persisted deadline.
- Cancellation succeeds only while `CONFIRMING` and while database time is strictly before the deadline.
- At or after the deadline, cancellation fails even if the worker has not processed expiry yet.
- Expiry changes the event to `ALERTING` and creates durable notification work in the same transaction.
- Only `ALERTING` can become `ACKNOWLEDGED`; only `ACKNOWLEDGED` can become `RESOLVED`.
- `CANCELLED` and `RESOLVED` are terminal.
- Conditional updates or row locks serialize competing actions. Persist the transition record in the same transaction as the event update.
- Repeating an already successful action returns its persisted result without adding another transition; incompatible actions return a conflict.

Delivery status is separate from event state. An event remains `ALERTING` if SMS delivery fails. The dashboard shows the delivery failure explicitly.

Twilio SMS replaces email in the MVP. Voice calls and backup-contact escalation follow in a separate milestone. Bounded retries of a failed SMS handle delivery failures; they do not implement escalation for an unanswered alert.

## 5. Proposed detection contract

Endpoint: `POST /api/v1/detections`

```json
{
  "detector_event_id": "9b668ae7-029e-4b53-b0af-a48dbdd03bea",
  "source": "simulator",
  "sensor_id": "living-room-radar-01",
  "detected_at": "2026-10-09T19:00:00Z",
  "detection_status": "suspected_fall",
  "confidence": 0.94
}
```

- Sources: `simulator`, `dataset_replay`, or `live_radar`.
- Initially accept only `suspected_fall`; confidence is optional and, when supplied, must be between 0 and 1.
- Require timezone-aware timestamps; store and return UTC.
- Derive device ownership and room from registered sensor configuration rather than trusting a caller-supplied home or room.
- Enforce uniqueness on `(sensor_id, detector_event_id)`. Retry the same ID and payload after ambiguous responses.
- A new accepted detection returns `201` with the persisted event. An identical duplicate returns `200` with the existing event, including its current state. Reuse of an ID with different detection data returns `409`.
- Responses include the backend event ID, sensor/room, source, detection and receipt times, state, deadline, action timestamps, and notification summary. Include server time so the UI can account for clock differences.

### Timestamp and delayed delivery policy

Proposed initial policy: create the confirmation deadline from server receipt time for a fresh event, with a configurable confirmation duration. Store the original detection time separately.

Before implementation, choose a maximum detection age and allowed future clock skew. Reject stale or excessively future-dated new detections with explicit error codes; do not create fresh confirmations from them. Check recognized duplicates first so a late retry can still retrieve its original event.

The edge must retain/report rejected stale detections as delivery failures. A later historical ingestion path may record them without starting a fresh alert. Offline cancellation reconciliation is not part of the first contract and needs an explicit design before live deployment.

## 6. API surface

| Method and endpoint | Behavior |
| --- | --- |
| `POST /api/v1/detections` | Create or retrieve a deduplicated event |
| `GET /api/v1/events` | Filter by state and return paginated active events or history |
| `GET /api/v1/events/{id}` | Return event details and delivery summary |
| `GET /api/v1/events/{id}/transitions` | Return ordered lifecycle history |
| `POST /api/v1/events/{id}/cancel` | Cancel before the confirmation deadline |
| `POST /api/v1/events/{id}/acknowledge` | Acknowledge an alert |
| `POST /api/v1/events/{id}/resolve` | Resolve an acknowledged event |
| `POST /api/v1/devices/{id}/heartbeat` | Record authenticated device health |
| `GET /api/v1/devices` | Return authorized device status and last contact time |
| `POST /api/v1/webhooks/twilio/message-status` | Validate and persist SMS delivery updates; never acknowledge an event |
| `GET /health/live` | Confirm the API process responds |
| `GET /health/ready` | Confirm required database access/schema readiness |

Use a consistent error body with a stable code and readable message. Distinguish validation failures, missing records, authentication/authorization failures, and invalid transitions. Return persisted state after actions.

The initial UI can poll every two seconds while a confirmation or active-alert screen is open; revisit this after measurement. The displayed countdown does not trigger state transitions. Agree example requests, responses, errors, and pagination with the interface owner before building against them.

## 7. Persistence model

| Table | Main fields and constraints |
| --- | --- |
| `devices` | ID, sensor identifier, room, enabled status, last heartbeat, health metadata; unique sensor identifier |
| `events` | ID, device/sensor, detector event ID, source, detection status, optional confidence, detected/received times, state, confirmation deadline, cancellation/acknowledgement/resolution times; unique sensor + detector event ID |
| `event_transitions` | ID, event ID, previous/new state, timestamp, actor, reason |
| `notifications` | ID, event ID, channel, recipient, status, attempt count, next attempt time, lease expiry; unique logical initial notification per event/recipient/channel |
| `notification_attempts` | ID, notification ID, attempt number, start/end times, outcome, sanitized provider result/message ID; unique notification + attempt number |

Add indexes for event listing, pending confirmation deadlines, and due notification work. Enforce valid states and relationships through schema constraints and application rules. Persist actor identity from authenticated context rather than request-body claims.

The authentication milestone will add or map caregiver identities, device credentials, and access assignments. A general multi-home tenancy model is deferred; all access still needs explicit authorization for the configured home.

## 8. Worker and notification reliability

- Poll due confirmation deadlines and transition them transactionally.
- Treat `notifications` as a durable outbox: notification creation cannot be lost between committing `ALERTING` and starting SMS delivery.
- Claim delivery work with a database-backed lease; do not hold a database transaction open across a provider request.
- Record attempts before sending and persist outcomes afterward. Expired leases allow recovery after worker failure.
- Proposed retry schedule: three total attempts, with retries after 30 seconds and 120 seconds. Make these values configurable.
- Retry transient failures such as timeouts and temporary provider errors. Mark permanent rejection or exhausted retries as failed and expose that status through the API.
- Stop unclaimed notification work when an event is acknowledged or resolved. A provider request already in flight may still complete.
- On restart, query overdue confirmations and due or abandoned notification work from the database. In-memory timers are not authoritative.
- Provider acceptance is not proof that an SMS reached or was read by the caregiver; name delivery statuses accordingly.

An ambiguous provider response or crash after sending can cause duplicate delivery. Use a stable provider idempotency key where supported, and document the remaining possibility. Do not claim exactly-once SMS delivery.

Use a fake notification provider in automated tests. Send real demo SMS only to the configured test caregiver.

### Twilio integration and escalation

For the MVP, send a brief SMS to the configured test caregiver with a link to the event page. Opening the link does not acknowledge the event; acknowledgement requires an explicit dashboard action. The deployed dashboard requires login. Email is outside the initial notification scope.

Persist Twilio's message SID and associate callbacks with the matching notification attempt. Validate webhook signatures using the Twilio SDK and the externally visible callback URL. Process duplicate and out-of-order callbacks safely so old updates cannot overwrite a later delivery outcome. A delivered SMS never changes the event to `ACKNOWLEDGED`. See [Twilio delivery callbacks](https://www.twilio.com/docs/messaging/guides/track-outbound-message-status) and [webhook signature validation](https://www.twilio.com/docs/usage/webhooks/webhooks-security).

Real callback testing requires a reachable HTTPS webhook endpoint. Local tests use a fake provider and callback fixtures; a deployed callback receiver must validate signatures even while the rest of the demo remains local.

Proposed follow-up escalation sequence:

1. Send the initial SMS when the event becomes `ALERTING`.
2. If still unacknowledged at a persisted deadline, call the primary caregiver.
3. If still unacknowledged after the call response window, notify a configured backup contact.
4. Stop after the configured sequence is exhausted, keep the event active, and show escalation exhaustion on the dashboard.

The backend owns escalation scheduling. Persist the current escalation step, next-action deadline, and contact/channel snapshot, with unique work per event and step. Recovery after restart must resume the sequence without recreating completed steps. Keep the event `ALERTING` during escalation; record steps separately in the audit history.

Voice may offer “Press 1 to acknowledge” using [Twilio Gather](https://www.twilio.com/docs/voice/twiml/gather). An answered or completed call alone is not acknowledgement. Bind keypad callbacks to the expected call SID, event, and configured contact, validate the signature, and apply the normal atomic acknowledgement rule. This proves interaction on the contacted line, not the personal identity of whoever answers; finalize the acceptance policy before enabling it.

Before launching each escalation step, recheck event state transactionally. Acknowledgement stops pending steps, although an SMS or call already in flight may complete. Handle busy, no-answer, failed, and missing-callback cases with bounded deadlines. Exact escalation intervals and the handling of definite delivery failures remain decisions for this milestone.

## 9. Cloud deployment and security

The current MVP tickets explicitly assume a local or trusted demo. Internet deployment is additional scope and requires its own completion gate.

- HTTPS for device, UI, and API traffic.
- Individually provisioned, revocable device credentials; devices can submit only their own detections and heartbeats.
- Authenticated caregiver sessions and authorization on event reads and actions.
- An authenticated confirmation client restricted to the relevant device/home and cancellation action.
- Explicit frontend origin configuration, appropriate CSRF protection for cookie sessions, request-size limits, and rate limits.
- Database accessible only to backend components; secrets supplied through deployment configuration and excluded from source control and logs.
- Controlled migrations, service restart policies, database backups, and a documented restore check.
- Structured logs with event/request IDs, worker last-success status, queue age, notification failures, and device last-seen times.
- A documented data-retention policy; avoid storing raw radar data or unnecessary personal information.

Device offline status derives from missed heartbeats rather than absence of fall events. Heartbeat interval and offline threshold remain configurable decisions.

## 10. Proposed repository layout

```text
backend/
  app/
    main.py
    api/                 # Routes and request/response schemas
    core/                # Configuration, authentication, logging
    db/                  # Sessions and models
    services/            # Lifecycle and ingestion rules
    notifications/       # Provider interface and delivery logic
    worker.py            # Deadlines and pending work
  migrations/
  tests/
  scripts/
    simulate_detection.py
  pyproject.toml
  Dockerfile
compose.yaml
.env.example
```

Keep HTTP handlers thin. API requests and the worker call the same lifecycle functions. Inject a clock and notification-provider interface where useful for deterministic tests.

## 11. Implementation milestones

| Milestone | Deliverables | Backlog mapping |
| --- | --- | --- |
| 1. Contract and foundation | Confirm lifecycle/API examples; scaffold FastAPI, PostgreSQL, migrations, Compose, health endpoints, CI | MVP-01, MVP-02; coordinate MVP-03 |
| 2. Persistent simulated detection | Simulator, validation, event creation, persisted deadline, deduplication | MVP-04; support MVP-05 |
| 3. Confirmation and notification | Atomic cancellation/expiry, outbox, fake/Twilio SMS adapters, bounded retries, event-list API | MVP-06, MVP-07; support MVP-08, MVP-09 |
| 4. Caregiver lifecycle and recovery | Acknowledge/resolve, history, pending-work recovery, failure checks, setup/demo documentation | MVP-10, MVP-12; support MVP-11, MVP-13 |
| 5. Cloud readiness | Authentication, authorization, device provisioning/heartbeats, HTTPS deployment, backups, operational checks | Additional tickets required |
| 6. Voice and backup-contact escalation | Persisted acknowledgement deadlines, call flow, validated keypad callbacks, backup contact, restart recovery, bounded exhaustion | Additional tickets required |
| 7. Edge integration | Live/replay producer, durable edge buffering, local warning, delayed-event handling and offline reconciliation | Later radar/system integration work |

The existing estimates do not cover the added cloud-readiness, voice escalation, and edge-offline work; re-estimate MVP-07 for Twilio callbacks as well. Re-estimate those separately. First implementation target: a single command starts the local stack, health responds, and a simulated detection creates exactly one persisted `CONFIRMING` event across repeated submissions.

## 12. Validation and acceptance

Use PostgreSQL integration tests for actual database constraints and races, alongside focused unit tests for lifecycle policy.

- Valid detection persists; identical and concurrent duplicates produce one event.
- Conflicting ID reuse, invalid payloads, stale timestamps, and excessive future skew fail clearly.
- Cancellation before expiry produces no notification; cancellation exactly at or after expiry fails.
- Concurrent cancellation and expiry have one valid outcome and one transition history.
- Expiry creates one logical notification; transient failures retry, permanent failures stop, and exhausted retries are visible.
- SMS callbacks reject invalid signatures, tolerate duplicates/out-of-order updates, and never imply caregiver acknowledgement.
- Voice milestone: cover no-answer/busy calls, keypad acknowledgement, acknowledgement racing escalation, exhausted contact steps, and restart with a pending escalation deadline.
- Acknowledgement/resolution enforce allowed transitions and repeated requests do not duplicate history.
- Restart with a pending or overdue confirmation recovers correctly.
- Restart around notification claiming/sending recovers work and exposes the documented duplicate-delivery limitation.
- History, timestamps, pagination, and current state remain consistent after refresh/restart.
- Cloud gate: unauthenticated and unauthorized requests fail; a device cannot act as another device; missing heartbeats produce offline status.

Record API latency, deadline-processing delay, and notification-attempt delay separately. Use a controllable clock for boundary tests rather than long sleeps. CI should run formatting/lint checks and relevant automated tests with a disposable PostgreSQL database.

## 13. Decisions to finalize

Before contract implementation:

1. Confirmation duration, maximum detection age, and permitted clock skew.
2. Exact response/error schemas and repeated-action behavior with the interface owner.
3. Initial sensor/room configuration and test caregiver configuration.

Before cloud deployment:

1. Hosting region/provider and budget.
2. Twilio sender/number setup for the destination country, account requirements, and usage budget.
3. Caregiver login, confirmation-client authentication, and device provisioning approach.
4. Heartbeat interval, offline threshold, retention period, backup policy, and operational ownership.

Before voice escalation:

1. Acknowledgement timeout, call response window, and backup-contact order.
2. Keypad acknowledgement policy, call script, and callback endpoints.
3. Delivery-failure escalation policy and maximum attempts/cost per event.

Before live edge integration:

1. Local warning/confirmation hardware or UI.
2. Durable buffering limits and historical handling of stale detections.
3. Reconciliation when a local cancellation occurs while the cloud is unreachable.
