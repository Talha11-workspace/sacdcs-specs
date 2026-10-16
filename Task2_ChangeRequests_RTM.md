# Task 2 – Change Requests & Requirements Traceability Matrix

## 2.1 Version-controlled change log

Baseline **v1.0** (Task 1) was approved, and then Mission Control issued three CRs. The result is baseline **v1.1**. This change log is kept under version control in this repository. The v1.0 wording stays visible in Task 1, and the v1.1 wording is recorded here.

| Change ID | CR | Req. ID | Status | Old text (v1.0) | New text (v1.1) | Reason |
|---|---|---|---|---|---|---|
| CHG-001 | CR-01 | FR-06 | **Modified** | The robot shall enter Safe Mode when a critical battery or thermal condition is detected. | The robot shall enter Safe Mode **within 3 s** when **chamber temperature exceeds the batch critical threshold (12 °C for the vaccine profile)** or **battery temperature exceeds 60 °C**, or when **battery capacity falls below the emergency level of 10 %**. | The original wording gave no time limit and no thresholds. "Critical" was undefined, and it was also confused with the 20 % return threshold in FR-05. |
| CHG-002 | CR-02 | NFR-02 | **Modified** | The system shall support communication with multiple robots simultaneously. | The system shall support **at least 20 simultaneously connected robots**, each sending telemetry every 5 s, while still meeting the NFR-01 latency limits (telemetry ≤ 10 s, command acknowledgement ≤ 15 s for 95 % of commands). | "Multiple" is satisfied by 2 robots, but the pilot serves 20 clinics. Without a number there is no pass/fail for load testing or capacity planning. |
| CHG-003 | CR-03 | FR-03 | **Modified** | Only authenticated operators shall be permitted to issue robot commands. | The system shall require operators to be **authenticated and role-authorised** before accepting any robot command. A command shall be accepted only if the operator's role permits it: **Viewer**: none. **Dispatcher**: DeliveryRequest, Recall, Pause. **Supervisor**: all Dispatcher commands + SafeModeReset, Batch release, Crossing override hold. **Administrator**: user management only, no robot commands. Rejected commands return `AUTH_REQUIRED` or `ROLE_NOT_PERMITTED`. | Authentication alone lets any logged-in user (for example, a read-only viewer) reset Safe Mode. CR-03 adds authorisation per command. |

**Knock-on changes (impact analysis):**

| Affected item | Change |
|---|---|
| Task 3: T-18, T-26 | Timing guard "≤ 3 s" added. Two battery thresholds separated (20 % → RETURNING via T-06, 10 % → SAFE_MODE via T-18). |
| Task 3: T-19, T-20, T-28 | Reset guarded by `role = Supervisor` (CR-03). |
| Task 5: C-10 | New constraint created directly from CR-01. |
| Task 1: NFR-01 | Now measured under 20-robot load (CR-02). |
| TC-03, TC-12 | New test cases (no test covered roles or 20-robot load before). |

## 2.2 Why the new wording is more measurable or testable

| CR | Original problem | Why the new wording is better |
|---|---|---|
| CR-01 | "Critical condition" and "when detected" have no number, so a tester can't write an expected value. "Thermal" doesn't say *which* temperature, chamber or battery. There is no deadline, so a robot reaching Safe Mode after 2 min would technically pass. | Three concrete trigger values (12 °C, 60 °C, 10 %) and a **3 s deadline** give a clear oracle: inject the value, measure `t(SAFE_MODE) − t(trigger)`, pass if ≤ 3 s. Boundary values (11.9/12.1 °C, 10.1/9.9 %) can now be tested. It also becomes monitorable at runtime (C-10). |
| CR-02 | "Multiple" has no lower bound. There is no number to load-test against or to size the server for. | "At least 20" is a pass/fail threshold that matches the pilot size. Combined with the NFR-01 latency limits, it defines a complete load test (TC-12): 20 robots, 5 s telemetry, 1 h, measure latency percentiles and message loss. |
| CR-03 | "Authenticated" only checks *who* the operator is, not *what* they are allowed to do. The only negative test is "not logged in". | A role-permission matrix gives a finite set of combinations to test (4 roles × command types). Each has an expected outcome (accept / `ROLE_NOT_PERMITTED`). Negative tests like "Viewer issues SafeModeReset" become possible. |

## 2.3 Test cases

Every test follows the scenario chain: **(1) command / trigger received → (2) authenticated & authorised → (3) executed → (4) status reported → (5) logged.** Test environment: Mission Control staging, robot simulator (hardware-in-the-loop for TC-05, TC-07), chamber test rig, ARLCCS simulator.

| TC-ID | Objective | (1) Received | (2) Auth | (3) Executed | (4) Status reported | (5) Logged | Pass criteria |
|---|---|---|---|---|---|---|---|
| TC-01 | Valid dispatch, end to end | Dispatcher sends `DeliveryRequest(R-07, Clinic-12, B-4471)`; R-07 IDLE, battery 85 %, chamber CONSERVING | Valid session, role Dispatcher → OK | T-01 IDLE → NAVIGATING, then T-04, T-05, T-07 | Dashboard shows NAVIGATING ≤ 10 s; ack ≤ 15 s | `CMD_RECEIVED`, `AUTH_OK`, `DISPATCH_OK`, each state change, `HANDOVER_CONFIRMED` | All 5 stages present with the same command ID; timings met |
| TC-02 | Unauthenticated command rejected | `DeliveryRequest` with no or expired session token | Fails → `AUTH_REQUIRED` | Not forwarded; robot stays IDLE | Operator sees rejection reason | `AUTH_FAIL` with source IP; after 3 failures in 5 min the account is locked (NFR-03) | Robot state unchanged; lockout works |
| TC-03 | Authenticated but wrong role | Viewer sends `SafeModeReset(R-03)`; Dispatcher sends `SafeModeReset(R-03)` | Authenticated, **not authorised** → `ROLE_NOT_PERMITTED` | Not executed (T-28); robot stays SAFE_MODE | Rejection shown | `AUTHZ_FAIL(role, command)` | Both rejected; Supervisor with condition cleared is accepted (T-19) |
| TC-04 | Invalid-state commands | (a) Force `Deliver` to IDLE robot; (b) `DeliveryRequest` to NAVIGATING robot; (c) `DeliveryRequest` at battery 18 % | Supervisor (all authorised) | Rejected by validation: (a) `STATE_NOT_ALLOWED` (T-08), (b) `ROBOT_BUSY` (T-17), (c) `LOW_BATTERY` (T-15) | Reason codes shown | `CMD_REJECTED(reason)` | No state change in any case; never IDLE → DELIVERING |
| TC-05 | Obstacle handling | Simulator places a pedestrian 1.5 m ahead while NAVIGATING; second run at the clinic gate; third run while RETURNING | n/a (sensor event) | T-02 stop ≤ 500 ms; T-03 resume; at the gate DestinationReached ignored (T-09); on the return trip T-11 → T-12 back to RETURNING | AVOIDING_OBSTACLE shown, then the previous state | `OBSTACLE_DETECTED`, `OBSTACLE_CLEARED`, stop latency | Stop ≤ 500 ms; no DELIVERING from AVOIDING; correct state resumed |
| TC-06 | Critical battery abort | BMS reports 20 % (a) while NAVIGATING, (b) while AVOIDING_OBSTACLE, (c) while DELIVERING | n/a | (a) T-06 → RETURNING; (b) T-13 then T-12 → RETURNING; (c) T-14 finish handover → RETURNING | Operator notified with batch ID | `DELIVERY_ABANDONED(B-id)` | Batch kept in chamber; no Safe Mode at 20 % |
| TC-07 | Safe Mode within 3 s (CR-01) | Inject in each of the 6 source states: chamber 12.1 °C; battery 9.9 %; battery temp 61 °C. Boundary: 11.9 °C / 10.1 % must **not** trigger | n/a for entry; reset by Supervisor | T-18 → SAFE_MODE; motion stopped; T-26 rejects other commands | Critical alarm on dashboard | `SAFE_MODE_ENTERED(trigger, latency)` | Latency ≤ 3 s in 18/18 runs; no false trigger at boundaries |
| TC-08 | Chamber correction verified by sensors | Chamber rig heated to 9.5 °C (profile 2–8 °C) | n/a | OP-05 cooling → OP-07 checks 60 s in range → CONSERVING | Excursion and recovery shown in telemetry | `TEMP_EXCURSION`, `TEMP_RECOVERED` | RECOVERED only after 60 s of sensor-confirmed in-range readings, not when the cooler is switched on |
| TC-09 | Correction failure → protection | Cooler disabled on the rig, temperature held at 9 °C | n/a | After 10 min OP-08: light reduced, batch AT_RISK | Operator alert ≤ 10 s | `PROTECTION_ACTIVATED(B-id)` | All three protection actions happen; at > 12 °C the robot also enters SAFE_MODE |
| TC-10 | Railway crossing entry | ARLCCS sim: (a) barrier open, no train; (b) train approaching; (c) status 3 s old; (d) detectors disagree | n/a | (a) cross; (b), (c), (d) hold at the stop line (C-04, C-06, C-07) | Crossing state on dashboard | `CROSSING_ENTERED` / `CROSSING_HOLD(reason)` | Never enters in (b), (c), (d) |
| TC-11 | Link loss & recovery | Link cut for 45 s while NAVIGATING; repeated while inside a crossing; repeated for 10+ min | n/a | T-21 → LINK_LOST at a safe point (inside a crossing: exit first, C-05); T-24 resume; T-25 autonomous return | Dashboard marks robot offline, then back online | Events buffered on the robot, uploaded after reconnection | 0 lost or duplicated log entries (NFR-04); never stops inside a crossing |
| TC-12 | 20-robot load (CR-02) | 20 simulated robots, telemetry every 5 s, 1 command per robot per minute, for 1 h | All commands from a valid Dispatcher | All commands processed | p95 telemetry ≤ 10 s; p95 ack ≤ 15 s | All events logged | 20/20 robots connected the whole time; 0 dropped messages |
| TC-13 | Door opened in transit | Door switch forced open while NAVIGATING | n/a | OP-11 → CH-7; robot stops (C-08) | Tamper alert with GPS position | `DOOR_OPENED`, `TAMPER_ALERT` | Robot stopped; conservation resumes only after OP-12 |
| TC-14 | At-risk batch not delivered | Robot arrives at the clinic with batch AT_RISK | Clinic handover request | T-04 guard fails / OP-15 denies → door stays locked | "Batch held" shown to the clinic and the operator | `REMOVAL_DENIED(AT_RISK)` | No DELIVERING with protection active (C-09, HR-03) |

### Test cases to re-run because of each CR

| CR | Re-run (regression) | New / rewritten | Why |
|---|---|---|---|
| CR-01 | TC-06, TC-09, TC-11 | **TC-07 rewritten** (3 s timing, 3 triggers, boundary values) | TC-06 must confirm 20 % still gives RETURNING, not Safe Mode. TC-09 protection can now trigger Safe Mode. TC-11 checks that LINK_LOST → SAFE_MODE still works. |
| CR-02 | TC-01, TC-11 | **TC-12 new** | TC-01 latency must still pass under load. Link loss and buffered upload with many robots reconnecting at once. |
| CR-03 | TC-01, TC-04, TC-07 (reset step) | **TC-03 new**, TC-02 updated (error code) | Every command path now has an authorisation step. Safe Mode reset is limited to Supervisor. |

## 2.4 Multi-layer RTM

**Requirement → State/Transition (Task 3) → Operation (Task 4) → Constraint (Task 5) → Test Case → Risk Level**

Risk level = severity × likelihood (Critical / High / Medium / Low). Rows marked **(v1.1)** were changed by a CR.

| Requirement | State / Transition | Operation | Constraint | Test Case | Risk |
|---|---|---|---|---|---|
| FR-01 Delivery dispatch | T-01, T-04, T-05, T-07, T-08 ✗, T-15, T-17 | OP-03, OP-R03, OP-R04 | C-09 | TC-01, TC-04 | Medium |
| FR-02 Command validation | T-08 ✗, T-17 | OP-R03 | – | TC-04 | High |
| FR-03 Auth + role **(v1.1, CR-03)** | T-01 guard, T-19, T-20, T-28 | OP-R01, OP-R02 | – | TC-02, TC-03 | High |
| FR-04 Obstacle handling | T-02, T-03, T-09 ✗, T-10, T-11, T-12 | OP-R09 | C-05 | TC-05 | High |
| FR-05 Low-battery abort | T-06, T-12, T-13, T-14, T-16 | OP-R06 | – | TC-06 | Medium |
| FR-06 Safe Mode **(v1.1, CR-01)** | T-18, T-19, T-20, T-26 | OP-R07, OP-08 | C-10 | TC-07 | Critical |
| FR-07 Chamber correction | Chamber CH-4 → CH-5 → CH-4 | OP-04, OP-05, OP-06, OP-07 | – | TC-08 | High |
| FR-08 Protection mode | Chamber CH-5 → CH-6; T-04 guard | OP-08, OP-16 | C-09, C-12 | TC-09 | High |
| FR-09 Telemetry | All states | OP-R05, OP-04 | – | TC-01, TC-12 | Medium |
| FR-10 Event logging | All transitions | OP-R06 | – | TC-01 … TC-14 (stage 5) | Medium |
| NFR-01 Performance | – | OP-R05 | – | TC-01, TC-12 | Medium |
| NFR-02 Scalability **(v1.1, CR-02)** | – | OP-R05 | – | TC-12 | Medium |
| NFR-03 Security | – | OP-R01 | – | TC-02 | High |
| NFR-04 Reliability | T-24 | OP-R06, OP-16 | C-07 | TC-11 | High |
| NFR-05 Usability | – | – | – | Usability trial (5 operators) | Low |
| HR-01 Crossing entry | NAVIGATING / RETURNING (crossing guard) | OP-R08 | C-01, C-02, C-03, C-04, C-06, C-07, C-11 | TC-10 | Critical |
| HR-02 Link loss | T-21, T-22, T-23, T-24, T-25, T-27 | OP-R08, OP-R06 | C-05, C-07 | TC-11 | Critical |
| HR-03 At-risk batch | T-04 guard; CH-6 | OP-08, OP-15 | C-08, C-09, C-12 | TC-13, TC-14 | Critical |
| HR-04 Obstacle stop ≤ 500 ms | T-02, T-11 | OP-R09 | – | TC-05 | Critical |

✗ = invalid transition that the test checks is **rejected**.

### Coverage checks I did on the RTM

- **Forward:** every requirement has at least one test case. NFR-05 is checked by a usability trial instead of an automated TC.
- **Backward:** every TC traces to at least one requirement. Every constraint C-01 to C-12 appears in at least one row.
- **Transitions:** all T-01 to T-28 appear in the RTM, including the invalid ones (T-08, T-09), so the "must not happen" rules are also tested.
- **CR impact:** the three modified rows lead directly to the re-run list in 2.3.
