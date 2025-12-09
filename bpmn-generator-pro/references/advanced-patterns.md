# Advanced BPMN Patterns

Complex patterns for real-world business processes.

## 1. Multi-Instance Patterns

### Sequential Multi-Instance (Process items one by one)
```xml
<bpmn:userTask id="Task_ReviewItem" name="Review Item">
  <bpmn:multiInstanceLoopCharacteristics isSequential="true"
      camunda:collection="${items}" camunda:elementVariable="item">
    <bpmn:completionCondition>${nrOfCompletedInstances >= 1 &amp;&amp; rejected}</bpmn:completionCondition>
  </bpmn:multiInstanceLoopCharacteristics>
</bpmn:userTask>
```

### Parallel Multi-Instance (Process all items simultaneously)
```xml
<bpmn:serviceTask id="Task_NotifyAll" name="Notify All Stakeholders">
  <bpmn:multiInstanceLoopCharacteristics isSequential="false"
      camunda:collection="${stakeholders}" camunda:elementVariable="stakeholder">
  </bpmn:multiInstanceLoopCharacteristics>
</bpmn:serviceTask>
```

## 2. Approval Chain Patterns

### Sequential Approval (must pass each level)
```
Start → Submit → [Level 1 Review] → Approved? 
                                      ├─ No → Rejected End
                                      └─ Yes → [Level 2 Review] → Approved?
                                                                    ├─ No → Rejected End  
                                                                    └─ Yes → Approved End
```

### Parallel Approval (all must approve)
```
Start → Submit → ║Fork║ → Manager Review ──────────┐
                        → Legal Review ─────────────┼→ ║Join║ → All Approved? → ...
                        → Finance Review ───────────┘
```

### First-Response Wins (race condition)
```
Start → Request → ◇Event Gateway◇ → ○ Manager Approved → Process
                                   → ○ Manager Rejected → Cancel
                                   → ⏱ 48h Timeout → Escalate
```

### Quorum Approval (N of M must approve)
Use multi-instance with completion condition:
```xml
<bpmn:userTask id="Task_Vote" name="Cast Vote">
  <bpmn:multiInstanceLoopCharacteristics isSequential="false"
      camunda:collection="${voters}" camunda:elementVariable="voter">
    <bpmn:completionCondition>
      ${nrOfCompletedInstances/nrOfInstances >= 0.5}  <!-- 50% quorum -->
    </bpmn:completionCondition>
  </bpmn:multiInstanceLoopCharacteristics>
</bpmn:userTask>
```

## 3. Error Handling Patterns

### Retry with Backoff
```
Task → [Error Boundary] → Retry Counter < 3?
                            ├─ Yes → Wait (exponential) → Task ↻
                            └─ No → Escalate/Fail
```

```xml
<bpmn:serviceTask id="Task_CallAPI" name="Call External API">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Success</bpmn:outgoing>
</bpmn:serviceTask>

<bpmn:boundaryEvent id="Boundary_APIError" attachedToRef="Task_CallAPI">
  <bpmn:errorEventDefinition errorRef="Error_APIFailure"/>
  <bpmn:outgoing>Flow_ToRetryLogic</bpmn:outgoing>
</bpmn:boundaryEvent>

<bpmn:exclusiveGateway id="Gateway_RetryCount" name="Retries < 3?">
  <bpmn:incoming>Flow_ToRetryLogic</bpmn:incoming>
  <bpmn:outgoing>Flow_Retry</bpmn:outgoing>
  <bpmn:outgoing>Flow_GiveUp</bpmn:outgoing>
</bpmn:exclusiveGateway>
```

### Compensation (Rollback)
```xml
<bpmn:serviceTask id="Task_ChargeCard" name="Charge Credit Card">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>
</bpmn:serviceTask>

<bpmn:boundaryEvent id="Compensate_ChargeCard" attachedToRef="Task_ChargeCard">
  <bpmn:compensateEventDefinition/>
</bpmn:boundaryEvent>

<bpmn:serviceTask id="Task_RefundCard" name="Refund Credit Card" isForCompensation="true">
</bpmn:serviceTask>

<bpmn:association id="Association_Compensate" 
    associationDirection="One" 
    sourceRef="Compensate_ChargeCard" 
    targetRef="Task_RefundCard"/>

<!-- Trigger compensation -->
<bpmn:intermediateThrowEvent id="Event_TriggerCompensation" name="Rollback Transaction">
  <bpmn:compensateEventDefinition/>
</bpmn:intermediateThrowEvent>
```

### Dead Letter Queue Pattern
```
Main Process: Task → [Error] → Log to DLQ → Continue/End

DLQ Process: Timer Start (poll DLQ) → Process Failed Items → Notify Admin
```

## 4. Timeout & Escalation Patterns

### SLA Timer (Non-Interrupting)
```xml
<bpmn:userTask id="Task_Review" name="Review Document">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>
</bpmn:userTask>

<!-- Warning at 80% of SLA -->
<bpmn:boundaryEvent id="Timer_Warning" attachedToRef="Task_Review" cancelActivity="false">
  <bpmn:timerEventDefinition>
    <bpmn:timeDuration>PT38H24M</bpmn:timeDuration>  <!-- 80% of 48h -->
  </bpmn:timerEventDefinition>
  <bpmn:outgoing>Flow_SendWarning</bpmn:outgoing>
</bpmn:boundaryEvent>

<!-- Breach at 100% of SLA -->
<bpmn:boundaryEvent id="Timer_Breach" attachedToRef="Task_Review" cancelActivity="false">
  <bpmn:timerEventDefinition>
    <bpmn:timeDuration>PT48H</bpmn:timeDuration>
  </bpmn:timerEventDefinition>
  <bpmn:outgoing>Flow_Escalate</bpmn:outgoing>
</bpmn:boundaryEvent>
```

### Progressive Escalation
```
Task_Pending → [24h] → Remind Assignee → [24h] → Notify Manager → [24h] → Auto-Reassign
```

### Hard Deadline (Interrupting)
```xml
<bpmn:boundaryEvent id="Timer_Deadline" attachedToRef="Task_Approval" cancelActivity="true">
  <bpmn:timerEventDefinition>
    <bpmn:timeDate>2024-12-31T23:59:59Z</bpmn:timeDate>
  </bpmn:timerEventDefinition>
  <bpmn:outgoing>Flow_DeadlineMissed</bpmn:outgoing>
</bpmn:boundaryEvent>
```

## 5. Message Correlation Patterns

### Request-Response
```
Process A: Send Request → ◇Event Gateway◇ → ○ Response Received → Continue
                                           → ⏱ Timeout → Handle Timeout

Process B: ○ Request Received → Process → Send Response
```

```xml
<!-- Process A -->
<bpmn:sendTask id="Task_SendRequest" name="Send Request">
  <bpmn:extensionElements>
    <camunda:connector>
      <camunda:connectorId>http-connector</camunda:connectorId>
    </camunda:connector>
  </bpmn:extensionElements>
</bpmn:sendTask>

<bpmn:intermediateCatchEvent id="Event_AwaitResponse" name="Response Received">
  <bpmn:messageEventDefinition messageRef="Message_Response">
    <bpmn:correlationKey>${correlationId}</bpmn:correlationKey>
  </bpmn:messageEventDefinition>
</bpmn:intermediateCatchEvent>
```

### Publish-Subscribe
```xml
<!-- Publisher -->
<bpmn:intermediateThrowEvent id="Event_PublishUpdate" name="Publish Status Update">
  <bpmn:signalEventDefinition signalRef="Signal_StatusUpdate"/>
</bpmn:intermediateThrowEvent>

<!-- Subscriber (separate process) -->
<bpmn:startEvent id="Start_OnStatusUpdate" name="Status Updated">
  <bpmn:signalEventDefinition signalRef="Signal_StatusUpdate"/>
</bpmn:startEvent>
```

## 6. Subprocess Patterns

### Event Subprocess (Exception Handler)
```xml
<bpmn:subProcess id="SubProcess_Main" name="Main Process">
  <!-- Main flow elements -->
  
  <!-- Event subprocess - triggered by error anywhere in parent -->
  <bpmn:subProcess id="SubProcess_ErrorHandler" triggeredByEvent="true">
    <bpmn:startEvent id="Start_OnError">
      <bpmn:errorEventDefinition errorRef="Error_Generic"/>
    </bpmn:startEvent>
    <bpmn:serviceTask id="Task_LogError" name="Log Error"/>
    <bpmn:serviceTask id="Task_NotifyAdmin" name="Notify Admin"/>
    <bpmn:endEvent id="End_ErrorHandled"/>
  </bpmn:subProcess>
</bpmn:subProcess>
```

### Transaction Subprocess
```xml
<bpmn:transaction id="Transaction_Payment" name="Payment Transaction">
  <bpmn:startEvent id="TxStart"/>
  <bpmn:serviceTask id="Task_Reserve" name="Reserve Inventory"/>
  <bpmn:serviceTask id="Task_Charge" name="Charge Payment"/>
  <bpmn:serviceTask id="Task_Confirm" name="Confirm Order"/>
  <bpmn:endEvent id="TxEnd"/>
  
  <!-- Cancel boundary triggers compensation -->
  <bpmn:boundaryEvent id="TxCancel" attachedToRef="Transaction_Payment">
    <bpmn:cancelEventDefinition/>
  </bpmn:boundaryEvent>
</bpmn:transaction>
```

## 7. State Machine Pattern

Model explicit states using exclusive gateways:

```
Start → Determine State → [State = Draft?] → Edit Draft → Determine State ↻
                        → [State = Submitted?] → Review → Determine State ↻
                        → [State = Approved?] → Process → End(Complete)
                        → [State = Rejected?] → Notify → End(Rejected)
```

## 8. Saga Pattern (Distributed Transactions)

For microservices orchestration:

```
Start → Create Order → [Error?] → Compensate ← ← ← ← ← ← ↵
                         ↓ OK                            ↑
      → Reserve Inventory → [Error?] → Compensate ← ← ← ↵
                              ↓ OK                      ↑
      → Process Payment → [Error?] → Compensate ← ← ← ↵
                            ↓ OK
      → Confirm All → End(Success)
```

Each step has compensating action; errors trigger rollback chain.

## 9. Dynamic Routing

### Content-Based Router
```
Receive Message → Determine Type → [Type = A?] → Process A
                                 → [Type = B?] → Process B
                                 → [Default] → Process Generic
```

### Recipient List (Dynamic Parallel)
```xml
<bpmn:serviceTask id="Task_NotifyRecipients" name="Notify All Recipients">
  <bpmn:multiInstanceLoopCharacteristics isSequential="false"
      camunda:collection="${determineRecipients(messageType)}" 
      camunda:elementVariable="recipient">
  </bpmn:multiInstanceLoopCharacteristics>
</bpmn:serviceTask>
```

## 10. Complex Gateway Combinations

### Discriminator (First of Parallel)
Wait for first response, continue without waiting for others:

```xml
<bpmn:parallelGateway id="Gateway_RaceStart">
  <bpmn:outgoing>Flow_ToService1</bpmn:outgoing>
  <bpmn:outgoing>Flow_ToService2</bpmn:outgoing>
  <bpmn:outgoing>Flow_ToService3</bpmn:outgoing>
</bpmn:parallelGateway>

<!-- Use event-based gateway after services to catch first response -->
<bpmn:eventBasedGateway id="Gateway_FirstResponse" eventGatewayType="Parallel">
  <bpmn:incoming>Flow_FromService1</bpmn:incoming>
  <bpmn:incoming>Flow_FromService2</bpmn:incoming>
  <bpmn:incoming>Flow_FromService3</bpmn:incoming>
</bpmn:eventBasedGateway>
```

### Conditional Split with Default
Always include default flow for unexpected conditions:

```xml
<bpmn:exclusiveGateway id="Gateway_Priority" name="Priority Level?" default="Flow_Normal">
  <bpmn:outgoing>Flow_Critical</bpmn:outgoing>
  <bpmn:outgoing>Flow_High</bpmn:outgoing>
  <bpmn:outgoing>Flow_Normal</bpmn:outgoing>
</bpmn:exclusiveGateway>

<bpmn:sequenceFlow id="Flow_Critical" name="Critical" sourceRef="Gateway_Priority" targetRef="Task_ImmediateAction">
  <bpmn:conditionExpression xsi:type="bpmn:tFormalExpression">${priority == 'CRITICAL'}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<bpmn:sequenceFlow id="Flow_High" name="High" sourceRef="Gateway_Priority" targetRef="Task_PriorityQueue">
  <bpmn:conditionExpression xsi:type="bpmn:tFormalExpression">${priority == 'HIGH'}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<!-- Default flow - no condition expression needed -->
<bpmn:sequenceFlow id="Flow_Normal" name="Normal" sourceRef="Gateway_Priority" targetRef="Task_StandardQueue"/>
```

## Pattern Selection Guide

| Scenario | Pattern |
|----------|---------|
| Process list of items | Multi-Instance |
| All approvers must approve | Parallel Gateway + Join |
| Any approver can approve | Event-Based Gateway |
| Majority must approve | Multi-Instance + Completion Condition |
| Retry failed operation | Error Boundary + Loop |
| Rollback on failure | Compensation Events |
| SLA monitoring | Non-interrupting Timer Boundary |
| Hard deadline | Interrupting Timer Boundary |
| Cross-system communication | Message Events + Correlation |
| Reusable logic | Call Activity |
| Exception handling scope | Event Subprocess |
| ACID transactions | Transaction Subprocess |
| Microservices orchestration | Saga Pattern |
