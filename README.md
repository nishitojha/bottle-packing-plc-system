##### **Description :**



This project implements a PLC-based control system for a bottle packing line — infeed, filling, capping, labeling, reject handling, transfer, and case packing — built and iteratively hardened in OpenPLC. Unlike many introductory or tutorial-level ladder logic designs for similar systems, which typically treat Emergency Stop as a single momentary input with no persistent fault state, this design distinguishes a normal operator stop from a latched safety trip, requires explicit fault acknowledgment via a Reset button (interlocked so it can't clear while the E-stop is still physically engaged), and ensures every station actuator — not just the main conveyor — is force-stopped mid-cycle during a fault rather than continuing to run to completion. The design went through several corrective revisions addressing latch/reset logic, contact polarity, and fault-propagation gaps.



###### **I/O Table :**


https://1drv.ms/x/c/1d28b52533173d0c/IQCbw198xIr6Q5I2I3SglyJ-AbY1l75Z9DMoTKS\_PIOyuaw?e=Wqlyh41




**Explanation :**
---


**1. Why Safety\_Trip Is a Separate Latched Bit From a Normal Stop**
---



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





