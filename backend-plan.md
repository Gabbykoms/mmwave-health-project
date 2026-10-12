# Backend Implementation Plan

Status: draft. Nothing is built yet. We have settled on Python with FastAPI, PostgreSQL and Twilio.

## 1. Objective and scope

The backend stores fall events and tracks them until a caregiver responds. For the first build, detections are simulated. The flow is:

`simulated fall → confirmation → cancellation or timeout → caregiver SMS → acknowledgement → resolution`

Who does each step:

- The device sends the detection. In the first build, a simulator stands in for the device.
- The person confirms they are okay, which cancels the event, or does not answer before the countdown ends.
- The backend sends the SMS to the caregiver.
- The caregiver acknowledges the alert, then resolves it.

This plan covers seven milestones (Section 11).

- **Milestones 1–4 are the first build:** one home, one device and one test caregiver, running locally. We keep it this small so we can get one alert right before adding more.
- **Milestones 5–7 come later:** cloud deployment, voice escalation, and the real radar (live, or replayed from a dataset). The radar will send the same kind of detection the simulator sends, so the backend does not change when it arrives.

The first thing we will get working: a single command starts the local stack, the health check responds, and a simulated detection creates exactly one saved `CONFIRMING` event, even when the same detection is sent more than once.

Not part of the backend:

- Radar processing and ML training. The edge device and the ML work handle these. The backend only receives the detection.
- The frontend. The backend supplies the data the screens show.
- Automatic emergency-service calls. The project does not make them at all.
- Preventive monitoring, meaning watching for slow changes in someone's activity over days or weeks. It is a stretch goal for the project, not part of this work.

See [README.md](README.md) for the overall architecture and the MVP features.

Words used in this plan:

| Word | Meaning |
| --- | --- |
| Event | One suspected fall and everything that happens about it afterwards. |
| Person | The one being monitored at home. |
| Caregiver | The one who gets the alert. |
| Device | The radar and the small computer beside it, in the home. Also called the edge. |
| Confirmation screen | Where the person cancels. |
| Caregiver dashboard | Where the caregiver acknowledges and resolves. |
| Worker | The background program that handles deadlines and sends SMS. |

## 2. Stack

| Part | Choice | What we use it for |
| --- | --- | --- |
| Language and API | Python + FastAPI | The HTTP API. FastAPI also generates the OpenAPI contract from the code. |
| Notifications | Twilio Programmable Messaging; Twilio Voice in a later milestone | SMS first, then phone-call escalation. |
| Validation | Pydantic | Checks requests, responses and configuration. |
| Database | PostgreSQL + SQLAlchemy | Stores events, transitions, devices and delivery work. |
| Migrations | Alembic | Keeps a versioned record of each database schema change. |
| Background work | A separate Python worker | Handles confirmation deadlines and notification retries. |
| Tests | pytest | Unit tests, database integration tests and API checks. |
| Local setup | Docker Compose | Runs the API, the worker and PostgreSQL together. |
| Hosting | TBD | A cloud server for the API, the worker and the database. The provider is not chosen. |

We use PostgreSQL locally as well as in the cloud, so the concurrency and transaction tests run against the same database we deploy on.

We start with one API service and one worker. They share the same application modules, so each rule is written once.

We do not use a message broker in the first build, because pending work is already saved in PostgreSQL. The cost is that the worker finds work by polling the database, so a deadline can be noticed up to one poll interval late.

## 3. Edge and cloud responsibilities

```text
Edge device                              Cloud
Radar → preprocessing → ML
             ↓
       decision filtering ── HTTPS ──→ FastAPI → PostgreSQL
             ↓                            ↑          ↑
       local warning (later)           web UI     worker
       durable send buffer (later)                   ↓
       device heartbeat (later)                caregiver SMS
```
Web UI will be the confirmation screen, where the person cancels and the caregiver dashboard, where the caregiver acknowledges and resolves.

In the first build, the simulator stands in for everything on the device side. Durable buffering on the device and local interactin without the cloud come in a later integration milestone (Section 11).

### What the device does

The device reads the radar, preprocesses the data, runs the ML model, and filters the predictions into suspected falls. The backend does none of this.

### What we expect from the device

- It sends small detection events, not raw radar streams. The backend never stores or sees raw radar data.
- When it resends a detection, it keeps the same event ID and the original observation time. This lets the backend recognise a repeat instead of creating a second event.
- Later, it sends regular health updates (heartbeats), separately from fall events. A quiet device is then different from a broken one.
- Later, it saves unsent detections so they survive a restart, and it gives a local warning even without cloud access.

### What the cloud does

- The cloud is the source of truth for event state, confirmation deadlines, and the times of caregiver actions. One clock and one record mean the screens cannot disagree.
- It saves events, transitions, notification work, and delivery outcomes.
- It serves the confirmation screen and the caregiver dashboard through the API.
- It processes deadlines, SMS retries, and device health status.

### If the internet is down

> **Safety limit.** The caregiver alert is sent from the cloud. If the home has no connection, the caregiver is not alerted. A detection saved on the device for later does not get the person help while the connection is down.

Both screens must tell a lost connection apart from a successful action. If a cancel or an acknowledgement does not reach the cloud, the screen shows an error. It never shows success for an action the cloud did not record.

## 4. Event lifecycle

```text
CONFIRMING ── cancel before deadline ──→ CANCELLED
    │
    └── deadline expires ──→ ALERTING ──→ ACKNOWLEDGED ──→ RESOLVED
                                │
                                └── SMS fails: the event stays ALERTING
```

### Rules

- An accepted new detection creates one `CONFIRMING` event with a saved deadline. The deadline is stored in the database, so it survives a restart.
- Cancel works only while the event is `CONFIRMING` and the database's time is strictly before the deadline. We judge by the database's clock because phone and device clocks cannot be trusted.
- At the deadline or after it, cancel fails, even if the worker has not processed the expiry yet. The deadline decides the outcome, not how fast the worker runs.
- When the deadline passes, the worker changes the event to `ALERTING` and creates the notification work in the same transaction. Either both are saved or neither is, so an alert cannot exist without its SMS waiting to be sent.
- Only `ALERTING` can become `ACKNOWLEDGED`. Only `ACKNOWLEDGED` can become `RESOLVED`.
- `CANCELLED` and `RESOLVED` are final. Nothing changes an event after that.
- Every change of state writes a transition record in the same transaction as the event update, so the history always matches the event.

### Two actions at the same moment

A cancel and an expiry can arrive together. We settle this with a conditional update: the change only applies if the event is still in the expected state. For example, cancel runs:

```sql
UPDATE events
SET state = 'CANCELLED'
WHERE id = :id
  AND state = 'CONFIRMING'
  AND now() < confirmation_deadline;
```

Then we check how many rows changed. One row means the cancel won. Zero rows means it was too late or the event had already moved on. The database lets only one of two competing updates succeed, so there is always exactly one outcome.

### What each action returns

Repeating an action that already succeeded returns the saved event and adds no new transition. This makes a retry safe. An action that does not fit the current state returns a conflict.

| Current state | cancel | acknowledge | resolve |
| --- | --- | --- | --- |
| `CONFIRMING`, before the deadline | 200, becomes `CANCELLED` | 409 | 409 |
| `CONFIRMING`, at or after the deadline | 409 | 409 | 409 |
| `CANCELLED` | 200, no change | 409 | 409 |
| `ALERTING` | 409 | 200, becomes `ACKNOWLEDGED` | 409 |
| `ACKNOWLEDGED` | 409 | 200, no change | 200, becomes `RESOLVED` |
| `RESOLVED` | 409 | 200, no change | 200, no change |

### Delivery is separate from state

- Whether the SMS was delivered is tracked separately from the event's state. If the SMS fails, the event stays `ALERTING` and the caregiver dashboard shows the failure.
- A delivered SMS never counts as acknowledgement. Only an explicit action by the caregiver moves the event to `ACKNOWLEDGED`. This applies everywhere in the plan.

### What the first build does not do

> **Safety limit.** If an SMS fails to send, we retry it a limited number of times (Section 8). If the SMS is sent and nobody answers, nothing else happens until milestone 6. Retrying a failed SMS is not escalation.

- The MVP notifies by SMS through Twilio. Email is not used.
- Voice calls and a backup contact come in a separate, later milestone.

## 5. Detection contract

Endpoint: `POST /api/v1/detections`

The device calls this to report a suspected fall. Until milestone 5 adds device credentials, the endpoint is open: anyone who can reach it can send a detection.

### Request

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

- `source` is `simulator`, `dataset_replay` or `live_radar`.
- `detection_status` must be `suspected_fall`. We accept nothing else for now.
- `confidence` is optional. When present, it must be between 0 and 1.
- `detected_at` must include a timezone. We store and return all times in UTC.
- The request does not carry a home or a room. We look these up from the sensor's registration, because the caller could get them wrong or lie.

### Response

The response is the saved event:

```json
{
  "id": "5f0c1d1e-8a40-4b7e-9d0a-2b6f3c1a7e55",
  "sensor_id": "living-room-radar-01",
  "room": "Living room",
  "source": "simulator",
  "detected_at": "2026-10-09T19:00:00Z",
  "received_at": "2026-10-09T19:00:01Z",
  "state": "CONFIRMING",
  "confirmation_deadline": "2026-10-09T19:00:31Z",
  "cancelled_at": null,
  "acknowledged_at": null,
  "resolved_at": null,
  "notification": null,
  "server_time": "2026-10-09T19:00:01Z"
}
```

`server_time` is the backend's clock at the moment of the reply. The confirmation screen uses it to draw the countdown correctly even when the viewer's own clock is wrong. `notification` holds a short summary of SMS delivery once the event is `ALERTING`.

### What the endpoint returns

We check in this order and stop at the first match:

| # | Situation | Result |
| --- | --- | --- |
| 1 | The request is not valid. | Rejected. No event. |
| 2 | The sensor is unknown or disabled. | Rejected. No event. |
| 3 | We have already seen this `sensor_id` and `detector_event_id`, with the same data. | `200` with the existing event in its current state. |
| 4 | We have already seen this `sensor_id` and `detector_event_id`, with different data. | `409`. |
| 5 | The detection is too old, or dated too far in the future. | Rejected with an explicit error code. No event. |
| 6 | The sensor already has an event in `CONFIRMING` or `ALERTING`. | `200` with that active event. No new event and no new SMS. |
| 7 | None of the above. | `201` with the new `CONFIRMING` event. |

Why:

- **Duplicates (3 and 4).** If the device gets no response, it resends the same request with the same ID. The ID makes that safe. The database enforces that `(sensor_id, detector_event_id)` is unique, so two copies arriving together still produce one event.
- **Duplicates come before the age check.** A late retry of a detection we already accepted must still get its original event back, not a "too old" error.
- **Repeat fires (6).** A detector can fire several times for one fall, each time with a new `detector_event_id`. One fall means one event, one alert and one SMS.

### Deadline and old detections

- The confirmation deadline is the time we received the detection plus the confirmation duration, which is a setting. We use our receipt time, not `detected_at`, so the person always gets the full countdown. `detected_at` is stored separately.
- We reject a new detection that is too old or dated too far in the future. A fall reported long after it happened must not start a fresh countdown as if it had just occurred. Both limits are settings.
- The device keeps a detection we rejected as too old, and reports it as a delivery failure.

## 6. API surface

| Method and endpoint | Caller | What it does | Milestone |
| --- | --- | --- | --- |
| `POST /api/v1/detections` | Device | Creates an event, or returns the one that already exists (Section 5). | 2 |
| `GET /api/v1/events` | Both screens | Lists events, filtered by state, one page at a time. Covers active events and history. | 3 |
| `GET /api/v1/events/{id}` | Both screens | Returns one event with its SMS delivery summary. | 2 |
| `GET /api/v1/events/{id}/transitions` | Caregiver dashboard | Returns the event's history in order. | 4 |
| `POST /api/v1/events/{id}/cancel` | Confirmation screen | Cancels before the deadline. | 3 |
| `POST /api/v1/events/{id}/acknowledge` | Caregiver dashboard | Acknowledges an alert. | 4 |
| `POST /api/v1/events/{id}/resolve` | Caregiver dashboard | Resolves an acknowledged event. | 4 |
| `POST /api/v1/webhooks/twilio/message-status` | Twilio | Saves SMS delivery updates after checking Twilio's signature. | 3 |
| `GET /health/live` | Anyone | Confirms the API process responds. | 1 |
| `GET /health/ready` | Anyone | Confirms the database is reachable and its schema is ready, and that the worker has run recently. | 1 |
| `POST /api/v1/devices/{sensor_id}/heartbeat` | Device | Records that the device is alive. | 5 |
| `GET /api/v1/devices` | Caregiver dashboard | Returns each device's status and last contact time. | 5 |

Login endpoints arrive with milestone 5 and voice callbacks with milestone 6.

Rules for every endpoint:

- **One device ID.** Clients always use `sensor_id`. The database's own device ID stays internal, so two endpoints cannot disagree about which device is meant.
- **Actions return the event.** Cancel, acknowledge and resolve return the full updated event, the same shape as `GET /api/v1/events/{id}`.
- **One error shape.** Every error has a stable code for programs and a message for people:

  ```json
  {"error": {"code": "invalid_transition", "message": "This event can no longer be cancelled."}}
  ```

  We use different codes for a bad request, a missing record, a login or permission failure, and an action that does not fit the event's state.

Rules for the screens:

- While a confirmation or an active alert is open, the screen asks the API for the latest state every two seconds.
- The countdown on the screen is only a display. It never changes an event. Only the saved deadline does (Section 4).
- When the countdown reaches zero, the screen disables the cancel button, because a cancel would fail from that moment.

## 7. Persistence model

All times are stored as `timestamptz`, in UTC.

**`devices`** — one row per sensor.

- Sensor ID, room, enabled or not, last heartbeat, health details.
- The sensor ID is unique.

**`events`** — one row per suspected fall.

- Device, detector event ID, source, detection status, optional confidence.
- Detected time, received time, state, confirmation deadline.
- Cancelled, acknowledged and resolved times.
- Sensor ID plus detector event ID is unique. This is what stops duplicates.
- The database rejects any state outside the five in Section 4.

**`event_transitions`** — the history: one row per change of state.

- Event, previous state, new state, time, actor, reason.

**`notifications`** — one row per message we owe someone. This is the worker's to-do list.

- Event, channel, recipient, status, attempt count, next attempt time, lease expiry.
- At most one first notification per event, recipient and channel.

**`notification_attempts`** — one row per try at sending.

- Notification, attempt number, start and end times, outcome, Twilio's message ID and cleaned-up result.
- Notification plus attempt number is unique.

We add indexes for the three lookups that run constantly: listing events, finding `CONFIRMING` events whose deadline has passed, and finding notifications due for sending.

Rules:

- **One home.** There is no `homes` table. The first build has a single home, so storing it would add nothing.
- **The caregiver's phone number is in `.env`.** Real contacts get tables when login arrives in milestone 5.
- **The actor is never taken from the request body**, because a caller could claim to be anyone. Until login exists, it is a fixed label: `system`, `device:<sensor_id>` or `caregiver:test`. After milestone 5 it comes from the logged-in identity.
- **Phone numbers are personal data.** We do not write them to logs.
- **Escalation needs more schema.** The step, its deadline and a snapshot of the contact are added by a migration in milestone 6.

## 8. Worker and notification reliability

### The worker

The worker wakes on a timer and looks in the database for work that is due. It keeps no timers in memory, so after a restart it finds overdue deadlines and unfinished sends on its own.

- **Deadlines.** It finds `CONFIRMING` events past their deadline and moves them to `ALERTING` (Section 4).
- **Nothing gets lost.** The notification row is written in the same transaction that sets `ALERTING`, so a crash between the two cannot lose the SMS. This pattern is called an outbox.
- **Claiming work.** Before sending, the worker marks the notification as taken until a set time, called a lease. If the worker dies, the lease runs out and the work is picked up again.
- **No transaction held open during a send.** A call to Twilio can take seconds. Holding a database transaction that long would block other work.
- **Each attempt is recorded before sending** and its outcome saved afterwards, so a crash mid-send leaves a trace.
- **Retries.** Three attempts in total, with retries after 30 seconds and 120 seconds. Both are settings. A first attempt that fails can therefore delay the alert by about two and a half minutes at worst.
- **What is retried.** Timeouts and temporary Twilio errors are retried. A permanent rejection is not. A notification that is rejected or runs out of attempts is marked failed, and the API shows it.
- **Stopping.** When an event is acknowledged or resolved, unsent notifications for it are stopped. A send already in progress may still complete.
- **Heartbeat.** The worker records each time it runs. `GET /health/ready` reports a problem if that record is stale, because a stopped worker means no deadline fires and no SMS goes out.

> **Delivery is at-least-once.** If the worker crashes after Twilio accepts a message but before we record it, the message is sent again. A caregiver may occasionally get a duplicate. We accept this, because a missing alert is worse than a repeated one.

Automated tests use a fake notification provider. Real SMS go only to the test caregiver.

### Twilio SMS

- **The message.** A short, generic SMS with a link to the event page. It does not name the person, because it shows on a lock screen.
- **The link does not acknowledge.** Opening it only shows the event. The caregiver must press acknowledge (Section 4).
- **Accepted is not delivered.** Twilio accepting a message does not mean the phone received it, and received does not mean read. Our status names keep these apart.
- **Delivery updates.** Twilio calls our webhook as a message's status changes. We save Twilio's message ID with the attempt and match each update to it.
- **Signature check.** We verify Twilio's signature on every webhook call, using the Twilio SDK and the public URL Twilio called. Without it, anyone could post fake delivery updates.
- **Repeated and out-of-order updates.** Twilio may send the same update twice or in the wrong order. An older update never overwrites a newer one.
- **Unknown message.** An update for a message we have no record of is logged and ignored.
- **Local demo.** We use a tunnel (ngrok or Cloudflare Tunnel) to the local stack, so the caregiver's phone can open the link and Twilio can reach the webhook. Automated tests use a fake provider and saved example callbacks.
- **Trial accounts.** A Twilio trial account adds a prefix to each message and can only text verified numbers.

Reference: [Twilio delivery callbacks](https://www.twilio.com/docs/messaging/guides/track-outbound-message-status) and [webhook signature validation](https://www.twilio.com/docs/usage/webhooks/webhooks-security).

### Escalation (milestone 6)

When an alert goes unanswered, the backend steps through a sequence:

1. Send the SMS when the event becomes `ALERTING`.
2. If still unacknowledged at a saved deadline, call the primary caregiver.
3. If still unacknowledged after the call's response window, notify the backup contact.
4. When the sequence runs out, stop. The event stays active and the dashboard shows that escalation is exhausted.

Rules:

- **The backend owns the schedule.** Each step and its deadline are saved, with one piece of work per event and step, so a restart resumes the sequence without repeating finished steps.
- **The event stays `ALERTING` throughout.** Steps are recorded separately in the history.
- **Check before each step.** The backend rechecks the event's state before starting a step. An acknowledgement stops the steps not yet started. An SMS or call already in progress may still complete.
- **A call is not an acknowledgement.** An answered call proves nothing. The call may offer "Press 1 to acknowledge" using [Twilio Gather](https://www.twilio.com/docs/voice/twiml/gather). A keypress counts only when the signature is valid and it matches the expected call, event and contact. It then follows the normal acknowledge rule.
- **A keypress proves someone used that phone**, not who they are.
- **Busy, no answer, failed, or no callback at all** each end the step after a fixed wait, so the sequence cannot stall.

## 9. Cloud deployment and security

The MVP tickets assume a local, trusted demo. Putting the backend on the internet is extra work (milestone 5), and it has to meet this section first.

### Who may do what

| Caller | Proves who it is with | May do |
| --- | --- | --- |
| Device | Its own credential, which can be revoked | Send its own detections and heartbeats. Never another device's. |
| Confirmation screen | A login limited to its device and home | Cancel. Nothing else. |
| Caregiver | A login session | Read events, acknowledge and resolve. |
| Twilio | A signature on each webhook call | Post delivery updates. This endpoint is public, so the signature is its only protection. |

### Requirements

- HTTPS for all device, screen and API traffic.
- The API accepts browser requests only from the frontend's address.
- If login uses cookies, requests are protected against CSRF (a trick where another website makes the browser act for a logged-in user).
- Limits on request size and request rate.
- A cap on SMS per event and per day, so a bug or a noisy sensor cannot send hundreds of messages.
- Only the backend can reach the database.
- Secrets, including the Twilio auth token, come from deployment settings. They never go in source control or logs.
- Services restart automatically if they crash.
- An alert fires when the worker's heartbeat goes stale.
- Logs are structured and carry event and request IDs. They record the worker's last successful run, the age of the oldest waiting work, notification failures, and when each device was last seen.
- We do not store raw radar data or personal information we do not need.

### Chores before going live

- Run migrations as a controlled step, not automatically on start.
- Set up database backups and test a restore once.
- Write down the data-retention policy, covering phone numbers.

### Device offline status

A device is offline when it misses heartbeats, not when it stops sending falls. No falls is the normal case. The heartbeat interval and the offline threshold are settings.

## 10. Repository layout

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
  tests/                 # API, lifecycle and worker tests
  scripts/
    simulate_detection.py
  alembic.ini
  pyproject.toml
  Dockerfile
compose.yaml
.env.example
```

Rules:

- **The lifecycle rules live in one place.** HTTP handlers stay thin, and both the API and the worker call the same functions in `services/`. The cancel-versus-expiry race is hard to get right, and two copies of the logic would drift apart.
- **The clock is passed in.** Code asks a clock object for the time instead of reading the system clock, so a test can set the time to exactly the deadline without waiting.
- **The notification provider is passed in** the same way, so tests use a fake and never send a real SMS.

## 11. Implementation milestones

### First build

| Milestone | What works at the end | Tickets |
| --- | --- | --- |
| 1. Contract and foundation | The lifecycle and API examples are agreed. One command starts the API, PostgreSQL and migrations, the health checks respond, and CI runs. | MVP-01, MVP-02; helps MVP-03 |
| 2. Simulated detection | The simulator sends a detection and exactly one `CONFIRMING` event is saved with its deadline. Duplicates and repeat fires create nothing new. | MVP-04; helps MVP-05 |
| 3. Confirmation and notification | A fall can be cancelled, or can expire and send an SMS. This includes the worker and its heartbeat, the outbox, fake and Twilio SMS, retries, the Twilio webhook, the tunnel setup, and the event list. | MVP-06, MVP-07; helps MVP-08, MVP-09 |
| 4. Caregiver actions and recovery | The caregiver can acknowledge and resolve, and can see the history. Pending work survives a restart. Setup and demo instructions are written. | MVP-10, MVP-12; helps MVP-11, MVP-13 |

### Later

| Milestone | What works at the end | Tickets |
| --- | --- | --- |
| 5. Cloud readiness | The backend runs on the internet over HTTPS, with logins, permissions, device credentials, heartbeats, backups and operational checks. | New tickets needed |
| 6. Voice and backup contact | An unanswered alert leads to a call and then to a backup contact. Keypad acknowledgement works, the sequence survives a restart, and it stops when exhausted. | New tickets needed |
| 7. Edge integration | Detections come from the live or replayed radar. The device buffers unsent detections and warns locally. Delayed detections and offline cancels are handled. | Later radar integration work |

Milestones 5–7 and the Twilio callbacks in MVP-07 are not in the current estimates.

## 12. Validation and acceptance

Tests that involve database constraints or races run against a real PostgreSQL database. Lifecycle rules also have small unit tests. Boundary tests set the clock (Section 10) instead of sleeping.

### Detections

- A valid detection is saved.
- The same detection sent twice produces one event.
- The same detection sent twice at the same instant produces one event.
- A reused ID with different data fails.
- An invalid request fails with a clear error.
- A detection that is too old fails.
- A detection dated too far in the future fails.
- A detection from an unknown or disabled sensor fails.
- A new detection while the sensor has an active event returns that event and creates nothing.

### Confirmation

- Cancel before the deadline succeeds and sends no SMS.
- Cancel exactly at the deadline fails.
- Cancel after the deadline fails.
- Cancel and expiry at the same instant give one outcome and one history.

### Notification

- Expiry creates exactly one notification.
- A temporary failure is retried.
- A permanent failure is not retried.
- Running out of attempts is visible through the API.
- A webhook call with a bad signature is rejected.
- Repeated and out-of-order delivery updates leave the newest status in place.
- A delivered SMS does not acknowledge the event.

### Caregiver actions

- Acknowledge and resolve work only from the allowed states.
- Every state and action pair returns what the table in Section 4 says.
- A repeated action adds no history.

### Restart and recovery

- A restart during a pending confirmation recovers it.
- A restart after a deadline has passed still alerts.
- A restart while a notification is being claimed or sent recovers the work. A duplicate SMS is possible and is the documented limit.
- History, times, pages and current state are the same after a refresh or restart.
- `GET /health/ready` reports a problem when the worker has stopped.

### Cloud readiness (milestone 5)

- A request with no login fails.
- A request without permission fails.
- A device cannot act as another device.
- Missed heartbeats mark a device offline.

### Voice (milestone 6)

- A call that is busy or not answered moves to the next step.
- A keypress acknowledges the event.
- An acknowledgement that races an escalation step gives one outcome.
- The sequence stops when its steps run out.
- A restart with a pending escalation deadline resumes the sequence.

### Measurements and CI

We record three delays separately: how long the API takes to answer, how long after a deadline the worker acts, and how long until an SMS attempt starts.

CI runs formatting, lint and the automated tests against a throwaway PostgreSQL database.

## 13. Decisions made

| Question | Decision |
| --- | --- |
| What does this plan cover? | All seven milestones. Milestones 1–4 are the first build. |
| Language, database and SMS provider? | Python with FastAPI, PostgreSQL, Twilio. |
| Message broker? | None. The worker polls PostgreSQL. |
| How is the cancel-versus-expiry race settled? | A conditional update, judged by the database's clock (Section 4). |
| What if the sensor fires again while an event is active? | Ignore it and return the active event. One fall, one alert, one SMS. |
| What ID do clients use for a device? | `sensor_id`, everywhere. |
| How is "home" stored? | It is not. There is one home. |
| Where is the test caregiver's phone number? | In `.env`. |
| What is the actor before login exists? | `system`, `device:<sensor_id>` or `caregiver:test`. |
| Does a delivered SMS count as acknowledgement? | Never. |
| How do the caregiver's phone and Twilio reach the local demo? | Through a tunnel (ngrok or Cloudflare Tunnel). |
| Does the worker have a heartbeat? | Yes, from the first build. The alert on a stale heartbeat comes with milestone 5. |
| Do we promise exactly one SMS? | No. Delivery is at-least-once. |
| Does the system call emergency services? | No. |
