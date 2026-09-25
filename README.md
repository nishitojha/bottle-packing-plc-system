##### **Description :**



This project implements a PLC-based control system for a bottle packing line — infeed, filling, capping, labeling, reject handling, transfer, and case packing — built and iteratively hardened in OpenPLC. Unlike many introductory or tutorial-level ladder logic designs for similar systems, which typically treat Emergency Stop as a single momentary input with no persistent fault state, this design distinguishes a normal operator stop from a latched safety trip, requires explicit fault acknowledgment via a Reset button (interlocked so it can't clear while the E-stop is still physically engaged), and ensures every station actuator — not just the main conveyor — is force-stopped mid-cycle during a fault rather than continuing to run to completion. The design went through several corrective revisions addressing latch/reset logic, contact polarity, and fault-propagation gaps.



###### **I/O Table :**



| # | Name | Class | Type | Location | Description |

|---|------|-------|------|----------|-------------|

| 0 | Emergency\_PB | Local | BOOL | %IX0.0 | Physical emergency stop pushbutton input. Drives Safety\_Trip and Siren; sets RS0 latch. |

| 1 | Start\_PB | Local | BOOL | %IX0.1 | Operator pushbutton to start the system (energizes M0 run permissive). |

| 2 | Stop\_PB | Local | BOOL | %IX0.2 | Operator pushbutton for a normal, non-fault stop. Resets M0/M1 latches alongside Emergency\_PB. |

| 3 | Fill\_PositionSensor | Local | BOOL | %IX0.4 | Detects a bottle in position at the filling station; triggers the fill pulse timer (TP0). |

| 4 | Fill\_LevelSensor | Local | BOOL | %IX0.5 | Verifies correct fill level after filling; drives RejectAcctuator\_1 when fill is incorrect. |

| 5 | Cap\_PresentSensor | Local | BOOL | %IX0.6 | Detects a bottle in position at the capping station; triggers the cap pulse timer (TP1). |

| 6 | NA\_LabelSensor | Local | BOOL | %IX0.7 | Detects a missing/misapplied label after labeling; drives RejectAcctuator\_2. |

| 7 | To\_M\_Conv\_Sensor | Local | BOOL | %IX1.0 | Detects a bottle reaching the transfer point to the main conveyor; triggers TransferAcctuator and sets M1. |

| 8 | CaseFull\_Sensor | Local | BOOL | %IX1.1 | Detects a bottle entering the case at the packing station; increments CTU0 (bottle count per case). |

| 9 | M0 | Local | BOOL | %IX1.2 | Internal run-permissive bit for the front end of the line (infeed side), set/reset via Start\_PB/Stop\_PB/Emergency\_PB latch. |

| 10 | M1 | Local | BOOL | %IX1.3 | Internal run-permissive bit for the main conveyor, set on transfer, reset via Stop\_PB/Emergency\_PB latch (RS2). |

| 11 | Safety\_Trip | Local | BOOL | %IX1.4 | Latched fault flag (RS0) set by Emergency\_PB; drives Siren, StackLight\_Red, and blocks/kills station actuators during a fault. |

| 12 | Fill\_Active | Local | BOOL | %IX1.5 | Output of the fill-duration pulse timer (TP0); indicates the fill station is mid-cycle. |

| 13 | Label\_PositionSensor | Local | BOOL | %IX1.6 | Detects a bottle in position at the labeling station; triggers the label pulse timer (TP2). |

| 14 | Reset\_PB | Local | BOOL | %IX1.7 | Operator pushbutton to clear Safety\_Trip (RS0 reset), interlocked so it only clears once Emergency\_PB is physically released. |

| 15 | InfeedConv\_M | Local | BOOL | %QX0.0 | Infeed conveyor motor output; runs on M0 with Safety\_Trip and station-busy interlocks. |

| 16 | MainConv\_M | Local | BOOL | %QX0.1 | Main conveyor motor output; runs on M1 latch (RS1), reset by Stop\_PB/Emergency\_PB. |

| 17 | FillValve | Local | BOOL | %QX0.2 | Fill valve output for dispensing product into the bottle at the filling station. |

| 18 | CapHead\_M | Local | BOOL | %QX0.3 | Capping head motor/actuator output; runs on Cap\_PresentSensor pulse (TP1), gated by Safety\_Trip at both trigger and coil. |

| 19 | LabelApplicator\_M | Local | BOOL | %QX0.4 | Label applicator motor/actuator output; runs on Label\_PositionSensor pulse (TP2), gated by Safety\_Trip at both trigger and coil. |

| 20 | RejectAcctuator\_1 | Local | BOOL | %QX0.5 | Reject mechanism for bottles that fail the fill-level check (driven by Fill\_LevelSensor). |

| 21 | RejectAcctuator\_2 | Local | BOOL | %QX0.6 | Reject mechanism for bottles that fail the label check (driven by NA\_LabelSensor). |

| 22 | TransferAcctuator | Local | BOOL | %QX0.7 | Actuator that transfers a bottle onto the main conveyor when To\_M\_Conv\_Sensor triggers. |

| 23 | CasePacker\_M | Local | BOOL | %QX1.0 | Case packer motor output; runs for a fixed pulse (TP3) once CTU0 reaches the per-case bottle count, gated by Safety\_Trip at both trigger and coil. |

| 24 | StackLight\_Green | Local | BOOL | %QX1.1 | Stack light indicating the system is running normally (M0 active). |

| 25 | StackLight\_Red | Local | BOOL | %QX1.2 | Stack light indicating an active fault (Safety\_Trip is true). |

| 26 | StackLight\_Yellow | Local | BOOL | %QX1.3 | Stack light indicating the system is halted (Emergency\_PB or Stop\_PB pressed, non-fault stop). |

| 27 | Alarm\_Buzzer | Local | BOOL | %QX1.4 | Audible alarm output tied to Emergency\_PB/Stop\_PB press events. |

| 28 | Siren | Local | BOOL | %QX1.5 | Audible siren output tied to the latched Safety\_Trip fault condition. |

| 29 | TP0 | Local | TP | | Pulse timer (5s) controlling Fill\_Active duration at the filling station. |

| 30 | TP1 | Local | TP | | Pulse timer (3s) controlling CapHead\_M run duration at the capping station. |

| 31 | TP2 | Local | TP | | Pulse timer (3s) controlling LabelApplicator\_M run duration at the labeling station. |

| 32 | TP3 | Local | TP | | Pulse timer (5s) controlling CasePacker\_M run duration once a case is full. |

| 33 | CTU0 | Local | CTU | | Up-counter tracking bottles per case (preset 12); resets on TP3.Q (completed pack cycle only). |

| 34 | TON0 | Local | TON | | Delay-on timer (3s) used as a debounce/transport delay ahead of RejectAcctuator\_1. |

| 35 | TON1 | Local | TON | | Delay-on timer (3s) used as a debounce/transport delay ahead of RejectAcctuator\_2. |

| 36 | TON2 | Local | TON | | Delay-on timer (2s) used as a transport delay ahead of TransferAcctuator. |

| 37 | RS0 | Local | RS | | Set/reset latch for Safety\_Trip: set by Emergency\_PB, reset by Reset\_PB AND (NOT Emergency\_PB). |

| 38 | RS1 | Local | RS | | Set/reset latch for MainConv\_M: set by M1, reset by Stop\_PB OR Emergency\_PB. |

| 39 | RS2 | Local | RS | | Set/reset latch for M1: set by To\_M\_Conv\_Sensor, reset by Stop\_PB OR Emergency\_PB. |





## **Explanation :**



## **1. Why Safety\_Trip Is a Separate Latched Bit From a Normal Stop**



* A control system needs to distinguish between two fundamentally different events: an operator choosing to pause the line, and something going wrong. These feel similar in the moment — both result in the line stopping — but they carry different implications and need different operator responses, so collapsing them into a single signal is a mistake even though it's a very natural first-draft simplification.



* If Stop\_PB and Emergency\_PB both feed the same flag with no distinction, the operator's panel gives identical feedback — same red light, same siren — whether someone paused the line for a routine reason or an actual fault occurred. Over time, this trains the operator to treat the red light as background noise, because it comes on constantly during normal operation. That's the mechanism behind alarm fatigue: when every event looks equally urgent, none of them are treated as urgent, and the one time it actually matters, the response is dulled by habit. This system avoids that by keeping Stop\_PB's effect (a clean halt, no latch, resumes on the next Start) architecturally separate from Safety\_Trip (a latched fault state set only by Emergency\_PB, requiring deliberate acknowledgment to clear). The stack lights and audible outputs follow this same split — StackLight\_Yellow and Alarm\_Buzzer track a halted state, while StackLight\_Red and Siren track an actual fault — so the operator can tell the two apart at a glance instead of guessing.



###### **2. Why RS-Block Resets and NC/NO Contact Choices Matter**



* Contact polarity (NC vs. NO) isn't a stylistic choice — it determines whether a signal represents a condition or an event, and those two things behave completely differently over the scan cycle, which matters a great deal when the signal feeds into a latch.



* A run-permissive check, like verifying Safety\_Trip is clear before allowing Start\_PB to energize M0, needs an NC contact, because the logic needs to continuously confirm "the fault condition is not currently present." NC contacts pass power whenever the underlying condition is false, meaning the permissive holds as long as the condition stays clear — exactly what a continuous check requires.



* A reset input on a latch is different, and this is the distinction that isn't obvious until you've been burned by it: RS/SR blocks are memory elements. They don't need a continuously-held signal to stay reset — they need a single momentary pulse to force the reset, and then the block itself remembers that state going forward. If a reset input is wired NC, the reset condition is true by default during all normal operation (since the button isn't pressed), which means the Reset input to the latch is asserted almost all the time. On most RS/SR implementations, Reset takes priority over Set, which means a permanently-true Reset input prevents the latch from ever being set in the first place — the output can never turn on at all, because the reset is effectively always winning. The correct choice is a momentary NO contact: false during normal operation, true only for the instant the button is pressed, which is all a latch needs to register the reset and hold it going forward without the operator's finger staying on the button.



* This project applies that distinction consistently: NC contacts for permissives (checking Safety\_Trip before allowing a start), NO contacts for anything feeding a Set or Reset input on an RS/SR block (Stop\_PB and Emergency\_PB resetting RS1 and RS2; Reset\_PB, interlocked with a released Emergency\_PB, resetting RS0). Getting this backward on any single latch produces a system that either never starts or never actually stays stopped.



###### **3. Why Safety\_Trip Is Gated at Both the Trigger and the Coil for Capping, Labeling, and Case Packing**



* TP (pulse timer) blocks are non-retriggerable: once the IN input goes true, the block ignores IN entirely for the duration of its preset time. The output stays true for the full preset regardless of what happens afterward, and there's no way to interrupt it early from the IN side.



* This creates a problem if Safety\_Trip is only checked before the timer's IN input. Gating the trigger stops a new cycle from starting during a fault, but it does nothing for a cycle that's already running — if Emergency\_PB is pressed one second into a three-second cap or label pulse, that TP block has already latched its output true and will run the remaining two seconds to completion no matter what the fault state becomes in the meantime. The motor keeps moving. For a physical mechanism like a capping head or a case packer, a "safety" system that lets the actuator run to completion after an E-stop is pressed isn't safety — it defeats the entire purpose of the button.



* The fix requires gating both ends of the rung: Safety\_Trip before the trigger (or count input, for the counter-driven case packer) to prevent new cycles from starting or counts from advancing during a fault, and Safety\_Trip in series with the actuator coil itself to cut power immediately to a cycle that was already in progress the instant the fault occurred. Gating only the first stops new activity but leaves a live actuator running; gating only the second (as an earlier revision of this project did) blocks the actuator but lets the TP timer run to internal completion anyway, invisibly — the pulse fires and expires while blocked, and if the fault clears after that window closes, that action is simply lost with nothing tracking that it should have happened. Both gates are necessary, and they solve two different halves of the same underlying problem.



###### **4. Why CTU0's Reset Comes Only From TP3.Q, Never From Safety\_Trip**



* CTU0 counts bottles toward a full case (preset of 12) and resets after a completed pack cycle, signaled by TP3.Q. An earlier version of this design also tied Safety\_Trip into that reset condition, on the reasoning that a fault should clear everything cleanly — but that reasoning doesn't hold up once you consider what the counter actually represents.



* CTU0's count is a record of physical reality: bottles that have actually been counted into the case so far. If a fault occurs partway through a batch — say seven bottles counted toward the twelve-bottle target — those seven bottles are still physically present in the case. They don't un-happen because a fault was raised. If Safety\_Trip resets the counter to zero at that moment, the system loses track of what's actually in the case. When the fault clears and production resumes, the counter starts fresh from zero and counts twelve more bottles before triggering the case packer again — meaning the case ends up with nineteen bottles instead of twelve, silently, because nothing in the logic distinguishes "the count is now wrong" from "the count is correct."



* The correct behavior separates two different concerns that got conflated: counting should pause during a fault (handled correctly by gating CTU0's count-up input with Safety\_Trip, so no new bottles are added to the tally while the line is stopped), but the accumulated count itself should not be discarded just because a fault occurred, since the physical bottles it represents haven't disappeared. Reset should only ever reflect an actual event that makes the count meaningless — a completed pack cycle, signaled by TP3.Q — not a fault state that has nothing to do with whether the count is still valid.



###### **5. The Serial, Not Pipelined, Process Flow**



* The infeed conveyor's interlock blocks it from advancing while filling, capping, or labeling is active at any station. In practice, this means the line processes one bottle through the entire fill → cap → label sequence before the next bottle is allowed to advance, rather than running multiple bottles through different stations simultaneously.



* This is a real limitation relative to how bottling lines actually operate physically — a real line keeps bottles moving continuously, with different bottles at fill, cap, and label at the same time, which is the entire reason a conveyor-based design exists instead of a single-station batch process. It's called out explicitly here rather than left implicit, because a design decision that reduces throughput should be a stated scope choice, not something a reader has to infer for themselves by tracing through the interlock logic. Pipelining the line properly would require independent interlocking per station rather than a single shared busy-check gating the infeed conveyor, and is noted as a known extension beyond the current scope of this project.



##### Limitations



* Software-only E-stop — Emergency\_PB is a PLC bit, not a hardwired safety circuit; won't stop motors if the PLC itself faults.
* Serial, not pipelined — infeed halts while any station is busy; only one bottle moves through fill→cap→label at a time.
* No sensor debounce — inputs feed logic directly with no noise filtering.
* Reject timing is fixed-delay, not position-tracked — can misalign if line speed varies.
* No completion confirmation — CapHead\_M, LabelApplicator\_M, CasePacker\_M all run on a timer with no feedback that the action actually completed.
* No manual/jog mode — full-auto or full-stop only.
* No jam detection on conveyors.
* Case count (CTU0) has no external visibility — no HMI/log for partial batches.
* Hardcoded batch size (12) — not a configurable parameter.
* No sensor/PLC self-diagnostics beyond OpenPLC defaults.



Future Updates



* Hardware safety relay for E-stop --> replace the software-only Emergency\_PB check with a hardwired safety relay in series with motor contactor coils, independent of PLC scan logic. (Note: this is a hardware addition, not a logic change — worth stating that distinction in the README itself so it doesn't read as just another code task on the list.)
* Pipeline the station interlocks --> redesign infeed/station interlocking so multiple bottles can be at fill, cap, and label simultaneously, instead of the current serial one-bottle-at-a-time flow.
* Position-tracked reject timing --> replace the fixed-delay TON-based reject actuation with a shift-register or counter-based approach that tracks the specific bottle to the reject point, so timing stays correct even if line speed varies.
* Completion feedback on station actuators --> add limit switches or position sensors on CapHead\_M, LabelApplicator\_M, and CasePacker\_M so a jam or stall is detected instead of assumed-complete after a fixed timer.
* Case count visibility on the HMI --> expose CTU0's live count and flag partial/short cases if the line stops mid-batch, rather than keeping it purely internal to the PLC.
* Configurable batch size --> move the case-size preset (currently hardcoded at 12) into a parameter that can be adjusted without editing the counter block directly.

