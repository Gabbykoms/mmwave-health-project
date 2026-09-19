# Low-Cost Privacy-Preserving In-Home Fall Detection and Emergency Escalation Using mmWave Radar

**Senior Capstone / Thesis Project**

## 1. Project Overview

This project will design and implement a privacy-preserving in-home fall/collapse detection and emergency escalation system using millimeter-wave (mmWave) radar.

The objective is not to develop a new medical diagnostic system or collect a new clinical dataset. Instead, the project focuses on the engineering problem of turning radar-based fall detection into a complete and reliable home-monitoring system.

The finished prototype will integrate:

**mmWave radar → radar processing → ML-based fall detection → decision logic → on-site confirmation → caregiver notification → acknowledgement/escalation**

A low-cost mmWave radar will monitor an instrumented room without recording identifiable video. A local edge computer will process radar observations and run the fall-detection pipeline. When a possible fall is detected, a decision engine will determine whether the event warrants confirmation. The system will first ask the occupant whether they are okay. If the alert is not cancelled, the system will notify a caregiver and track the event until it is acknowledged or resolved.

Machine-learning development and quantitative detector evaluation will use existing publicly available radar datasets. **No new human-subject fall dataset will be collected as part of this project.**

The primary contribution of the capstone is therefore the **design, implementation, integration, and evaluation of the complete system**, rather than the creation of a new dataset or novel ML architecture.

The architecture will also be designed so that the system can later support broader **preventive and longitudinal home monitoring**. These capabilities are treated as stretch goals and future extensions rather than requirements for successful completion of the core project.

---

## 2. Motivation

Falls and unexpected collapses at home are particularly dangerous when a person is alone and unable to summon assistance.

Many fall-detection projects focus almost entirely on the classification problem:

> Did a fall occur?

A useful home safety system must solve a larger problem:

> If something resembling a fall occurs, can the system determine that the event warrants attention, confirm whether assistance is required, and reliably notify another person?

The project therefore treats fall detection as only one component of a larger safety system.

The system must:

1. continuously receive sensor observations;
2. identify possible falls;
3. avoid escalating every transient classifier output;
4. allow the occupant to cancel a false alarm;
5. notify a caregiver if the occupant does not respond;
6. retry or escalate if an alert is not acknowledged;
7. record the state and outcome of the event; and
8. recover gracefully from common software and network failures.

Beyond immediate emergencies, ambient sensing may also support a second timescale of monitoring. Changes in mobility, inactivity, room usage, sleep/wake routines, and other daily activity patterns may provide useful information to caregivers before an acute emergency occurs.

The long-term vision is therefore a system capable of supporting three timescales:

**Immediate safety:** seconds to minutes, such as detecting a possible fall.

**Situational safety:** minutes to hours, such as unusually prolonged inactivity.

**Preventive monitoring:** days to weeks, such as substantial changes from an individual's established activity baseline.

The core capstone focuses on the first category. The other two are optional extensions.

---

## 3. Scope

### 3.1 In Scope

The core project includes:

* low-cost mmWave radar sensing;
* radar data ingestion;
* preprocessing of radar observations;
* fall/activity classification using an existing or reproduced ML approach;
* training and evaluation using public datasets;
* replay of recorded radar datasets through the system;
* live radar integration where compatible with the selected hardware;
* temporal filtering and event decision logic;
* on-site confirmation interface;
* caregiver notifications;
* acknowledgement and escalation logic;
* caregiver web dashboard;
* persistent event logging;
* edge deployment;
* system monitoring;
* automated integration testing;
* fault-injection testing;
* latency and reliability evaluation;
* privacy and security analysis.

### 3.2 Stretch Scope

If the core system is completed ahead of schedule, additional features may include:

* prolonged inactivity detection;
* long-lie detection;
* passive environmental sensor integration;
* whole-home activity estimation;
* personalized activity baselines;
* longitudinal behavioral anomaly detection;
* wellness trend visualization;
* non-emergency caregiver notifications;
* basic sleep/rest pattern estimation;
* experimental stationary respiration estimation where supported by the radar.

Stretch features are explicitly **not required for successful completion of the capstone**.

### 3.3 Explicitly Out of Scope

The project will not attempt to:

* collect a new human-subject fall dataset;
* recruit elderly, frail, or clinical participants;
* reproduce real syncope experimentally;
* diagnose syncope or determine the medical cause of a fall;
* diagnose disease from behavioral changes;
* claim clinical effectiveness;
* develop a medical device;
* call emergency services automatically;
* prove performance on real medical emergencies;
* develop a novel radar signal-processing algorithm solely for publication;
* implement a novel neural-network architecture unless required by available data;
* perform raw-ADC radar research unless necessary for system integration;
* support arbitrary multi-person environments.

The system is a **research and engineering prototype**, not a medical device and not a substitute for professional emergency or medical monitoring.

---

## 4. Project Objectives

The project has five primary objectives.

### Objective 1 — Implement Radar-Based Fall Detection

Develop a software pipeline capable of accepting radar observations and producing fall/activity predictions.

Public mmWave radar datasets will be used for model development and evaluation.

Depending on the selected dataset, inputs may include:

* point clouds;
* range-Doppler representations;
* range-angle representations;
* micro-Doppler spectrograms; or
* derived radar features.

The project will prioritize a representation that can reasonably be reproduced by affordable radar hardware.

### Objective 2 — Build a Robust Decision Engine

The raw ML classifier will not directly trigger caregiver notifications.

Instead, predictions will enter a temporal decision engine that considers factors such as:

* confidence over multiple frames;
* transition from upright to low posture;
* duration of the detected condition;
* recent motion;
* repeated detections;
* cancellation state;
* cooldown/debounce intervals; and
* sensor availability.

The objective is to transform noisy frame-level predictions into meaningful system events.

### Objective 3 — Implement Confirmation and Escalation

A suspected fall will enter a state machine such as:

`MONITORING → DETECTED → CONFIRMING → ALERTING → ACKNOWLEDGED → RESOLVED`

with alternative paths such as:

`CONFIRMING → CANCELLED`

and:

`ALERTING → ESCALATED`

The occupant will be given a configurable period to indicate that assistance is unnecessary.

If no response is received, the system will notify a configured caregiver.

### Objective 4 — Build the Caregiver Interface

A lightweight web application will allow a caregiver to:

* view active alerts;
* see the room and detection time;
* view relevant event context;
* acknowledge an alert;
* resolve an event;
* inspect recent event history; and
* view sensor/system status.

A web application is preferred over a native mobile application to keep the project focused on the core system.

### Objective 5 — Evaluate the Complete System

Evaluation will distinguish between two different questions:

**Detector evaluation:**
How well does the selected ML approach perform on existing labeled radar datasets?

**System evaluation:**
How reliably and quickly does the complete prototype react to detector events, failures, cancellations, acknowledgements, and network conditions?

These questions will be measured separately so that system reliability is not confused with clinical fall-detection effectiveness.

---

## 5. System Architecture

The proposed architecture is:

```text
                   ┌──────────────────┐
                   │   mmWave Radar   │
                   └────────┬─────────┘
                            │
                   UART / USB / Serial
                            │
                            ▼
                   ┌──────────────────┐
                   │   Edge Device    │
                   │                  │
                   │ Radar ingestion  │
                   │ Preprocessing    │
                   │ ML inference     │
                   │ Decision engine  │
                   └────────┬─────────┘
                            │
                     possible fall
                            │
                            ▼
                   ┌──────────────────┐
                   │   Confirmation   │
                   │                  │
                   │  "Are you OK?"   │
                   │     countdown    │
                   └───────┬──────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
          User cancels             No response
              │                         │
              ▼                         ▼
          CANCELLED                  ALERTING
                                        │
                              ┌─────────┴─────────┐
                              │                   │
                              ▼                   ▼
                             SMS             Web/Push
                              │                   │
                              └─────────┬─────────┘
                                        ▼
                               Caregiver Interface
                                        │
                              acknowledge / resolve
```

The edge device may be a Raspberry Pi-class computer, mini PC, laptop, or similar system capable of running the selected inference model.

The ML model will normally run on this edge computer rather than directly on the radar sensor.

The architecture will use standardized sensor and event interfaces so additional monitoring components can later be added without redesigning the emergency pipeline.

---

## 6. Two Sensor Input Modes

A major architectural feature of the project will be a common detector interface supporting both recorded and live radar data.

### 6.1 Dataset Replay Mode

The first mode uses prerecorded public radar data:

```text
Public Dataset
      ↓
Dataset Replay Service
      ↓
Preprocessing
      ↓
ML Detector
      ↓
Decision Engine
      ↓
Confirmation / Escalation
```

The replay service will reproduce the timing and ordering of recorded radar frames so that the rest of the application behaves as though it were receiving live measurements.

This mode provides a deterministic and repeatable end-to-end test environment.

### 6.2 Live Radar Mode

The second mode uses the physical radar:

```text
mmWave Radar
      ↓
Radar Adapter
      ↓
Preprocessing
      ↓
ML Detector / Sensor Detector
      ↓
Decision Engine
      ↓
Confirmation / Escalation
```

The same downstream interface will be used by both modes.

This separation prevents the entire capstone from depending on perfect transfer between a public dataset and the selected physical radar.

If the public dataset and physical sensor are sufficiently compatible, the trained model may be deployed directly against live observations.

If they are not compatible, dataset replay will remain the basis for quantitative ML evaluation, while the physical radar will demonstrate live sensing and system integration.

---

## 7. Machine-Learning Component

### 7.1 Data Source

No original human-subject dataset will be collected.

Instead, one or more existing public mmWave radar datasets containing falls and ordinary activities will be selected.

Selection criteria include:

* availability for academic use;
* fall labels;
* availability of non-fall activities;
* multiple participants where possible;
* documented radar configuration;
* accessible preprocessing code;
* compatibility with affordable radar hardware;
* point-cloud or other manageable representations; and
* sufficient documentation to reproduce a baseline.

One dataset will be selected as the **primary dataset** rather than indiscriminately combining incompatible radar datasets.

### 7.2 ML Approach

Novel ML architecture development is not a requirement.

Depending on the selected dataset, candidate approaches include:

* engineered features + gradient-boosted trees;
* 1D/2D CNN;
* CNN + LSTM/GRU;
* temporal convolutional network;
* point-cloud network; or
* reproduction of a published baseline supplied with the dataset.

The simplest approach capable of providing reliable and reproducible results will be preferred.

### 7.3 Training and Deployment

The offline pipeline will resemble:

```text
Public radar dataset
        ↓
Data loader
        ↓
Preprocessing
        ↓
Train / validation / test split
        ↓
Model training
        ↓
Evaluation
        ↓
Export model
        ↓
ONNX / PyTorch model artifact
```

The exported model will then be loaded by the edge inference service.

Training occurs offline. The deployed system performs inference rather than continuously retraining.

### 7.4 Evaluation Metrics

Depending on dataset structure, detector evaluation will report:

* precision;
* recall;
* F1 score;
* confusion matrix;
* false-positive rate;
* false-negative rate;
* precision-recall curve where useful; and
* inference latency.

Subject-independent splits will be used when subject identities and sufficient participants are available.

Results will be reported as performance on the selected public dataset, not as evidence of clinical effectiveness.

---

## 8. Decision Engine

The classifier answers:

> How likely is this observation to represent a fall?

The decision engine answers:

> Should the system begin an emergency confirmation workflow?

These are intentionally separate.

A possible rule might require:

1. fall probability above a threshold;
2. persistence across multiple observations;
3. evidence of a posture transition;
4. continued low posture or limited movement; and
5. no active cooldown or recently cancelled event.

The exact rules will depend on the information available from the selected dataset and sensor.

The state machine will also prevent problems such as duplicate notifications, repeated alerts from one event, single-frame anomalies, alerts continuing after cancellation, and stale detections generating new events.

---

## 9. Confirmation and Escalation

When a possible fall is accepted by the decision engine, the system enters `CONFIRMING`.

The local interface displays or announces:

**“A possible fall was detected. Are you okay?”**

A countdown begins.

The occupant can cancel the event using an available local interface such as a physical button, touchscreen/web button, or optional voice interface.

If the user responds:

`CONFIRMING → CANCELLED`

If the countdown expires without a response:

`CONFIRMING → ALERTING`

The system then sends an alert to the configured caregiver.

If acknowledgement is not received within a configurable period, the system retries and/or escalates to another configured contact.

The capstone will stop at caregiver/contact escalation and will not automatically contact emergency services.

---

## 10. Caregiver Application

The caregiver interface will be implemented as a responsive web application.

The core dashboard will provide:

* monitoring status;
* radar/sensor connection status;
* active alerts;
* room and event timestamp;
* acknowledgement controls;
* resolution controls; and
* event history.

If preventive-monitoring stretch goals are implemented, the dashboard can additionally include a separate **Wellness Trends** section.

Emergency alerts and preventive observations will remain visually and logically separate so that a gradual behavioral change is not presented with the urgency of a possible fall.

---

## 11. Backend and Reliability

The backend will maintain persistent event state.

A simplified event record may contain:

```text
event_id
sensor_id
timestamp
event_type
detector_confidence
state
confirmation_deadline
notification_attempts
acknowledged_by
acknowledged_at
resolved_at
```

The system will be designed so that restarting a service does not silently erase an active emergency state.

Reliability mechanisms may include:

* persistent event storage;
* notification retry;
* acknowledgement tracking;
* process watchdogs;
* service auto-restart;
* health checks;
* sensor-disconnect detection;
* structured logs;
* duplicate-event protection; and
* graceful degradation during network failure.

---

## 12. Testing Strategy

Because no original fall dataset will be collected, automated testing and event replay become major parts of the project.

### 12.1 Unit Testing

Individual components will be tested independently.

### 12.2 Dataset Replay Testing

Recorded radar sequences will be replayed through the complete system.

Known labeled sequences allow deterministic testing from recorded sensor input through caregiver acknowledgement.

### 12.3 Synthetic Detector Events

A simulated detector will also be retained for system testing.

This allows backend, state-machine, notification, and UI testing independently of the ML subsystem.

### 12.4 Fault Injection

Evaluation will intentionally introduce failures such as:

* internet disconnection;
* notification failure;
* backend restart;
* edge-service restart;
* sensor disconnection;
* duplicate detector events;
* delayed acknowledgement; and
* database/service interruption where feasible.

---

## 13. Evaluation

The thesis will explicitly separate **ML evaluation** from **systems evaluation**.

### 13.1 Detector Evaluation

Using public labeled data:

* classification performance;
* false positives and false negatives;
* confusion matrix;
* inference time;
* model size; and
* computational requirements.

### 13.2 End-to-End Latency

Measure the complete path from detector output through decision processing, confirmation, notification, and caregiver acknowledgement.

### 13.3 Reliability

Repeated automated scenarios will determine whether the system reaches the expected final state.

### 13.4 Edge Performance

The deployed system will measure:

* CPU utilization;
* memory utilization;
* inference rate;
* model size;
* process stability; and
* sustained operation.

A multi-hour or multi-day soak test may evaluate system stability without requiring simulated human falls.

---

## 14. Privacy and Security

Privacy is a major reason for selecting mmWave radar instead of conventional RGB cameras.

The deployed system will not require continuous identifiable video.

Security considerations include:

* authentication for the caregiver application;
* authorization for event-management operations;
* TLS for network communication;
* secure credential storage;
* minimal collection of personal information;
* configurable event-data retention;
* audit/event logging; and
* protection against unauthorized access to occupancy information.

The thesis will include a basic threat model because occupancy and behavioral information can itself be privacy-sensitive even when no camera is present.

---

## 15. Hardware Strategy

The project will prioritize **low-cost commercially available mmWave hardware** rather than expensive radar development platforms unless low-level access becomes necessary.

The hardware must provide sufficient information for human presence/activity sensing, radar point-cloud or equivalent output, communication with the edge computer, and continuous operation.

A Raspberry Pi-class computer, mini PC, or existing laptop can serve as the initial edge device.

The project will avoid purchasing expensive raw-ADC capture hardware unless a clear technical requirement emerges.

---

## 16. Software Architecture

The implementation will be modular.

```text
sensor/
    live_radar_adapter
    dataset_replay_adapter

ml/
    preprocessing
    model
    inference_service

decision/
    temporal_filter
    event_state_machine

monitoring/
    activity_aggregator
    baseline_model
    anomaly_detector

backend/
    event_api
    notification_service
    persistence
    authentication

frontend/
    caregiver_dashboard
    confirmation_ui

testing/
    detector_simulator
    dataset_replay
    fault_injection
    integration_tests
```

The `monitoring/` components are optional and are required only if the preventive-monitoring stretch goals are pursued.

---

## 17. Project Deliverables

### Core Software Deliverables

1. radar/dataset ingestion service;
2. reproducible ML training pipeline;
3. exported fall-detection model;
4. real-time inference service;
5. dataset replay engine;
6. detector simulator;
7. temporal decision engine;
8. confirmation interface;
9. notification/escalation service;
10. caregiver web dashboard;
11. persistent event datastore;
12. automated test suite;
13. fault-injection tests; and
14. deployment/configuration documentation.

### Core Demonstration

The final demonstration will show:

**Scenario A — Normal activity**

`Radar/replay → detector → normal → no action`

**Scenario B — Possible fall, occupant responds**

`Fall detected → confirmation → "I'm OK" → cancelled`

**Scenario C — Possible fall, no response**

`Fall detected → confirmation timeout → caregiver notification → acknowledgement → resolved`

At least one failure scenario will also be demonstrated.

---

# 18. Stretch Goals: Preventive and Longitudinal Monitoring

The core project addresses acute events such as falls. If time permits, the same sensing and backend infrastructure will be extended to investigate whether ambient sensing can also provide useful **long-term behavioral information**.

These features are exploratory engineering extensions and are not required for successful completion of the thesis.

The goal is not to diagnose medical conditions. Instead, the system would identify meaningful deviations from an individual's established behavioral baseline and present those changes to a caregiver.

## 18.1 Stretch Goal 1 — Prolonged Inactivity and Long-Lie Detection

The first extension is the most closely related to the core fall-detection system.

Rather than only detecting the transition associated with a fall, the system can monitor whether a person remains inactive or near floor level for an unusually long period.

For example:

`person detected → low posture → little/no movement → duration threshold exceeded → wellness/safety event`

This provides a secondary safety mechanism if the original fall transition was missed.

This feature may use radar occupancy, posture, centroid height, movement, or other available measurements.

---

## 18.2 Stretch Goal 2 — Passive Whole-Home Sensors

Low-cost passive sensors may supplement the radar.

Potential devices include:

* PIR motion sensors;
* door/contact sensors;
* bed/chair pressure sensors; and
* selected smart plugs or other simple activity indicators.

These sensors do not need to determine exactly what the occupant is doing.

Instead, they provide coarse environmental events such as:

```text
07:34 bedroom motion
07:42 kitchen motion
08:05 refrigerator/door activity
09:17 living-room motion
10:03 front-door activity
```

Combining these observations can provide a broader view of daily activity without requiring cameras throughout the home.

---

## 18.3 Stretch Goal 3 — Personal Activity Baseline

The system may maintain a rolling baseline describing normal activity patterns.

Possible aggregated features include:

* total daily activity;
* duration of inactive periods;
* approximate wake/start-of-day time;
* room-transition frequency;
* nighttime activity;
* time spent in selected rooms;
* number of unusually long sedentary periods; and
* general routine consistency.

The objective is to compare the person primarily against **their own previous behavior**, rather than against a universal definition of normal behavior.

For example:

```text
PERSONAL BASELINE

Daily active time             4.8 hr
Longest daytime inactivity    52 min
Typical start of activity     7:20 AM
Nighttime activity events     1.4/night
Room transitions              31/day
```

A later period might show:

```text
CURRENT PERIOD

Daily active time             3.1 hr    ↓
Longest daytime inactivity    108 min   ↑
Typical start of activity     9:05 AM   changed
Nighttime activity events     3.7/night ↑
Room transitions              19/day    ↓
```

These changes would not be interpreted as a diagnosis.

Instead, the system could report:

> Activity patterns have changed substantially relative to the established personal baseline.

---

## 18.4 Stretch Goal 4 — Behavioral Anomaly Detection

A lightweight anomaly-detection model may analyze longitudinal activity summaries.

Candidate techniques include:

* rolling mean/variance thresholds;
* exponentially weighted baselines;
* change-point detection;
* Isolation Forest;
* one-class classification; and
* autoencoder reconstruction error.

Unlike the fall detector, this subsystem does not necessarily require medically labeled examples.

Its task is:

> Is recent behavior substantially different from this person's previous behavioral pattern?

The system would generate a low-priority wellness observation rather than an emergency alert.

---

## 18.5 Stretch Goal 5 — Wellness Trends Dashboard

If longitudinal monitoring is implemented, the caregiver dashboard will include a separate wellness section.

For example:

```text
WELLNESS TRENDS
────────────────────────────

Overall activity
↓ 18% from personal baseline

Longest inactive period
↑ 37 minutes

Night activity
Within expected range

Routine consistency
Moderate change

Last 7 days
[ activity trend visualization ]
```

The interface will distinguish between:

**Emergency events**

> Possible fall detected — immediate attention requested.

and:

**Wellness observations**

> Activity has decreased relative to the recent personal baseline.

This distinction is important because behavioral anomalies should not be presented as medical diagnoses or emergency conditions.

---

## 18.6 Stretch Goal 6 — Experimental Contactless Respiration Monitoring

If supported by the selected radar, stationary respiration estimation may be explored.

This feature would be explicitly experimental.

Radar-based vital-sign estimation can be sensitive to:

* body movement;
* sensor placement;
* distance;
* orientation;
* multiple people;
* environmental reflections; and
* signal quality.

Therefore, respiration monitoring will not be relied upon for emergency decisions and will not be presented as clinical-grade measurement.

It may instead be evaluated as an exploratory demonstration of additional information that could eventually be derived from the same privacy-preserving sensing platform.

---

## 18.7 Long-Term Platform Architecture

These extensions produce a broader architecture:

```text
                        HOME SENSORS
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
      mmWave radar        PIR/contact       other sensors
          │                  │                  │
          └──────────────────┼──────────────────┘
                             ▼
                     Edge Monitoring
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
       ACUTE EVENT PATH             LONGITUDINAL PATH
              │                             │
       fall detection                activity summaries
              │                             │
       decision engine                personal baseline
              │                             │
       confirmation                   anomaly detection
              │                             │
       emergency alert               wellness observation
              │                             │
              └──────────────┬──────────────┘
                             ▼
                    Caregiver Platform
```

This architecture supports three monitoring timescales:

| Timescale       | Example                   | Response                    |
| --------------- | ------------------------- | --------------------------- |
| Seconds/minutes | Possible fall             | Confirm and alert           |
| Minutes/hours   | Prolonged inactivity      | Safety/wellness check       |
| Days/weeks      | Sustained activity change | Caregiver trend information |

The core thesis is required to implement the first timescale.

The second and third timescales are stretch goals.

---

## 18.8 Boundary of Preventive Monitoring

Preventive monitoring must be described carefully.

The system may identify:

* decreased activity;
* increased inactivity;
* changes in routine;
* unusual nighttime movement;
* prolonged room occupancy; or
* other deviations from an established baseline.

It must **not** conclude from these observations that the occupant has:

* dementia;
* cardiovascular disease;
* depression;
* infection;
* frailty;
* sleep disorder; or
* another medical condition.

Such relationships may motivate future clinical research, but establishing them is outside the scope of this engineering capstone.

---

## 19. Thesis Structure

### Chapter 1 — Introduction

Motivation, problem definition, objectives, contributions, and scope.

### Chapter 2 — Background and Related Work

mmWave radar, fall detection, camera-based approaches, wearables, radar-based activity recognition, ambient assisted living, and emergency monitoring systems.

### Chapter 3 — Requirements and System Architecture

Functional/non-functional requirements, architecture, detector interface, state machine, failure model, and privacy requirements.

### Chapter 4 — Radar and Machine-Learning Pipeline

Public dataset, preprocessing, selected model, training procedure, evaluation methodology, and deployment format.

### Chapter 5 — System Implementation

Sensor integration, edge inference, decision engine, confirmation UI, backend, notification service, caregiver dashboard, and persistence.

If implemented, preventive-monitoring extensions will also be described here.

### Chapter 6 — Evaluation

ML results, latency experiments, integration tests, resource usage, fault-injection experiments, soak testing, and results from any completed stretch features.

### Chapter 7 — Discussion and Limitations

Interpretation of results, cross-sensor limitations, dataset limitations, lack of original human-subject evaluation, preventive-monitoring limitations, privacy considerations, and deployment challenges.

### Chapter 8 — Conclusion and Future Work

Summary and future development toward longitudinal aging-in-place monitoring and eventual real-world validation.

---

## 20. Two-Semester Timeline

### Semester 1 — Core System

**Weeks 1–3**

* dataset selection;
* hardware selection;
* literature review;
* interfaces and architecture;
* repository and CI.

**Weeks 4–7**

* ML baseline;
* preprocessing;
* model evaluation;
* model export;
* dataset replay.

**Weeks 8–11**

* decision state machine;
* backend;
* persistence;
* confirmation workflow;
* caregiver dashboard.

**Weeks 12–14**

Complete first end-to-end system:

`public radar sequence → ML → decision → confirmation → notification → acknowledgement`

### Semester 2 — Deployment and Evaluation

**Weeks 1–4**

* physical radar integration;
* edge deployment;
* live sensor adapter;
* inference optimization.

**Weeks 5–7**

* reliability;
* notification retries;
* health monitoring;
* fault recovery;
* security.

**Weeks 8–10**

* final evaluation;
* latency measurements;
* fault injection;
* resource measurements;
* soak testing.

### Stretch Feature Gate

**Only after the core end-to-end system and required evaluation are operational**, remaining development time may be allocated to:

1. prolonged inactivity detection;
2. passive sensor integration;
3. personal activity baselines;
4. behavioral anomaly detection;
5. wellness dashboard;
6. experimental respiration monitoring.

Incomplete stretch features will not prevent completion of the core thesis.

---

## 21. Risk Management

### Risk: Public dataset is incompatible with physical radar

Use dataset replay for quantitative ML evaluation and treat live radar as a separate integration result unless compatibility is demonstrated.

### Risk: ML performance is mediocre

Novel classifier performance is not the sole contribution. The project evaluates the complete detection-to-assistance architecture.

### Risk: Physical radar integration takes too long

Dataset replay provides an independent end-to-end system path.

### Risk: Preventive-monitoring features cause scope creep

All preventive-monitoring functionality remains behind a formal stretch-feature gate. Core fall detection, escalation, evaluation, and thesis work take priority.

### Risk: Behavioral anomalies are overinterpreted

The system reports deviations from personal baselines rather than diagnosing their cause.

---

## 22. Success Criteria

### Required

The core project is successful if:

1. a fall detector can be trained or reproduced using a public radar dataset;
2. recorded radar sequences can be replayed through the complete system;
3. detector output is converted into persistent events;
4. an occupant can cancel an event during confirmation;
5. an unanswered event generates a caregiver notification;
6. a caregiver can acknowledge and resolve the event;
7. repeated and duplicate events are handled correctly;
8. selected failures are detected or recovered from;
9. the system operates on an edge-class computer;
10. detector and system performance are quantitatively evaluated; and
11. the architecture, implementation, results, and limitations are documented in the thesis.

### Optional Stretch Success

Depending on available time, the project may additionally demonstrate one or more of:

* prolonged inactivity detection;
* passive whole-home sensor fusion;
* personal activity baseline generation;
* longitudinal anomaly detection;
* wellness trend visualization; and
* experimental contactless respiration sensing.

These are explicitly **not required for the project's minimum successful completion**.

---

## 23. Final Project Definition

The core project asks:

> **How can an existing radar-based fall-detection capability be integrated into a low-cost, privacy-preserving home monitoring system that handles detection, confirmation, caregiver notification, acknowledgement, failure, and recovery as one complete pipeline?**

The broader stretch vision asks:

> **Can the same privacy-preserving sensing infrastructure also support longer-term awareness of changes in everyday activity without requiring continuous camera monitoring?**

This produces a natural progression:

**Fall Detector → Home Safety System → Ambient Preventive Monitoring Platform**

The capstone will guarantee the first two stages.

The third stage provides a direction for stretch development and future work without making successful completion dependent on longitudinal human-subject data collection or clinical validation.
