# Task 5 – Constraints, Formalisation & Violations (Railway Crossing + Robot + Chamber)

## 5.0 Variables used

I defined every variable so that a runtime monitor or test oracle can read its value directly from a sensor or a status message.

| Variable | Type | Source |
|---|---|---|
| `Train_Approaching` | Bool | ARLCCS approach sensors (2 independent detectors, A and B, about 2 km out) |
| `Train_Present` | Bool | ARLCCS island-circuit sensor at the crossing |
| `Det_A`, `Det_B` | Bool | Individual approach detectors |
| `Barrier_Open`, `Barrier_Closed` | Bool | Barrier position limit switches |
| `Barrier_Cmd_Close` | Bool | ARLCCS command to the barrier motor |
| `t_cmd` | seconds | Time since `Barrier_Cmd_Close` became TRUE |
| `Lights_On`, `Alarm_On` | Bool | Warning light / audible alarm feedback |
| `Crossing_Clear` | Bool | Safety-monitoring unit: no train and no object in the crossing area |
| `Sensor_Fault` | Bool | ARLCCS self-diagnostics |
| `Status_Age` | seconds | Age of the last ARLCCS status message received by the robot |
| `Robot_Enters` | Bool | Robot's front edge crosses the stop line |
| `Robot_In_Crossing` | Bool | Robot position is inside the crossing polygon |
| `Robot_Speed` | m/s | Robot odometry |
| `Robot_State` | enum | Task 3 states (IDLE … LINK_LOST) |
| `At_Warehouse` | Bool | Robot docked |
| `Door_Open` | Bool | Chamber door switch |
| `Protection_Active`, `Batch_At_Risk` | Bool | Chamber (OP-08) |
| `Chamber_Temp`, `Battery_Level`, `Battery_Temp` | °C / % / °C | Chamber probe, BMS |
| `t_trig` | seconds | Time since a Safe Mode trigger condition became TRUE |

Thresholds: `T_crit` = 12 °C (vaccine profile), `BT_crit` = 60 °C, emergency battery = 10 %, barrier closing limit = 8 s, status freshness = 2 s.

## 5.1 Constraints

| C-ID | Constraint (simple English) | Why it is necessary |
|---|---|---|
| C-01 | When a train is in the crossing, the barrier must not be open. | Basic protection of road users (including our robots) from a train. |
| C-02 | When a train is approaching, the barrier must be closed and the lights and alarm must be on. | Gives enough warning time. A barrier that closes too late is as dangerous as one that stays open. |
| C-03 | The barrier may only be open if the crossing has been confirmed clear and no train is approaching or present. | Opening after a train has passed must not happen while a second train is approaching on the other track. |
| C-04 | The robot may only enter the crossing when the barrier is open, no train is approaching, and the ARLCCS status is no older than 2 s. | Core safety rule from the scenario (HR-01). A stale status could hide a train that has just been detected. |
| C-05 | While the robot is inside the crossing it must keep moving toward the exit, and ARLCCS must know that the crossing is occupied. | A robot stopped on the tracks (link loss, Safe Mode, obstacle) is a collision risk, and ARLCCS must be able to warn the train. |
| C-06 | If any crossing sensor has failed, or the two approach detectors disagree, the crossing must be treated as unsafe and the barrier kept closed. | Fail-safe design. An incorrect reading must never make the crossing look safe. |
| C-07 | If the link to ARLCCS is down, or the last status is older than 2 s, the robot must not be permitted to enter. | Communication loss: no information means no permission. |
| C-08 | The chamber door may only be open while the robot is stationary **and** either in DELIVERING or IDLE at the warehouse. | Robot–chamber link. Stops tampering or theft in transit and stops medicine being exposed while the robot moves. |
| C-09 | The robot may only be in DELIVERING if the chamber has no active protection and the batch is not at risk. | Robot–chamber link (HR-03): a damaged vaccine must not reach a patient. |
| C-10 | If chamber temperature exceeds the critical threshold, or battery level falls below the emergency level, or battery temperature exceeds the critical threshold, the robot must be in SAFE_MODE within 3 s. | CR-01 / FR-06: a measurable time limit for the safety reaction. |
| C-11 | If the barrier was commanded to close but has not reached the closed position within 8 s, a barrier fault must be raised and the crossing treated as unsafe. | Barrier failure: a stuck barrier must be detected, not assumed closed. |
| C-12 | A batch may only be removed when no protection response is active and the chamber is not in fault. | Scenario rule for batch removal (OP-15). |

## 5.2 Formalisation

Notation: ∧ and, ∨ or, ¬ not, → implies. Time limits are written as explicit conditions on timer variables so that a monitor can evaluate them.

| C-ID | Formal constraint |
|---|---|
| C-01 | `Train_Present → ¬Barrier_Open` |
| C-02 | `Train_Approaching → (Barrier_Closed ∧ Lights_On ∧ Alarm_On)` |
| C-03 | `Barrier_Open → (Crossing_Clear ∧ ¬Train_Approaching ∧ ¬Train_Present)` |
| C-04 | `Robot_Enters → (Barrier_Open ∧ ¬Train_Approaching ∧ ¬Train_Present ∧ Status_Age ≤ 2)` |
| C-05 | `Robot_In_Crossing → (Robot_Speed > 0 ∧ Crossing_Occupied_Reported)` |
| C-06 | `(Sensor_Fault ∨ (Det_A ≠ Det_B)) → (Train_Approaching ∧ Barrier_Closed)` *(failure is treated as "train approaching")* |
| C-07 | `(¬Link_OK ∨ Status_Age > 2) → ¬Entry_Permitted` |
| C-08 | `Door_Open → (Robot_Speed = 0 ∧ (Robot_State = DELIVERING ∨ (Robot_State = IDLE ∧ At_Warehouse)))` |
| C-09 | `Robot_State = DELIVERING → (¬Protection_Active ∧ ¬Batch_At_Risk)` |
| C-10 | `((Chamber_Temp > T_crit ∨ Battery_Level < 10 ∨ Battery_Temp > BT_crit) ∧ t_trig > 3) → Robot_State = SAFE_MODE` |
| C-11 | `(Barrier_Cmd_Close ∧ t_cmd > 8 ∧ ¬Barrier_Closed) → (Barrier_Fault ∧ Train_Approaching_Assumed)` |
| C-12 | `Batch_Removed → (¬Protection_Active ∧ Chamber_State ≠ FAULT)` |

## 5.3 / 5.4 Violation scenarios, detection and expected response

| C-ID | Violation scenario (variable values) | How it is detected | Expected response |
|---|---|---|---|
| C-01 | Barrier motor controller fault re-opens the barrier while a freight train is still passing: `Train_Present = TRUE`, `Barrier_Open = TRUE`. | Safety-monitoring unit compares the island-circuit sensor with the barrier limit switch every 100 ms. | Force barrier closed (fail-safe spring). Lights and alarm on. Broadcast `CROSSING_UNSAFE` to all robots; any robot at the stop line holds. Operator alert. Event logged. |
| C-02 | Train detected 2 km away but the warning light bulb has failed: `Train_Approaching = TRUE`, `Barrier_Closed = TRUE`, `Lights_On = FALSE`, `Alarm_On = TRUE`. | Light current feedback reads 0 A while it is commanded on. | Keep the barrier closed. Raise a maintenance alarm. Mark the crossing DEGRADED (robots may not cross until it is repaired). Log. |
| C-03 | First train has passed, but a second train is approaching on track 2 and the barrier starts to lift: `Barrier_Open = TRUE`, `Train_Approaching = TRUE`, `Crossing_Clear = TRUE`. | Interlock checks `Train_Approaching` before every open command; the monitor flags `Barrier_Open ∧ Train_Approaching`. | Abort the open and re-close the barrier. Robots receive `Train_Approaching = TRUE` and hold. Log `INTERLOCK_VIOLATION`. |
| C-04 | Robot uses a cached status: `Barrier_Open = TRUE`, `Train_Approaching = FALSE`, but `Status_Age = 6 s` (a train was detected 4 s ago). The robot crosses the stop line: `Robot_Enters = TRUE`. | Onboard crossing guard (OP-R08) checks `Status_Age` before releasing the brake; ARLCCS stop-line camera also detects entry without permission. | Robot brakes at the stop line (it must not get to `Robot_Enters`). If it has already entered: continue to exit at full speed (C-05), ARLCCS raises occupied alarm. Operator alert, **Safe Mode** review afterwards. Log. |
| C-05 | Robot enters, and the link is lost halfway; naive logic stops it on the rails: `Robot_In_Crossing = TRUE`, `Robot_Speed = 0`, `Robot_State = LINK_LOST`. | Onboard monitor: position inside the crossing polygon with speed 0 for > 1 s. ARLCCS occupancy sensor also sees an object. | Override the LINK_LOST stop: drive to the exit (T-21 guard). If physically blocked: ARLCCS `CROSSING_OBSTRUCTED` → railway control alerted to stop the train. Operator alert. Log. |
| C-06 | Detector A reports a train and detector B does not (B is frozen): `Det_A = TRUE`, `Det_B = FALSE`, `Sensor_Fault = FALSE`, `Barrier_Open = TRUE`. | Cross-check of the redundant detectors every cycle; the disagreement is a violation. | Treat as a train approaching: close the barrier, lights and alarm on. Report `SENSOR_DISAGREE`. Robots hold. Maintenance ticket. Log. |
| C-07 | Robot at the stop line, cellular link down for 12 s: `Link_OK = FALSE`, `Status_Age = 12`, `Entry_Permitted = TRUE` (bug: last permission kept). | Onboard monitor checks the timestamp of the permission each control cycle. | Revoke permission. Robot holds at the stop line, and after 30 s enters LINK_LOST (HR-02) at that safe point. Events buffered. Log on reconnect. |
| C-08 | Door switch reads open while the robot drives at 1.2 m/s on the road: `Door_Open = TRUE`, `Robot_Speed = 1.2`, `Robot_State = NAVIGATING`. | Chamber OP-11 detects the door change; the robot state is not DELIVERING → violation. | Robot stops at a safe point. Tamper alert to the operator with the GPS position. Chamber suspends (CH-7). If the temperature becomes critical → **Safe Mode** (C-10). Log. |
| C-09 | Correction failed earlier, but the robot arrives and the handover starts anyway: `Robot_State = DELIVERING`, `Protection_Active = TRUE`, `Batch_At_Risk = TRUE`. | Guard on T-04 and OP-15 checks the chamber flags; the monitor flags the combination. | Refuse handover (door stays locked). Clinic display shows "Batch held – contact Mission Control". Operator alert. Robot → RETURNING after a Supervisor decision. Log. |
| C-10 | Chamber temperature reaches 13.5 °C, but 5 s later the robot is still NAVIGATING: `Chamber_Temp = 13.5`, `t_trig = 5`, `Robot_State = NAVIGATING`. | Watchdog compares the trigger timestamp with the state-change timestamp. | Hardware watchdog forces a motion stop and `SAFE_MODE`. Operator alert of priority "critical". Log the timing breach as a defect. |
| C-11 | Barrier commanded to close, but it is stuck at 45° after 10 s: `Barrier_Cmd_Close = TRUE`, `t_cmd = 10`, `Barrier_Closed = FALSE`. | Limit-switch timeout in the ARLCCS safety-monitoring unit. | `BARRIER_FAULT`: lights and alarm stay on, crossing UNSAFE, robots hold. Railway control notified. Log. |
| C-12 | Clinic staff try to remove a batch during protection mode: `Batch_Removed = TRUE`, `Protection_Active = TRUE`. | OP-15 rejects the unlock; if the door is forced, the door switch with `Protection_Active` is a violation. | Door stays locked. If forced: tamper alert, batch marked "removed under protection", Supervisor alerted. Log. |

## 5.5 Coverage check

- **At least two robot–chamber constraints:** C-08 (door vs. robot motion/state), C-09 (delivering vs. protection), plus C-10 and C-12.
- **Sensor failure or communication loss:** C-06 (sensor failure / disagreeing readings), C-07 (communication loss), C-11 (barrier failure).
- **Railway crossing:** C-01 to C-07, C-11.
- All 12 constraints are formalised, and each one has a violation scenario with concrete values. Each formula uses only observable variables, so a runtime monitor can evaluate it every cycle and a test can assert it.
