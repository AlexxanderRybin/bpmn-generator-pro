# BPMN Anti-Patterns - Complete Catalog

**Reference for:** `SKILL.md` Anti-Patterns section

**Purpose:** Comprehensive catalog of BPMN anti-patterns to avoid, with examples and fixes.

**Source:** Based on research analyzing thousands of BPMN models, identifying most common mistakes.

---

## Anti-Pattern #1: Wrong Usage of Connecting Objects ❌

**Prevalence:** 48% of models (MOST COMMON ERROR)

**Problem:** Using wrong type of flow for connections.

### Three Critical Rules

**Rule 1: Sequence Flow INSIDE Pool Only**
```xml
<!-- ✅ CORRECT: Sequence Flow connects elements within Pool -->
<bpmn:sequenceFlow sourceRef="Task_A" targetRef="Task_B" />
```

**Rule 2: Message Flow BETWEEN Pools Only**
```xml
<!-- ✅ CORRECT: Message Flow crosses Pool boundaries -->
<bpmn:messageFlow sourceRef="Task_SendOrder" targetRef="Task_ReceiveOrder" />
```

**Rule 3: No Message Flow Inside Pool**
```xml
<!-- ❌ WRONG: Message Flow within Pool -->
<bpmn:messageFlow sourceRef="Task_A" targetRef="Task_B" />
<!-- Both tasks in same Pool - should be Sequence Flow! -->
```

### Common Mistakes

**Mistake 1: Sequence Flow crossing Pool boundary**
```
┌────────────────────────┐
│ Customer Pool          │
│  Task_PlaceOrder ──────┼──→ ❌ Sequence Flow to Supplier
└────────────────────────┘
┌────────────────────────┐
│ Supplier Pool          │
│  Task_ReceiveOrder     │
└────────────────────────┘
```

**FIX:** Use Message Flow instead.

**Mistake 2: Activities not connected within Pool**
```xml
<!-- ❌ WRONG: No Sequence Flow between tasks -->
<bpmn:userTask id="Task_A" />
<bpmn:userTask id="Task_B" />
<!-- Missing <bpmn:sequenceFlow> connecting them! -->
```

### Visual Reminder
```
INSIDE Pool  → ———— Sequence Flow (solid arrow)
BETWEEN Pools → - - → Message Flow (dashed arrow)
```

---

## Anti-Pattern #2: Missing Start/End Events ❌

**Prevalence:** 25% of models

**Problem:** Process lacks clear boundaries (start/end points).

**WRONG:**
```
[Task 1] → [Task 2] → [Task 3]
```
**Issues:**
- No trigger defined (how does process start?)
- No outcome states (how does process end?)
- Unclear scope (where does process begin/end?)

**CORRECT:**
```
(Start: Order Received) → [Task 1] → [Task 2] → [Task 3] → (End: Order Completed)
```

### Rules

1. **Every process MUST have at least ONE Start Event**
   - Exception: Subprocess without start = inherits trigger

2. **Every process MUST have at least ONE End Event**
   - Can have multiple End Events for different outcomes

3. **Name Start Events with trigger/state**
   - ✅ "Order Received", "Request Submitted"
   - ❌ "Start", "Process Start"

4. **Name End Events with outcome/result**
   - ✅ "Order Completed", "Request Approved", "Process Cancelled"
   - ❌ "End", "Process End", "Stop"

### XML Example

**WRONG:**
```xml
<bpmn:process id="Process_1">
  <!-- Missing Start Event! -->
  <bpmn:userTask id="Task_1" name="Do Something" />
  <bpmn:userTask id="Task_2" name="Do Something Else" />
  <!-- Missing End Event! -->
</bpmn:process>
```

**CORRECT:**
```xml
<bpmn:process id="Process_1">
  <bpmn:startEvent id="Event_Start" name="Request Received" />
  <bpmn:userTask id="Task_1" name="Do Something" />
  <bpmn:userTask id="Task_2" name="Do Something Else" />
  <bpmn:endEvent id="Event_End" name="Request Completed" />

  <!-- Sequence Flows connecting them -->
  <bpmn:sequenceFlow sourceRef="Event_Start" targetRef="Task_1" />
  <bpmn:sequenceFlow sourceRef="Task_1" targetRef="Task_2" />
  <bpmn:sequenceFlow sourceRef="Task_2" targetRef="Event_End" />
</bpmn:process>
```

---

## Anti-Pattern #3: Redundant Event Naming ❌

**Problem:** Event names don't add business value, just repeat element type.

**WRONG:**
```xml
<bpmn:startEvent id="Start1" name="Process Start" />  ❌
<bpmn:startEvent id="Start2" name="Start" />          ❌
<bpmn:endEvent id="End1" name="Process End" />        ❌
<bpmn:endEvent id="End2" name="End" />                ❌
```

**CORRECT:**
```xml
<bpmn:startEvent id="Event_Start" name="Order Received" />      ✅
<bpmn:startEvent id="Event_Trigger" name="Monthly Schedule" />  ✅
<bpmn:endEvent id="End_Success" name="Order Completed" />       ✅
<bpmn:endEvent id="End_Cancel" name="Order Cancelled" />        ✅
```

### Naming Rules

**Start Events:**
- Describe the **trigger** or **initial state**
- ✅ "Customer Request Received", "Timer Fired", "Payment Received"
- ❌ "Start", "Begin Process", "Initiate"

**End Events:**
- Describe the **outcome** or **result state**
- ✅ "Request Approved", "Order Shipped", "Payment Failed", "Process Cancelled"
- ❌ "End", "Finish", "Done", "Complete"

**Intermediate Events:**
- Describe the **event** that occurs
- ✅ "Payment Timeout", "Document Received", "Approval Deadline Reached"
- ❌ "Timer", "Message", "Event"

---

## Anti-Pattern #4: Inconsistent Naming ❌

**Prevalence:** 35% of models

**Problem:** Mixed naming styles reduce readability.

### Common Inconsistencies

**Issue 1: Mixed Verb Forms**
```xml
❌ WRONG:
<bpmn:userTask name="Validation" />           <!-- Noun -->
<bpmn:userTask name="Review application" />   <!-- verb + object, lowercase -->
<bpmn:userTask name="approval of request" />  <!-- noun + preposition -->
```

**CORRECT:**
```xml
✅ CONSISTENT:
<bpmn:userTask name="Validate Application" />
<bpmn:userTask name="Review Application" />
<bpmn:userTask name="Approve Request" />
```

**Issue 2: Mixed Case Styles**
```xml
❌ WRONG:
<bpmn:userTask name="review Application" />   <!-- lowercase verb -->
<bpmn:userTask name="Approve request" />       <!-- lowercase object -->
<bpmn:userTask name="VALIDATE DOCUMENT" />     <!-- all caps -->
```

**CORRECT:**
```xml
✅ CONSISTENT (Title Case):
<bpmn:userTask name="Review Application" />
<bpmn:userTask name="Approve Request" />
<bpmn:userTask name="Validate Document" />
```

### Mandatory Naming Convention

**Tasks: Verb + Object (Title Case)**
- Pattern: `[Action Verb] [Object Noun]`
- ✅ "Review Application", "Calculate Discount", "Send Notification"
- ❌ "application review", "Reviewing", "Calculation of discount"

**Gateways: Question (Title Case)**
- Pattern: `[Question]?` or `[Condition]?`
- ✅ "Approved?", "Amount > 1000?", "Documents Complete?"
- ❌ "Check Approval", "Amount Gateway", "Document Validation"

**Events: State Description (Title Case)**
- Pattern: `[Subject] [State/Action]`
- ✅ "Order Received", "Payment Timeout", "Request Approved"
- ❌ "receiving order", "timeout", "approved"

---

## Anti-Pattern #5: Overcomplicated Models ❌

**Prevalence:** 32% of models

**Problem:** Too many elements on single diagram, reducing comprehensibility.

### Red Flags

**Warning Signs:**
- More than 30 elements on one diagram
- Deep nesting (3+ levels of subprocesses)
- "Everything on one page" mentality
- Diagram requires scrolling/zooming to view

### Complexity Limits

| Complexity | Element Count | Actions |
|------------|---------------|---------|
| **Simple** | ≤ 10 | One diagram, no decomposition needed |
| **Medium** | 10-25 | Consider grouping related tasks |
| **Complex** | 25-30 | **RECOMMENDED**: Decompose into subprocesses |
| **Too Complex** | > 30 | **MANDATORY**: Split into multiple diagrams/subprocesses |

### Solutions

**Solution 1: Extract Subprocesses**
```
BEFORE (40 elements):
Start → [10 tasks for validation] → [15 tasks for processing] → [10 tasks for finalization] → End

AFTER (15 elements + 3 subprocesses):
Start → [Validation Subprocess] → [Processing Subprocess] → [Finalization Subprocess] → End

Each subprocess: 10-15 elements (manageable)
```

**Solution 2: Use Call Activities for Reusable Logic**
```xml
<!-- Main process -->
<bpmn:callActivity id="Activity_Payment" name="Standard Payment Process"
                   calledElement="Process_StandardPayment" />

<!-- Separate process file: standard-payment.bpmn -->
<bpmn:process id="Process_StandardPayment">
  <!-- Complex payment logic here, reused across multiple processes -->
</bpmn:process>
```

**Solution 3: Hierarchical Decomposition**
```
Level 0: High-level process (5-7 major phases)
Level 1: Each phase = subprocess (10-15 elements)
Level 2: Complex phase = nested subprocess (10-15 elements)

STOP at Level 2 - if deeper needed, reconsider design
```

---

## Anti-Pattern #6: Gateway Mistakes ❌

**Prevalence:** 28% of models

Detailed in separate sections below (#6a-6d).

### #6a: Exclusive Gateway Without Default Flow ❌

**Problem:** XOR gateway without default can cause **DEADLOCK** if no condition matches.

**WRONG:**
```xml
<bpmn:exclusiveGateway id="Gateway_Risk">
  <bpmn:outgoing>Flow_High</bpmn:outgoing>
  <bpmn:outgoing>Flow_Low</bpmn:outgoing>
</bpmn:exclusiveGateway>

<bpmn:sequenceFlow id="Flow_High" sourceRef="Gateway_Risk" targetRef="Task_HighRisk">
  <bpmn:conditionExpression>${risk == 'HIGH'}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<bpmn:sequenceFlow id="Flow_Low" sourceRef="Gateway_Risk" targetRef="Task_LowRisk">
  <bpmn:conditionExpression>${risk == 'LOW'}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<!-- ❌ If risk = 'MEDIUM', neither condition true → DEADLOCK! -->
```

**CORRECT:**
```xml
<bpmn:exclusiveGateway id="Gateway_Risk"
  default="Flow_Default">  <!-- ✅ Default flow defined -->
  <bpmn:outgoing>Flow_High</bpmn:outgoing>
  <bpmn:outgoing>Flow_Default</bpmn:outgoing>
</bpmn:exclusiveGateway>

<bpmn:sequenceFlow id="Flow_High" sourceRef="Gateway_Risk" targetRef="Task_HighRisk">
  <bpmn:conditionExpression>${risk == 'HIGH'}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<!-- Default flow has NO condition - always taken if others false -->
<bpmn:sequenceFlow id="Flow_Default" name="Medium/Low"
  sourceRef="Gateway_Risk" targetRef="Task_StandardRisk" />
```

**Rule:** **EVERY** Exclusive Gateway MUST have `default` attribute pointing to one outgoing flow.

### #6b: Overlapping Conditions ❌

**Problem:** Multiple conditions can be true simultaneously in XOR gateway (non-deterministic behavior).

**WRONG:**
```xml
<bpmn:exclusiveGateway id="Gateway_Discount" default="Flow_None" />

<bpmn:sequenceFlow id="Flow_Student">
  <bpmn:conditionExpression>${age < 25}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<bpmn:sequenceFlow id="Flow_Premium">
  <bpmn:conditionExpression>${loyaltyYears > 5}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<!-- ❌ What if age=22 AND loyaltyYears=6? Both conditions true! -->
<!-- XOR should only take ONE path, but which one? -->
```

**CORRECT (Option A: Make Conditions Mutually Exclusive):**
```xml
<bpmn:sequenceFlow id="Flow_Student">
  <bpmn:conditionExpression>${age < 25 && loyaltyYears <= 5}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<bpmn:sequenceFlow id="Flow_Premium">
  <bpmn:conditionExpression>${loyaltyYears > 5}</bpmn:conditionExpression>
</bpmn:sequenceFlow>
```

**CORRECT (Option B: Use Priority Order + Default):**
```xml
<!-- Check Premium first (higher priority) -->
<bpmn:sequenceFlow id="Flow_Premium">
  <bpmn:conditionExpression>${loyaltyYears > 5}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<!-- Check Student second -->
<bpmn:sequenceFlow id="Flow_Student">
  <bpmn:conditionExpression>${age < 25}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<!-- Default if neither -->
<bpmn:sequenceFlow id="Flow_Standard" default="true" />
```

### #6c: Incomplete Conditions ❌

**Problem:** Conditions don't cover all possible values.

**WRONG:**
```xml
<bpmn:sequenceFlow id="Flow_Child">
  <bpmn:conditionExpression>${age < 18}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<bpmn:sequenceFlow id="Flow_Adult">
  <bpmn:conditionExpression>${age >= 18 && age < 65}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<!-- ❌ What if age >= 65? Not covered! Falls through to default (if exists) -->
```

**CORRECT:**
```xml
<bpmn:sequenceFlow id="Flow_Child">
  <bpmn:conditionExpression>${age < 18}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<bpmn:sequenceFlow id="Flow_Adult">
  <bpmn:conditionExpression>${age >= 18 && age < 65}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<bpmn:sequenceFlow id="Flow_Senior">
  <bpmn:conditionExpression>${age >= 65}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<!-- OR use default for last case -->
<bpmn:sequenceFlow id="Flow_Senior" default="true" />
```

### #6d: Unbalanced Gateway Types (DEADLOCK!) ❌

**Problem:** Parallel split with non-parallel join causes token mismatch.

**WRONG:**
```xml
<!-- Parallel split creates 3 tokens -->
<bpmn:parallelGateway id="Gateway_Fork">
  <bpmn:outgoing>Flow_A</bpmn:outgoing>
  <bpmn:outgoing>Flow_B</bpmn:outgoing>
  <bpmn:outgoing>Flow_C</bpmn:outgoing>
</bpmn:parallelGateway>

<!-- ❌ Exclusive join expects 1 token, gets 3! -->
<bpmn:exclusiveGateway id="Gateway_BadJoin">
  <bpmn:incoming>Flow_A</bpmn:incoming>
  <bpmn:incoming>Flow_B</bpmn:incoming>
  <bpmn:incoming>Flow_C</bpmn:incoming>
</bpmn:exclusiveGateway>

<!-- Result: First token passes through, 2 tokens stuck forever → DEADLOCK -->
```

**CORRECT:**
```xml
<bpmn:parallelGateway id="Gateway_Fork">
  <bpmn:outgoing>Flow_A</bpmn:outgoing>
  <bpmn:outgoing>Flow_B</bpmn:outgoing>
  <bpmn:outgoing>Flow_C</bpmn:outgoing>
</bpmn:parallelGateway>

<!-- ✅ Parallel join waits for ALL 3 tokens -->
<bpmn:parallelGateway id="Gateway_Join">
  <bpmn:incoming>Flow_A</bpmn:incoming>
  <bpmn:incoming>Flow_B</bpmn:incoming>
  <bpmn:incoming>Flow_C</bpmn:incoming>
  <bpmn:outgoing>Flow_Continue</bpmn:outgoing>
</bpmn:parallelGateway>
```

**Safe Patterns:**
- Exclusive Split → Exclusive Merge ✅
- Parallel Split → Parallel Join ✅
- Inclusive Split → Inclusive Join ✅

**DANGEROUS Patterns:**
- Parallel Split → Exclusive Join ❌ DEADLOCK
- Exclusive Split → Parallel Join ❌ DEADLOCK
- Inclusive Split → Parallel Join ❌ DEADLOCK

---

## Anti-Pattern #7: Gateway Sending Messages ❌

**Problem:** Gateway directly connected to message throw event.

**Why wrong:** Gateways route tokens; they don't perform actions. Sending messages is an action.

**WRONG:**
```
Gateway_Decision → ○ Message: Notify Customer
```

**CORRECT:**
```
Gateway_Decision → Task: Prepare Notification → ○ Message: Send Notification
```

**Rule:** Messages must be sent FROM tasks (User Task, Service Task, Send Task), not FROM gateways.

---

## Anti-Pattern #8: Inclusive Gateway Token Issues ❌

**Problem:** Tokens terminated between OR-split and OR-join.

**WRONG:**
```xml
<bpmn:inclusiveGateway id="Gateway_Split">
  <bpmn:outgoing>Flow_ProcessA</bpmn:outgoing>
  <bpmn:outgoing>Flow_ProcessB</bpmn:outgoing>
</bpmn:inclusiveGateway>

<!-- Path A terminates early -->
<bpmn:sequenceFlow sourceRef="Task_A" targetRef="End_PathA" />  <!-- ❌ Token terminated! -->

<!-- Path B goes to join -->
<bpmn:inclusiveGateway id="Gateway_Join">
  <bpmn:incoming>Flow_FromA</bpmn:incoming>  <!-- Never arrives! -->
  <bpmn:incoming>Flow_FromB</bpmn:incoming>
</bpmn:inclusiveGateway>

<!-- If both paths activated, Join waits forever for Path A token → DEADLOCK -->
```

**CORRECT:**
```xml
<bpmn:inclusiveGateway id="Gateway_Split">
  <bpmn:outgoing>Flow_ProcessA</bpmn:outgoing>
  <bpmn:outgoing>Flow_ProcessB</bpmn:outgoing>
</bpmn:inclusiveGateway>

<!-- Both paths go to join -->
<bpmn:sequenceFlow sourceRef="Task_A" targetRef="Gateway_Join" />  <!-- ✅ -->
<bpmn:sequenceFlow sourceRef="Task_B" targetRef="Gateway_Join" />  <!-- ✅ -->

<bpmn:inclusiveGateway id="Gateway_Join">
  <bpmn:incoming>Flow_FromA</bpmn:incoming>
  <bpmn:incoming>Flow_FromB</bpmn:incoming>
  <bpmn:outgoing>Flow_Continue</bpmn:outgoing>
</bpmn:inclusiveGateway>

<!-- End after join -->
<bpmn:endEvent id="End_Complete">
  <bpmn:incoming>Flow_Continue</bpmn:incoming>
</bpmn:endEvent>
```

**Rule:** Between Inclusive Split and Inclusive Join, **NEVER** terminate tokens with End Events.

---

## Anti-Pattern #9: Multi-Merge ❌ CRITICAL

**Prevalence:** Very common, especially in complex models

**Problem:** Multiple incoming flows converge directly into element without explicit merge gateway.

**FULL DETAILS:** See `multi-merge-prevention.md`

### Quick Summary

**DANGEROUS (After Parallel Split):**
```xml
<bpmn:parallelGateway id="Gateway_Fork" />
  → Task_A → Task_Final  <!-- ❌ -->
  → Task_B → Task_Final  <!-- ❌ Task_Final receives 2 tokens! -->
```

**Impact:** Task executes TWICE (once per token) → race conditions, duplicate data

**CORRECT:**
```xml
<bpmn:parallelGateway id="Gateway_Fork" />
  → Task_A → Gateway_Join
  → Task_B → Gateway_Join
<bpmn:parallelGateway id="Gateway_Join" />  <!-- ✅ Synchronization -->
  → Task_Final  <!-- Now receives 1 token -->
```

**NOT RECOMMENDED (After Exclusive Splits):**
```xml
<!-- Technically valid but unclear -->
<bpmn:userTask id="Task_Notify">
  <bpmn:incoming>Flow_Path1</bpmn:incoming>  <!-- XOR -->
  <bpmn:incoming>Flow_Path2</bpmn:incoming>  <!-- XOR -->
  <bpmn:incoming>Flow_Path3</bpmn:incoming>  <!-- XOR -->
</bpmn:userTask>
<!-- ⚠️ Only 1 token arrives, but not visually clear -->
```

**BEST PRACTICE:**
```xml
<bpmn:exclusiveGateway id="Gateway_Merge">
  <bpmn:incoming>Flow_Path1</bpmn:incoming>
  <bpmn:incoming>Flow_Path2</bpmn:incoming>
  <bpmn:incoming>Flow_Path3</bpmn:incoming>
  <bpmn:outgoing>Flow_ToTask</bpmn:outgoing>
</bpmn:exclusiveGateway>

<bpmn:userTask id="Task_Notify">
  <bpmn:incoming>Flow_ToTask</bpmn:incoming>  <!-- ✅ Single incoming -->
</bpmn:userTask>
```

**Golden Rule:** Only **merge gateways** and **end events** should have multiple incoming flows.

---

## Anti-Pattern #10: No Error Handling ❌

**Problem:** Critical tasks without error handling (boundary events).

**WRONG:**
```
[Process Payment Task] → [Success Path]
```
**Issues:**
- What if payment fails?
- What if timeout?
- What if invalid card?

**CORRECT:**
```
[Process Payment Task] → [Success Path]
     ⚡ Error: Payment Failed → [Handle Error] → [Notify Customer] → End(Error)
     ⏱ Timer: 30s Timeout → [Retry Payment] → ...
```

**XML Example:**
```xml
<bpmn:serviceTask id="Task_ProcessPayment" name="Process Payment" />

<!-- Boundary Error Event -->
<bpmn:boundaryEvent id="Error_Payment" attachedToRef="Task_ProcessPayment">
  <bpmn:errorEventDefinition errorRef="Error_PaymentFailed" />
  <bpmn:outgoing>Flow_HandleError</bpmn:outgoing>
</bpmn:boundaryEvent>

<!-- Boundary Timer Event -->
<bpmn:boundaryEvent id="Timer_Timeout" attachedToRef="Task_ProcessPayment">
  <bpmn:timerEventDefinition>
    <bpmn:timeDuration>PT30S</bpmn:timeDuration>
  </bpmn:timerEventDefinition>
  <bpmn:outgoing>Flow_Retry</bpmn:outgoing>
</bpmn:boundaryEvent>
```

**Rule:** Add boundary events to tasks that can:
- Fail (API calls, payments, validations)
- Take too long (SLA requirements)
- Receive external signals (cancellation, updates)

---

## Anti-Pattern #11: Multiple Ends Without Semantic Difference ❌

**Problem:** Multiple end events with same meaning (redundant).

**WRONG:**
```
Path A → (End) "Finished"
Path B → (End) "Done"        ❌ Both mean success
Path C → (End) "Completed"   ❌ Redundant
```

**CORRECT:**
```
Path A → Merge Gateway → (End) "Request Approved"  ✅
Path B ↗
Path C ↗
```

**Rule:** Multiple end events are OK ONLY if they represent **different outcomes**:
- ✅ "Order Completed" vs "Order Cancelled" vs "Order Failed"
- ❌ "Success End 1" vs "Success End 2" vs "Done"

---

## Anti-Pattern #12: Implicit Gateway ❌ CRITICAL

**Problem:** Task with multiple outgoing flows (decision logic hidden).

**WRONG:**
```xml
<bpmn:userTask id="Task_RequestDocuments">
  <bpmn:outgoing>Flow_Continue</bpmn:outgoing>
  <bpmn:outgoing>Flow_Cancel</bpmn:outgoing>  <!-- ❌ Two outgoing! -->
</bpmn:userTask>

<bpmn:sequenceFlow id="Flow_Continue" sourceRef="Task_RequestDocuments" targetRef="Task_Next" />

<bpmn:sequenceFlow id="Flow_Cancel" sourceRef="Task_RequestDocuments" targetRef="End_Cancelled">
  <bpmn:conditionExpression>${cancelled}</bpmn:conditionExpression>  <!-- ❌ Condition on task outflow! -->
</bpmn:sequenceFlow>
```

**Issues:**
- Decision point is **invisible** (no gateway on diagram)
- Violates "one outgoing flow per task" rule
- Hard to understand and maintain

**CORRECT:**
```xml
<bpmn:userTask id="Task_RequestDocuments">
  <bpmn:outgoing>Flow_ToGateway</bpmn:outgoing>  <!-- ✅ Single outgoing -->
</bpmn:userTask>

<bpmn:exclusiveGateway id="Gateway_CancelDecision" name="Cancel Process?"
  default="Flow_Continue">
  <bpmn:incoming>Flow_ToGateway</bpmn:incoming>
  <bpmn:outgoing>Flow_Continue</bpmn:outgoing>
  <bpmn:outgoing>Flow_Cancel</bpmn:outgoing>
</bpmn:exclusiveGateway>
```

**Rule:** Tasks must have **EXACTLY ONE** outgoing sequence flow. All decisions must use explicit gateways.

---

## Anti-Pattern #13: Logically Incomplete Loop ❌

**Problem:** Loop pattern without clear state change between iterations.

**WRONG:**
```
Gateway "Data Valid?" → No → Task "Get More Data" → Gateway "Data Valid?"
```
**Issues:**
- ❌ **Who updates the data?** (no actor/system action)
- ❌ **How does state change?** (magic transition)
- ❌ **What triggers re-evaluation?** (implicit)

**CORRECT:**
```
Gateway "Data Valid?" (Actor A)
  ↓ No
Task "Get More Data" (Actor B)
  ↓
Task "Update Record with New Data" (Actor B)  <!-- ✅ Explicit state change -->
  ↓
Gateway "Data Valid?" (Actor A - re-evaluates)
```

### Loop Validation Checklist

For EVERY loop pattern, verify:
1. ✅ **State Change**: Clear action that modifies the condition
2. ✅ **Actor Transition**: If different actors, explicit handoff (resubmit, notify)
3. ✅ **Exit Condition**: Loop can terminate (max attempts, timeout, approval)

**Example from Real Process:**
```
❌ WRONG:
  Gateway "Clear?" (Committee) → No → Task "Get Info" (Requester) → Gateway "Clear?"
  Problem: No explicit resubmission!

✅ CORRECT:
  Gateway "Clear?" (Committee)
    → No → Task "Get Info" (Requester)
    → Task "Update and Resubmit Notification" (Requester)  <!-- ✅ Explicit state change -->
    → Gateway "Clear?" (Committee)
```

**CRITICAL:** If you detect logically incomplete loop, **NOTIFY USER** (do NOT auto-fix).

---

## Anti-Pattern #14: Unbalanced Parallel Paths ❌

**Problem:** Parallel split has different number of paths than join.

**WRONG:**
```xml
<bpmn:parallelGateway id="Gateway_Fork">
  <bpmn:outgoing>Flow_A</bpmn:outgoing>
  <bpmn:outgoing>Flow_B</bpmn:outgoing>
  <bpmn:outgoing>Flow_C</bpmn:outgoing>  <!-- 3 paths out -->
</bpmn:parallelGateway>

<bpmn:parallelGateway id="Gateway_Join">
  <bpmn:incoming>Flow_A</bpmn:incoming>
  <bpmn:incoming>Flow_B</bpmn:incoming>  <!-- Only 2 paths in! -->
  <!-- ❌ Missing Flow_C! Join waits forever for 3rd token → DEADLOCK -->
</bpmn:parallelGateway>
```

**CORRECT:**
```xml
<bpmn:parallelGateway id="Gateway_Fork">
  <bpmn:outgoing>Flow_A</bpmn:outgoing>
  <bpmn:outgoing>Flow_B</bpmn:outgoing>
  <bpmn:outgoing>Flow_C</bpmn:outgoing>
</bpmn:parallelGateway>

<bpmn:parallelGateway id="Gateway_Join">
  <bpmn:incoming>Flow_A</bpmn:incoming>
  <bpmn:incoming>Flow_B</bpmn:incoming>
  <bpmn:incoming>Flow_C</bpmn:incoming>  <!-- ✅ All 3 paths -->
  <bpmn:outgoing>Flow_Continue</bpmn:outgoing>
</bpmn:parallelGateway>
```

**Exception (Acceptable if Documented):**
```xml
<!-- Path C terminates independently (e.g., sends notification only) -->
<bpmn:parallelGateway id="Gateway_Fork">
  <bpmn:outgoing>Flow_A</bpmn:outgoing>
  <bpmn:outgoing>Flow_B</bpmn:outgoing>
  <bpmn:outgoing>Flow_Notify</bpmn:outgoing>  <!-- Independent path -->
</bpmn:parallelGateway>

<!-- Notification path ends early (doesn't need sync) -->
<bpmn:sequenceFlow sourceRef="Task_NotifyAsync" targetRef="End_NotificationSent" />

<!-- Main paths sync -->
<bpmn:parallelGateway id="Gateway_Join">
  <bpmn:incoming>Flow_A</bpmn:incoming>
  <bpmn:incoming>Flow_B</bpmn:incoming>  <!-- Only 2 paths sync -->
</bpmn:parallelGateway>
```

**Rule:** Asymmetric join is OK **ONLY IF** intentional and documented (e.g., fire-and-forget notifications).

---

## Anti-Pattern #15: Decision After Decision (No Task Between) ❌

**Problem:** Two gateways directly connected without intervening task.

**WRONG:**
```
Gateway_1 "Approved?" → Gateway_2 "Amount > 1000?" → ...
```

**Issues:**
- Unclear semantics (why two decisions?)
- Harder to understand flow
- May indicate design flaw

**CORRECT (Option A: Combine into Single Gateway):**
```
Gateway "Approved AND Amount > 1000?" → ...
```

**CORRECT (Option B: Add Clarifying Task):**
```
Gateway_1 "Approved?" → Task "Calculate Final Amount" → Gateway_2 "Amount > 1000?" → ...
```

**Rule:** If two gateways are sequential, add task between them OR combine into one gateway.

---

## Quick Reference: Anti-Pattern Detection

| Anti-Pattern | Detection Signal | Quick Fix |
|--------------|------------------|-----------|
| #1 Wrong Flows | Message Flow inside Pool | Change to Sequence Flow |
| #2 Missing Start/End | Process without Start/End Event | Add Start and End Events |
| #3 Redundant Names | Event named "Start" or "End" | Use business-meaningful names |
| #4 Inconsistent Naming | Mixed case/verb forms | Apply naming convention consistently |
| #5 Overcomplicated | > 30 elements | Extract subprocesses |
| #6a No Default | XOR without `default` attribute | Add default flow |
| #6b Overlapping Conditions | Multiple XOR conditions can be true | Make conditions mutually exclusive |
| #6c Incomplete Conditions | XOR doesn't cover all values | Add missing conditions or default |
| #6d Unbalanced Types | Parallel Split → XOR Join | Match gateway types (Parallel → Parallel) |
| #7 Gateway Messages | Gateway → Message Event | Insert task before message |
| #8 OR Token Issues | End Event between OR-split/join | Route all paths to join |
| #9 Multi-Merge | Task with multiple incoming | Insert merge gateway |
| #10 No Error Handling | Critical task without boundary event | Add error/timer boundary events |
| #11 Redundant Ends | Multiple ends with same meaning | Merge paths before single End |
| #12 Implicit Gateway | Task with multiple outgoing + conditions | Add explicit gateway after task |
| #13 Incomplete Loop | Loop without state change | Add state-changing task |
| #14 Unbalanced Parallel | Fork paths ≠ Join paths | Match path counts or document asymmetry |
| #15 Decision Chain | Gateway → Gateway | Add task between OR combine gateways |

---

**Back to:** `SKILL.md` Anti-Patterns section

**See Also:**
- `multi-merge-prevention.md` - Detailed algorithm for #9
- `gateway-rules-and-antipatterns.md` - Comprehensive gateway guide for #6
- `examples-guide.md` - Correct pattern examples
