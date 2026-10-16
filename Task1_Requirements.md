# Task 1 – Requirement Extraction (Baseline v1.0)

I extracted the requirements below from the SACDCS scenario. Each one names a specific action and a way to check it. Verification methods use the standard set: **T** = Test, **D** = Demonstration, **I** = Inspection, **A** = Analysis, **S** = Simulation.

Priority uses MoSCoW: **M** = Must, **S** = Should, **C** = Could.

> **Note on the baseline:** this is the version Mission Control approved, *before* the change requests. Three entries (FR-03, FR-06, NFR-02) keep their original wording on purpose, because Task 2 changes them. In review I had already flagged these three as weak (vague or not measurable), and the CRs fix exactly that.

## 1.1 Functional Requirements (10)

| Req. ID | Requirement | Type | Priority | Verification Method |
|---|---|---|---|---|
| FR-01 | The system shall dispatch a delivery request (robot ID, clinic ID, batch ID) only to a robot in IDLE whose battery is above 20 % and whose chamber is in active conservation. The robot then moves to NAVIGATING. Otherwise the request is rejected with a reason code. | FR | M | T (TC-01, TC-04) |
| FR-02 | Mission Control shall validate every command before sending it to a robot. Validation checks the message format, that the robot ID exists, and that the command is allowed in the robot's current state. An invalid command shall be rejected with a reason code and shall not reach the robot. | FR | M | T (TC-04) |
| FR-03 | Only authenticated operators shall be permitted to issue robot commands. *(baseline wording, see Task 2, CR-03)* | FR | M | T (TC-02) |
| FR-04 | When the robot detects an obstacle on its planned path while moving, it shall suspend navigation, go to AVOIDING_OBSTACLE, and resume its previous movement once the path is reported clear. The robot shall not start delivering while it is handling an obstacle. | FR | M | T, S (TC-05) |
| FR-05 | If battery charge drops to 20 % (critical) during navigation, the robot shall abandon the delivery, keep the batch in the chamber, return to the warehouse, and notify the operator with the abandoned batch ID. | FR | M | T (TC-06) |
| FR-06 | The robot shall enter Safe Mode when a critical battery or thermal condition is detected. *(baseline wording, see Task 2, CR-01)* | FR | M | T (TC-07) |
| FR-07 | When chamber temperature or humidity leaves the loaded batch profile range, the chamber shall start a correction action. It shall mark the correction successful only after sensor readings stay inside the range for 60 s in a row. | FR | M | T (TC-08) |
| FR-08 | If a correction does not return the value to range within the 10-minute recovery period, the chamber shall enter protection mode, reduce light exposure to the batch, mark the batch "at risk" and alert the operator. | FR | M | T (TC-09) |
| FR-09 | Each robot shall report status telemetry to Mission Control every 5 s: position, robot state, battery %, chamber temperature, humidity, vibration level, door status and power source. | FR | M | T, D (TC-01, TC-12) |
| FR-10 | The system shall record every command, authentication attempt, state change, alarm and operator acknowledgement in an append-only event log. Each entry holds a UTC timestamp, robot ID, operator ID (if any), event type and outcome. | FR | M | T, I (TC-01 to TC-14) |

## 1.2 Non-Functional Requirements (5)

| Req. ID | Requirement | Type | Priority | Verification Method |
|---|---|---|---|---|
| NFR-01 (Performance) | With a one-way link delay of up to 5 s, telemetry shall appear on the Mission Control dashboard within 10 s of being generated on the robot. Every command shall receive an acknowledgement (accepted or rejected) within 15 s for 95 % of commands. | NFR | M | T (TC-01, TC-12) |
| NFR-02 (Scalability) | The system shall support communication with multiple robots simultaneously. *(baseline wording, see Task 2, CR-02)* | NFR | M | T (TC-12) |
| NFR-03 (Security) | All operator–robot traffic shall be encrypted (TLS 1.2 or higher). An operator account shall be locked for 15 min after 3 failed logins within 5 min. Idle sessions shall expire after 10 min. | NFR | M | T, I (TC-02) |
| NFR-04 (Reliability) | During a link outage the robot shall buffer at least 24 h of events and telemetry locally, and upload them after reconnection with **zero** lost or duplicated log entries. Mission Control availability during the pilot shall be at least 99.5 % per month. | NFR | M | T, A (TC-11) |
| NFR-05 (Usability) | A trained operator shall be able to find a robot in alarm and send a Safe Mode or recall command within 30 s. Pass criterion: at least 9 of 10 trial runs across 5 operators. Pages of the static project portal shall load in under 3 s on a 2 Mbps link. | NFR | S | D (usability trial) |

## 1.3 High-Risk Requirements (4)

| Req. ID | Requirement | Type | Priority | Verification Method | Why high-risk |
|---|---|---|---|---|---|
| HR-01 | The robot shall enter a railway level crossing only when ARLCCS reports **Barrier_Open = TRUE** and **Train_Approaching = FALSE**, and that status is less than 2 s old. Otherwise it shall wait at the stop line. | Safety | M | T, S (TC-10) | Railway crossing: a robot–train collision |
| HR-02 | If no heartbeat is received for 30 s, the robot shall continue on its own to the nearest safe stopping point, which must never be inside a crossing, and enter LINK_LOST. No safety action may depend on a remote command arriving. | Safety | M | T, S (TC-11) | Depends on the link: commands can arrive late or not at all |
| HR-03 | A batch that has been in protection mode or marked "at risk" shall not be handed over at a clinic. The robot shall refuse the DELIVERING handover for that batch until a Supervisor releases it. | Safety | M | T (TC-14) | Patient safety: a damaged vaccine is given to a patient |
| HR-04 | When the robot detects an obstacle inside its stopping distance, it shall command a stop within 500 ms. | Safety | M | T, S (TC-05) | Real-time: a collision with a person or vehicle |

## 1.4 Summary

- 10 FRs. Each names a concrete action: dispatch, command validation, authentication, obstacle handling, battery abort, Safe Mode entry, chamber correction, protection, telemetry, logging.
- 5 NFRs, one per quality attribute. Four of them are already measurable. NFR-02 is fixed by CR-02.
- 4 high-risk requirements covering a railway crossing, link loss, chamber safety and real-time obstacle handling.
