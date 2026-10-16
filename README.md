# SACDCS – Software Quality Engineering (Open-Ended Lab)

**Smart Autonomous Cold-Chain Delivery & Control System**

This repo is my work for the Open-Ended Lab. I took the role of Software Quality Engineering Lead for the SACDCS pilot (20 clinics). I analysed, modelled, constrained and verified the system across five tasks.

## Contents

| Task | File | What it covers |
|------|------|----------------|
| 1 | [Task1_Requirements.md](Task1_Requirements.md) | 10 FRs, 5 NFRs, 4 high-risk requirements (baseline v1.0) |
| 2 | [Task2_ChangeRequests_RTM.md](Task2_ChangeRequests_RTM.md) | Change log for CR-01 to CR-03, multi-layer RTM, test cases and re-run analysis |
| 3 | [Task3_State_Model.md](Task3_State_Model.md) | Robot states, events, transition table, Safe Mode / Link Lost, verifier findings |
| 4 | [Task4_Chamber_Operations.md](Task4_Chamber_Operations.md) | Conservation chamber operations and operation schemas |
| 5 | [Task5_Railway_Constraints.md](Task5_Railway_Constraints.md) | Constraints, logical formalisation, violation scenarios, detection and response |

## ID scheme used across all tasks

| Prefix | Meaning | Defined in |
|--------|---------|-----------|
| `FR-xx` | Functional requirement | Task 1 |
| `NFR-xx` | Non-functional requirement | Task 1 |
| `HR-xx` | High-risk (safety) requirement | Task 1 |
| `CR-xx` | Change request | Task 2 |
| `TC-xx` | Test case | Task 2 |
| `S-x` / `E-xx` / `T-xx` | Robot state / event / transition | Task 3 |
| `CH-x` | Chamber state | Task 4 |
| `OP-xx` / `OP-Rxx` | Chamber operation / robot & mission-control operation | Task 4 |
| `C-xx` | Constraint | Task 5 |

I wrote Task 1 and Tasks 3 to 5 first and the RTM (Task 2) last, so that every ID in the matrix points to something that actually exists.

## Assumptions I made

The scenario leaves some values open, so I fixed them to keep the requirements testable:

- Medicine profile example: vaccines at **+2 °C to +8 °C**, humidity **35 % to 65 % RH**.
- Low-bandwidth link: one-way delay up to **5 s**. Heartbeat every **5 s**. The link counts as *lost* after **30 s** without a heartbeat.
- Battery levels: **critical = 20 %** (abandon the delivery and return). **Emergency = 10 %** (Safe Mode, from CR-01).
- Chamber recovery period: **10 min**. Vibration stabilisation period: **60 s**.
- Operator roles: **Viewer**, **Dispatcher**, **Supervisor**, **Administrator** (see Task 1, FR-03).
