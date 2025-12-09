# BPMN Structural Patterns & Examples Guide

**Reference for:** `SKILL.md` Structural Patterns section

**Purpose:** Detailed catalog of common BPMN process patterns with complete examples and use cases.

## Pattern Catalog

### Pattern 1: Simple Approval

**Use Case:** Basic approval workflow with binary decision (approve/reject).

**When to Use:**
- Single decision maker
- Binary outcome (yes/no)
- No parallel processing needed
- Simple linear flow

**Structure:**
```
Start → Submit Request → Review Request → [Approved?]
                                            ├─ Yes → Process Request → Notify Success → End(Approved)
                                            └─ No → Notify Rejection → End(Rejected)
```

**XML Example:**
```xml
<bpmn:startEvent id="Event_Start" name="Request Received" />

<bpmn:userTask id="Task_Submit" name="Submit Request" />

<bpmn:userTask id="Task_Review" name="Review Request" />

<bpmn:exclusiveGateway id="Gateway_Approved" name="Approved?" default="Flow_No" />

<bpmn:userTask id="Task_Process" name="Process Request" />

<bpmn:serviceTask id="Task_NotifySuccess" name="Notify Success" />

<bpmn:endEvent id="End_Approved" name="Request Approved" />

<bpmn:serviceTask id="Task_NotifyReject" name="Notify Rejection" />

<bpmn:endEvent id="End_Rejected" name="Request Rejected" />
```

**Key Elements:**
- 1 Start Event
- 2-3 User Tasks (submit, review, optional process)
- 1 Exclusive Gateway (decision point)
- 2 End Events (approved vs rejected outcomes)
- Service Tasks for notifications

**Common Variations:**
- Add timer boundary event on Review Task (SLA escalation)
- Add error boundary event on Process Task (handle failures)
- Add conditional notification based on request type

---

### Pattern 2: Multi-Level Approval

**Use Case:** Hierarchical approval requiring multiple decision points based on thresholds.

**When to Use:**
- Multiple decision makers (manager, director, etc.)
- Approval requirements based on value/risk
- Escalation based on conditions

**Structure:**
```
Start → Submit → Manager Review → [Approved?]
                                    ├─ No → End(Rejected)
                                    └─ Yes → [Amount > Threshold?]
                                              ├─ No → Process → End(Approved)
                                              └─ Yes → Director Review → [Approved?]
                                                                          ├─ Yes → Process → End(Approved)
                                                                          └─ No → End(Rejected)
```

**Key Elements:**
- Cascading decision points
- Threshold-based routing
- Multiple approval levels
- Early rejection paths

**Common Variations:**
- Add 3rd level (CEO approval) for very high amounts
- Add automatic approval for very low amounts (skip reviews)
- Add notification to requester at each stage

---

### Pattern 3: Parallel Processing with Synchronization

**Use Case:** Multiple independent checks/tasks executed simultaneously, followed by synchronization.

**When to Use:**
- Tasks can be done in parallel (no dependencies)
- All tasks must complete before continuing
- Time-sensitive process (parallel = faster)

**Structure:**
```
Start → Receive Order → ║Parallel Split║ → Check Inventory ──────┐
                                          → Validate Payment ──────┼→ ║Parallel Join║ → [All OK?] → Ship → End
                                          → Verify Address ────────┘                        │
                                                                                            └─ No → Cancel → End
```

**XML Example:**
```xml
<!-- Parallel Split -->
<bpmn:parallelGateway id="Gateway_Split" name="Start Background Checks">
  <bpmn:incoming>Flow_FromOrder</bpmn:incoming>
  <bpmn:outgoing>Flow_ToInventory</bpmn:outgoing>
  <bpmn:outgoing>Flow_ToPayment</bpmn:outgoing>
  <bpmn:outgoing>Flow_ToAddress</bpmn:outgoing>
</bpmn:parallelGateway>

<!-- Parallel tasks -->
<bpmn:serviceTask id="Task_Inventory" name="Check Inventory" />
<bpmn:serviceTask id="Task_Payment" name="Validate Payment" />
<bpmn:serviceTask id="Task_Address" name="Verify Address" />

<!-- Parallel Join (Synchronization) -->
<bpmn:parallelGateway id="Gateway_Join" name="Synchronize Checks">
  <bpmn:incoming>Flow_InventoryDone</bpmn:incoming>
  <bpmn:incoming>Flow_PaymentDone</bpmn:incoming>
  <bpmn:incoming>Flow_AddressDone</bpmn:incoming>
  <bpmn:outgoing>Flow_ToValidation</bpmn:outgoing>
</bpmn:parallelGateway>

<!-- Validation after sync -->
<bpmn:exclusiveGateway id="Gateway_AllOK" name="All Checks OK?" default="Flow_Cancel" />
```

**CRITICAL Rules:**
- **ALWAYS** use matching Parallel Join after Parallel Split
- **SAME NUMBER** of paths in and out (3 out → 3 in)
- **NO conditions** on flows from Parallel Split (all paths always execute)
- **NO End events** between split and join (will cause deadlock)

**Common Variations:**
- Add timeout on parallel tasks (if one takes too long, escalate)
- Add error handling on each parallel task (use boundary events)
- Asymmetric join (one path terminates early with notification, others continue)

---

### Pattern 4: Timeout/Escalation

**Use Case:** Wait for external event with timeout fallback (e.g., waiting for customer response).

**When to Use:**
- Process waits for external input
- Need fallback if response not received in time
- SLA requirements for response times

**Structure:**
```
... → Send Request → ◇Event Gateway◇ → ○ Response Received → Process Response → ...
                                      → ⏱ 48h Timeout → Escalate to Manager → ...
```

**XML Example:**
```xml
<!-- Send request to external party -->
<bpmn:sendTask id="Task_SendRequest" name="Send Request to Customer" />

<!-- Event-Based Gateway (waits for external event) -->
<bpmn:eventBasedGateway id="Gateway_Event" name="Wait for Response or Timeout">
  <bpmn:incoming>Flow_FromSend</bpmn:incoming>
  <bpmn:outgoing>Flow_ToResponse</bpmn:outgoing>
  <bpmn:outgoing>Flow_ToTimeout</bpmn:outgoing>
</bpmn:eventBasedGateway>

<!-- Catch events -->
<bpmn:intermediateCatchEvent id="Event_Response" name="Response Received">
  <bpmn:incoming>Flow_ToResponse</bpmn:incoming>
  <bpmn:outgoing>Flow_ProcessResponse</bpmn:outgoing>
  <bpmn:messageEventDefinition messageRef="Message_CustomerResponse" />
</bpmn:intermediateCatchEvent>

<bpmn:intermediateCatchEvent id="Event_Timeout" name="48h Timeout">
  <bpmn:incoming>Flow_ToTimeout</bpmn:incoming>
  <bpmn:outgoing>Flow_Escalate</bpmn:outgoing>
  <bpmn:timerEventDefinition>
    <bpmn:timeDuration>P2D</bpmn:timeDuration>
  </bpmn:timerEventDefinition>
</bpmn:intermediateCatchEvent>

<!-- Follow-up actions -->
<bpmn:userTask id="Task_ProcessResponse" name="Process Response" />
<bpmn:userTask id="Task_Escalate" name="Escalate to Manager" />
```

**Key Elements:**
- Event-Based Gateway (waits for one of multiple events)
- Intermediate Catch Events (message, timer, signal)
- Mutually exclusive paths (only one event fires)

**Common Variations:**
- Multiple timeout stages (reminder at 24h, escalation at 48h, cancellation at 72h)
- Cancellation event (customer can cancel request)
- Multiple response types (approval, rejection, request for info)

---

### Pattern 5: Loop with Exit Condition

**Use Case:** Iterative refinement process with retry limit.

**When to Use:**
- Task may need multiple attempts (e.g., document revisions)
- Exit condition exists (approval or max attempts)
- State changes between iterations

**Structure:**
```
Start → Prepare Document → Review → [Approved?]
                             ↑         ├─ Yes → Finalize → End(Success)
                             │         └─ No → [Attempts < 3?]
                             │                   ├─ Yes → Revise ─┘
                             │                   └─ No → End(Failed)
                             └─────────────────────────────┘
```

**CRITICAL Rules for Loops:**
1. **State Change Required:** Something MUST change between iterations
   - ❌ BAD: `Review → [Not OK?] → Get Info → Review` (no state change!)
   - ✅ GOOD: `Review → [Not OK?] → Get Info → Update Document → Review`

2. **Exit Condition Required:** Loop must be able to terminate
   - Max attempts counter
   - Approval/rejection decision
   - Timeout

3. **Actor Transition:** If different actors, make handoff explicit
   - ❌ BAD: `Committee Reviews → Get Info (Requester) → Committee Reviews` (implicit resubmission)
   - ✅ GOOD: `Committee Reviews → Get Info → Resubmit Notification → Committee Reviews`

**XML Example:**
```xml
<bpmn:userTask id="Task_Prepare" name="Prepare Document" />

<bpmn:userTask id="Task_Review" name="Review Document" />

<bpmn:exclusiveGateway id="Gateway_Approved" name="Approved?" default="Flow_NotApproved" />

<bpmn:userTask id="Task_Finalize" name="Finalize Document" />
<bpmn:endEvent id="End_Success" name="Document Approved" />

<!-- Not approved path -->
<bpmn:exclusiveGateway id="Gateway_AttemptsCheck" name="Attempts < 3?" default="Flow_Failed" />

<bpmn:userTask id="Task_Revise" name="Revise Document" />
<bpmn:scriptTask id="Task_IncrementCounter" name="Increment Attempt Counter">
  <bpmn:script>${execution.setVariable("attempts", attempts + 1)}</bpmn:script>
</bpmn:scriptTask>

<bpmn:endEvent id="End_Failed" name="Max Attempts Reached" />

<!-- Loop back -->
<bpmn:sequenceFlow sourceRef="Task_IncrementCounter" targetRef="Task_Review" />
```

**Common Variations:**
- Add notification on each iteration (inform requester of feedback)
- Add escalation after 2 attempts (involve senior reviewer)
- Add timeout for entire loop (max 7 days for all iterations)

---

### Pattern 6: Subprocess for Reusable Logic

**Use Case:** Extract complex or reusable logic into a subprocess.

**When to Use:**
- Process section is complex (> 10 elements)
- Logic is reused in multiple places
- Want to hide implementation details
- Separate concerns (main flow vs detail)

**Structure:**
```
Main Process:
  Start → Collect Data → [Payment Subprocess] → Ship Order → End

Payment Subprocess:
  Start → Calculate Total → Process Payment → [Success?]
                                              ├─ Yes → End(Success)
                                              └─ No → Retry Logic → [Attempts?]
                                                                    ├─ < 3 → Process Payment (loop)
                                                                    └─ ≥ 3 → End(Failed)
```

**Two Types of Subprocesses:**

#### A. Embedded Subprocess (Part of Main Process)
```xml
<bpmn:subProcess id="Subprocess_Payment" name="Payment Processing">
  <!-- Subprocess elements here -->
  <bpmn:startEvent id="Event_SubStart" />
  <bpmn:serviceTask id="Task_CalculateTotal" name="Calculate Total" />
  <bpmn:serviceTask id="Task_ProcessPayment" name="Process Payment" />
  <bpmn:endEvent id="Event_SubEnd" />
</bpmn:subProcess>
```

**Use When:** Logic is specific to this process, not reused elsewhere.

#### B. Call Activity (References External Process)
```xml
<!-- Main process -->
<bpmn:callActivity id="Activity_CallPayment" name="Payment Processing"
                   calledElement="Process_PaymentStandard">
  <bpmn:incoming>Flow_ToPayment</bpmn:incoming>
  <bpmn:outgoing>Flow_PaymentDone</bpmn:outgoing>
</bpmn:callActivity>

<!-- Separate process definition (Process_PaymentStandard) -->
<bpmn:process id="Process_PaymentStandard" isExecutable="true">
  <bpmn:startEvent id="Event_Start" />
  <bpmn:serviceTask id="Task_Calculate" name="Calculate Total" />
  <!-- ... -->
</bpmn:process>
```

**Use When:** Logic is reused across multiple processes (e.g., standard payment flow).

**Common Variations:**
- Transaction subprocess (with compensation)
- Event subprocess (triggered by error or message)
- Multi-instance subprocess (process collection of items)

---

### Pattern 7: Inclusive Gateway (OR-Split)

**Use Case:** Multiple optional paths, where one or more can be active simultaneously.

**When to Use:**
- Multiple conditions can be true at the same time
- Need to activate 1, 2, or all paths depending on data
- Example: Document needs legal review (if contract), financial review (if > $10k), technical review (if new technology)

**Structure:**
```
Start → Prepare Request → ◊Inclusive Split◊ → Legal Review ───────┐
                                             → Financial Review ───┼→ ◊Inclusive Join◊ → Finalize → End
                                             → Technical Review ───┘
```

**XML Example:**
```xml
<!-- Inclusive Split -->
<bpmn:inclusiveGateway id="Gateway_ReviewTypes"
  name="Which Reviews Needed?"
  default="Flow_Technical">
  <bpmn:incoming>Flow_FromPrepare</bpmn:incoming>
  <bpmn:outgoing>Flow_Legal</bpmn:outgoing>
  <bpmn:outgoing>Flow_Financial</bpmn:outgoing>
  <bpmn:outgoing>Flow_Technical</bpmn:outgoing>
</bpmn:inclusiveGateway>

<!-- Conditional flows -->
<bpmn:sequenceFlow id="Flow_Legal" name="Contract Involved"
  sourceRef="Gateway_ReviewTypes" targetRef="Task_LegalReview">
  <bpmn:conditionExpression>${hasContract}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<bpmn:sequenceFlow id="Flow_Financial" name="Amount > $10k"
  sourceRef="Gateway_ReviewTypes" targetRef="Task_FinancialReview">
  <bpmn:conditionExpression>${amount > 10000}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<!-- Default flow (ensures at least one path) -->
<bpmn:sequenceFlow id="Flow_Technical" name="New Technology"
  sourceRef="Gateway_ReviewTypes" targetRef="Task_TechnicalReview" />

<!-- Review tasks -->
<bpmn:userTask id="Task_LegalReview" name="Legal Review" />
<bpmn:userTask id="Task_FinancialReview" name="Financial Review" />
<bpmn:userTask id="Task_TechnicalReview" name="Technical Review" />

<!-- Inclusive Join (waits for ALL activated paths) -->
<bpmn:inclusiveGateway id="Gateway_ReviewJoin" name="All Reviews Complete">
  <bpmn:incoming>Flow_LegalDone</bpmn:incoming>
  <bpmn:incoming>Flow_FinancialDone</bpmn:incoming>
  <bpmn:incoming>Flow_TechnicalDone</bpmn:incoming>
  <bpmn:outgoing>Flow_ToFinalize</bpmn:outgoing>
</bpmn:inclusiveGateway>
```

**CRITICAL Rules:**
- **MUST have default flow** (or ensure at least one condition is always true)
- **NO End events** between OR-split and OR-join (causes deadlock!)
- **Join waits for ALL activated paths** (not just first one)

**Common Mistake:**
```xml
<!-- ❌ WRONG: Path terminates early -->
<bpmn:inclusiveGateway id="Gateway_Split" />
  → Task_A → End_PathA  <!-- ❌ Token terminated! -->
  → Task_B → Gateway_Join  <!-- Waits forever for Path A -->
```

**Correct:**
```xml
<!-- ✅ CORRECT: All paths go to join -->
<bpmn:inclusiveGateway id="Gateway_Split" />
  → Task_A → Gateway_Join
  → Task_B → Gateway_Join
```

---

### Pattern 8: Error Handling with Boundary Events

**Use Case:** Handle exceptions and errors during task execution.

**When to Use:**
- Task can fail (payment, API call, validation)
- Need to handle error gracefully (retry, compensate, notify)
- Don't want process to crash

**Structure:**
```
... → Process Payment → Continue Success Flow → ...
        ⚡ Error: Payment Failed → Handle Error → Notify Customer → End(Error)
```

**XML Example:**
```xml
<!-- Task that can fail -->
<bpmn:serviceTask id="Task_ProcessPayment" name="Process Payment" />

<!-- Boundary Error Event (attached to task) -->
<bpmn:boundaryEvent id="Error_PaymentFailed"
  name="Payment Failed"
  attachedToRef="Task_ProcessPayment">
  <bpmn:outgoing>Flow_ToErrorHandler</bpmn:outgoing>
  <bpmn:errorEventDefinition errorRef="Error_Payment" />
</bpmn:boundaryEvent>

<!-- Error definition -->
<bpmn:error id="Error_Payment" name="PaymentError" errorCode="PAYMENT_ERROR" />

<!-- Error handler task -->
<bpmn:userTask id="Task_HandleError" name="Handle Payment Error" />
<bpmn:serviceTask id="Task_NotifyCustomer" name="Notify Customer of Error" />
<bpmn:endEvent id="End_PaymentError" name="Payment Failed" />
```

**Types of Boundary Events:**
1. **Error Event** - Interrupting (task is cancelled when error occurs)
2. **Timer Event** - Can be interrupting (timeout cancels task) or non-interrupting (escalation without cancelling)
3. **Message Event** - External signal to interrupt/escalate
4. **Escalation Event** - Escalate to higher level without interrupting task

**Common Variations:**
- Retry pattern (error → wait 5 minutes → retry task)
- Compensation pattern (error → undo previous tasks)
- Escalation pattern (timer → notify manager, but don't cancel task)

---

### Pattern 9: Multi-Instance Tasks

**Use Case:** Execute same task multiple times for collection of items (e.g., review 10 documents, approve 5 purchase orders).

**When to Use:**
- Process collection of items
- Each item needs same processing
- Can be sequential or parallel

**Two Modes:**

#### A. Sequential Multi-Instance
```xml
<!-- Process items one at a time -->
<bpmn:userTask id="Task_ReviewDoc" name="Review Document">
  <bpmn:multiInstanceLoopCharacteristics isSequential="true">
    <bpmn:loopCardinality>#{documents.size()}</bpmn:loopCardinality>
    <bpmn:inputDataItem>${document}</bpmn:inputDataItem>
  </bpmn:multiInstanceLoopCharacteristics>
</bpmn:userTask>
```

**Use When:** Items must be processed in order (e.g., sequential approvals).

#### B. Parallel Multi-Instance
```xml
<!-- Process all items simultaneously -->
<bpmn:userTask id="Task_ReviewDoc" name="Review Document">
  <bpmn:multiInstanceLoopCharacteristics isSequential="false">
    <bpmn:loopCardinality>#{documents.size()}</bpmn:loopCardinality>
    <bpmn:inputDataItem>${document}</bpmn:inputDataItem>
    <bpmn:completionCondition>#{completedInstances >= documents.size()}</bpmn:completionCondition>
  </bpmn:multiInstanceLoopCharacteristics>
</bpmn:userTask>
```

**Use When:** Items can be processed independently (e.g., parallel background checks).

**Visual Notation:** Task with **|||** marker at bottom indicates multi-instance.

---

### Pattern 10: Event-Driven Process

**Use Case:** Process triggered by messages from external systems, with correlation.

**When to Use:**
- Process waits for external signals
- Multiple messages expected throughout process
- Message correlation needed (which process instance gets which message)

**Structure:**
```
○ Message Start: Order Received → Acknowledge Order → Wait for Payment → Process Order → End
                                                        ↑
                                                  ○ Message: Payment Received
```

**XML Example:**
```xml
<!-- Message Start Event -->
<bpmn:startEvent id="Event_OrderReceived" name="Order Received">
  <bpmn:messageEventDefinition messageRef="Message_NewOrder" />
</bpmn:startEvent>

<!-- Message definition -->
<bpmn:message id="Message_NewOrder" name="New Order" />

<bpmn:userTask id="Task_AcknowledgeOrder" name="Acknowledge Order" />

<!-- Intermediate Message Catch Event -->
<bpmn:intermediateCatchEvent id="Event_PaymentReceived" name="Payment Received">
  <bpmn:incoming>Flow_WaitForPayment</bpmn:incoming>
  <bpmn:outgoing>Flow_ProcessOrder</bpmn:outgoing>
  <bpmn:messageEventDefinition messageRef="Message_Payment" />
</bpmn:intermediateCatchEvent>

<bpmn:message id="Message_Payment" name="Payment Received" />
```

**Message Correlation:**
```xml
<!-- Camunda-specific message correlation -->
<bpmn:startEvent id="Event_Start">
  <bpmn:messageEventDefinition messageRef="Message_NewOrder"
    camunda:messageCorrelationKey="${orderId}" />
</bpmn:startEvent>
```

**Common Variations:**
- Multiple message events (order → payment → shipment)
- Message + timer (wait for payment OR timeout)
- Signal events (broadcast to multiple processes)

---

## Pattern Selection Guide

| Requirement | Recommended Pattern |
|-------------|---------------------|
| Simple yes/no decision | Pattern 1: Simple Approval |
| Multiple approval levels | Pattern 2: Multi-Level Approval |
| Parallel independent tasks | Pattern 3: Parallel Processing |
| Wait for external response | Pattern 4: Timeout/Escalation |
| Retry with limit | Pattern 5: Loop with Exit Condition |
| Complex reusable logic | Pattern 6: Subprocess |
| Multiple optional checks | Pattern 7: Inclusive Gateway |
| Handle task failures | Pattern 8: Error Handling |
| Process list of items | Pattern 9: Multi-Instance |
| Message-driven flow | Pattern 10: Event-Driven |

---

## Combining Patterns

Real-world processes often combine multiple patterns:

**Example: E-commerce Order Processing**
```
Start (Message)
  → Validate Order (Pattern 1: Simple validation)
  → ║Parallel Split║ (Pattern 3)
      → Check Inventory
      → Validate Payment (with error boundary - Pattern 8)
      → Verify Shipping Address
  → ║Parallel Join║
  → [All OK?]
      ├─ Yes → Process Order (Pattern 6: Subprocess)
      └─ No → Notify Customer → End
```

This combines:
- Message start event (Pattern 10)
- Parallel processing (Pattern 3)
- Error handling (Pattern 8)
- Subprocess (Pattern 6)
- Simple decision (Pattern 1)

---

**Back to:** `SKILL.md` Structural Patterns section

**See Also:**
- `advanced-patterns.md` - Complex patterns (compensation, escalation, transactions)
- `antipatterns-full.md` - What NOT to do
- `xml-templates.md` - XML syntax for all elements
