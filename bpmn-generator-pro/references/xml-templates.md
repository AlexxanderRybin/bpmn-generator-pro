# XML Templates Reference

Complete XML snippets for all BPMN 2.0 elements.

## Document Structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<bpmn:definitions xmlns:bpmn="http://www.omg.org/spec/BPMN/20100524/MODEL"
                  xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI"
                  xmlns:dc="http://www.omg.org/spec/DD/20100524/DC"
                  xmlns:di="http://www.omg.org/spec/DD/20100524/DI"
                  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                  xmlns:camunda="http://camunda.org/schema/1.0/bpmn"
                  id="Definitions_[ProcessName]"
                  targetNamespace="http://bpmn.io/schema/bpmn"
                  exporter="BPMN Generator Pro"
                  exporterVersion="2.0">

  <!-- For multi-participant: wrap in collaboration -->
  <bpmn:collaboration id="Collaboration_1">
    <bpmn:participant id="Participant_Main" name="[Organization]" processRef="Process_Main"/>
  </bpmn:collaboration>

  <bpmn:process id="Process_Main" name="[Process Name]" isExecutable="true">
    <!-- Lanes (if needed) -->
    <bpmn:laneSet id="LaneSet_1">
      <bpmn:lane id="Lane_[Role]" name="[Role Name]">
        <bpmn:flowNodeRef>[element_ids]</bpmn:flowNodeRef>
      </bpmn:lane>
    </bpmn:laneSet>
    
    <!-- Elements -->
    <!-- Sequence Flows -->
  </bpmn:process>

  <bpmndi:BPMNDiagram id="BPMNDiagram_1">
    <bpmndi:BPMNPlane id="BPMNPlane_1" bpmnElement="[Collaboration_1 or Process_Main]">
      <!-- Shapes and Edges -->
    </bpmndi:BPMNPlane>
  </bpmndi:BPMNDiagram>
</bpmn:definitions>
```

---

## ⚠️ CRITICAL: BPMNDiagram Structure Rules

### Rule 1: Single Process (99% of cases)

When creating a **single-process diagram** (one organization, PMA approach, no message flows):

✅ **CORRECT Structure:**
```xml
<bpmn:process id="Process_Main" name="..." isExecutable="true">
  <!-- Process elements -->
</bpmn:process>

<bpmndi:BPMNDiagram id="BPMNDiagram_1">
  <bpmndi:BPMNPlane id="BPMNPlane_1" bpmnElement="Process_Main">
    <!-- Directly add element shapes - NO Pool shape -->
    <bpmndi:BPMNShape id="Event_Start_di" bpmnElement="Event_Start">
      <dc:Bounds x="..." y="..." width="36" height="36" />
    </bpmndi:BPMNShape>
    <!-- More shapes... -->
  </bpmndi:BPMNPlane>
</bpmndi:BPMNDiagram>
```

❌ **WRONG - Causes "multiple DI elements" error:**
```xml
<bpmndi:BPMNPlane bpmnElement="Process_Main">
  <!-- DO NOT ADD THIS: -->
  <bpmndi:BPMNShape id="Participant_Pool" bpmnElement="Process_Main">
    <dc:Bounds x="..." y="..." width="..." height="..." />
  </bpmndi:BPMNShape>
  <!-- ⬆️ This duplicates the BPMNPlane reference! -->
</bpmndi:BPMNPlane>
```

**Why it's wrong:**
- `BPMNPlane` with `bpmnElement="Process_Main"` already creates the visual container
- Adding a `BPMNShape` with the same `bpmnElement="Process_Main"` creates duplicate DI elements
- Camunda Modeler error: "multiple DI elements defined for <bpmn:Process>"

### Rule 2: Collaboration (Rare - Only for cross-organizational processes)

When creating a **collaboration diagram** (multiple organizations, message flows between them):

✅ **CORRECT Structure:**
```xml
<bpmn:collaboration id="Collaboration_1">
  <bpmn:participant id="Participant_Company1" name="Company A" processRef="Process_Company1" />
  <bpmn:participant id="Participant_Company2" name="Company B" processRef="Process_Company2" />
  <bpmn:messageFlow id="Flow_Message1" sourceRef="Task_SendRequest" targetRef="Task_ReceiveRequest" />
</bpmn:collaboration>

<bpmn:process id="Process_Company1" name="Company A Process" isExecutable="true">
  <!-- Process elements for Company A -->
</bpmn:process>

<bpmn:process id="Process_Company2" name="Company B Process" isExecutable="true">
  <!-- Process elements for Company B -->
</bpmn:process>

<bpmndi:BPMNDiagram id="BPMNDiagram_1">
  <bpmndi:BPMNPlane id="BPMNPlane_1" bpmnElement="Collaboration_1">
    <!-- NOW Pool shapes are correct -->
    <bpmndi:BPMNShape id="Participant_Company1_di" bpmnElement="Participant_Company1" isHorizontal="true">
      <dc:Bounds x="160" y="80" width="600" height="250" />
    </bpmndi:BPMNShape>

    <bpmndi:BPMNShape id="Participant_Company2_di" bpmnElement="Participant_Company2" isHorizontal="true">
      <dc:Bounds x="160" y="400" width="600" height="250" />
    </bpmndi:BPMNShape>

    <!-- Element shapes... -->
    <!-- Message flow edges... -->
  </bpmndi:BPMNPlane>
</bpmndi:BPMNDiagram>
```

### Decision Tree: Which Structure to Use?

```
Do you have multiple organizations exchanging messages?
│
├─ NO → Use Rule 1 (Single Process)
│        - BPMNPlane references Process directly
│        - NO Pool shapes
│        - NO collaboration element
│        - Use PMA approach (role overlays)
│
└─ YES → Use Rule 2 (Collaboration)
         - BPMNPlane references Collaboration
         - Each organization = Participant with Pool shape
         - Message flows between participants
         - Sequence flows within each pool
```

### Common Mistake Pattern

❌ **Incorrect thinking:** "I want to show a pool visually, so I'll add a Pool shape"

✅ **Correct thinking:**
- Single process → BPMNPlane IS the visual container (no explicit pool shape needed)
- Multiple organizations → Use Collaboration + Participant elements + Pool shapes

### Error Messages & Fixes

| Error Message | Cause | Fix |
|---------------|-------|-----|
| "multiple DI elements defined for Process_XXX" | Added Pool shape when BPMNPlane references Process | Remove the Pool shape |
| "element referenced by sourceRef not yet drawn" | Process element not in correct pool | Check collaboration structure |
| "unparsable content detected" | XML syntax error | Validate closing tags |

---

## Events

### Start Events
```xml
<!-- None (manual trigger) -->
<bpmn:startEvent id="Start_[Name]" name="[Trigger Description]">
  <bpmn:outgoing>Flow_Start</bpmn:outgoing>
</bpmn:startEvent>

<!-- Message Start -->
<bpmn:startEvent id="Start_Message_[Name]" name="[Message] Received">
  <bpmn:messageEventDefinition messageRef="Message_[Name]"/>
  <bpmn:outgoing>Flow_Start</bpmn:outgoing>
</bpmn:startEvent>

<!-- Timer Start -->
<bpmn:startEvent id="Start_Timer_[Name]" name="[Schedule Description]">
  <bpmn:timerEventDefinition>
    <bpmn:timeCycle>0 9 * * MON-FRI</bpmn:timeCycle>  <!-- Cron: weekdays 9AM -->
  </bpmn:timerEventDefinition>
  <bpmn:outgoing>Flow_Start</bpmn:outgoing>
</bpmn:startEvent>

<!-- Signal Start -->
<bpmn:startEvent id="Start_Signal_[Name]" name="[Signal] Triggered">
  <bpmn:signalEventDefinition signalRef="Signal_[Name]"/>
  <bpmn:outgoing>Flow_Start</bpmn:outgoing>
</bpmn:startEvent>
```

### End Events
```xml
<!-- None (normal completion) -->
<bpmn:endEvent id="End_[OutcomeName]" name="[Outcome Description]">
  <bpmn:incoming>Flow_ToEnd</bpmn:incoming>
</bpmn:endEvent>

<!-- Error End -->
<bpmn:endEvent id="End_Error_[Name]" name="Process Failed: [Reason]">
  <bpmn:errorEventDefinition errorRef="Error_[Name]"/>
  <bpmn:incoming>Flow_ToError</bpmn:incoming>
</bpmn:endEvent>

<!-- Terminate End (stops all parallel flows) -->
<bpmn:endEvent id="End_Terminate_[Name]" name="[Reason] - Terminate">
  <bpmn:terminateEventDefinition/>
  <bpmn:incoming>Flow_ToTerminate</bpmn:incoming>
</bpmn:endEvent>

<!-- Message End -->
<bpmn:endEvent id="End_Message_[Name]" name="[Message] Sent">
  <bpmn:messageEventDefinition messageRef="Message_[Name]"/>
  <bpmn:incoming>Flow_ToMessageEnd</bpmn:incoming>
</bpmn:endEvent>
```

### Intermediate Events
```xml
<!-- Timer Catch (wait) -->
<bpmn:intermediateCatchEvent id="Timer_Wait_[Name]" name="Wait [Duration]">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:timerEventDefinition>
    <bpmn:timeDuration>PT24H</bpmn:timeDuration>  <!-- ISO 8601: 24 hours -->
  </bpmn:timerEventDefinition>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>
</bpmn:intermediateCatchEvent>

<!-- Message Catch -->
<bpmn:intermediateCatchEvent id="Catch_Message_[Name]" name="[Message] Received">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:messageEventDefinition messageRef="Message_[Name]"/>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>
</bpmn:intermediateCatchEvent>

<!-- Message Throw -->
<bpmn:intermediateThrowEvent id="Throw_Message_[Name]" name="Send [Message]">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:messageEventDefinition messageRef="Message_[Name]"/>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>
</bpmn:intermediateThrowEvent>

<!-- Signal Throw -->
<bpmn:intermediateThrowEvent id="Throw_Signal_[Name]" name="Broadcast [Signal]">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:signalEventDefinition signalRef="Signal_[Name]"/>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>
</bpmn:intermediateThrowEvent>
```

### Boundary Events
```xml
<!-- Timer Boundary (non-interrupting for escalation) -->
<bpmn:boundaryEvent id="Boundary_Timer_[Name]" name="[Duration] Elapsed" 
                    attachedToRef="Activity_[Parent]" cancelActivity="false">
  <bpmn:timerEventDefinition>
    <bpmn:timeDuration>P2D</bpmn:timeDuration>  <!-- 2 days -->
  </bpmn:timerEventDefinition>
  <bpmn:outgoing>Flow_Escalation</bpmn:outgoing>
</bpmn:boundaryEvent>

<!-- Timer Boundary (interrupting for timeout) -->
<bpmn:boundaryEvent id="Boundary_Timeout_[Name]" name="[Duration] Timeout" 
                    attachedToRef="Activity_[Parent]" cancelActivity="true">
  <bpmn:timerEventDefinition>
    <bpmn:timeDuration>P7D</bpmn:timeDuration>
  </bpmn:timerEventDefinition>
  <bpmn:outgoing>Flow_Timeout</bpmn:outgoing>
</bpmn:boundaryEvent>

<!-- Error Boundary -->
<bpmn:boundaryEvent id="Boundary_Error_[Name]" name="[Error Type]" 
                    attachedToRef="Activity_[Parent]">
  <bpmn:errorEventDefinition errorRef="Error_[Name]"/>
  <bpmn:outgoing>Flow_ErrorHandler</bpmn:outgoing>
</bpmn:boundaryEvent>

<!-- Message Boundary -->
<bpmn:boundaryEvent id="Boundary_Message_[Name]" name="[Message] Received" 
                    attachedToRef="Activity_[Parent]" cancelActivity="false">
  <bpmn:messageEventDefinition messageRef="Message_[Name]"/>
  <bpmn:outgoing>Flow_MessageHandler</bpmn:outgoing>
</bpmn:boundaryEvent>
```

## Tasks

```xml
<!-- User Task -->
<bpmn:userTask id="Task_[VerbNoun]" name="[Verb] [Object]" camunda:assignee="${assignee}">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>
</bpmn:userTask>

<!-- Service Task (External) -->
<bpmn:serviceTask id="Task_[VerbNoun]" name="[Verb] [Object]" 
                  camunda:type="external" camunda:topic="[topicName]">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>
</bpmn:serviceTask>

<!-- Service Task (Delegate) -->
<bpmn:serviceTask id="Task_[VerbNoun]" name="[Verb] [Object]" 
                  camunda:delegateExpression="${[beanName]}">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>
</bpmn:serviceTask>

<!-- Script Task -->
<bpmn:scriptTask id="Task_[VerbNoun]" name="[Verb] [Object]" scriptFormat="groovy">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>
  <bpmn:script><![CDATA[
    // Script content
    execution.setVariable("result", calculatedValue)
  ]]></bpmn:script>
</bpmn:scriptTask>

<!-- Business Rule Task -->
<bpmn:businessRuleTask id="Task_[VerbNoun]" name="[Verb] [Object]"
                       camunda:resultVariable="[outputVar]"
                       camunda:decisionRef="[dmnDecisionId]"
                       camunda:mapDecisionResult="singleEntry">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>
</bpmn:businessRuleTask>

<!-- Manual Task -->
<bpmn:manualTask id="Task_[VerbNoun]" name="[Verb] [Object]">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>
</bpmn:manualTask>

<!-- Send Task -->
<bpmn:sendTask id="Task_Send_[Name]" name="Send [Message/Document]"
               camunda:type="external" camunda:topic="[sendTopic]">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>
</bpmn:sendTask>

<!-- Receive Task -->
<bpmn:receiveTask id="Task_Receive_[Name]" name="Await [Response]"
                  messageRef="Message_[Name]">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>
</bpmn:receiveTask>
```

## Gateways

```xml
<!-- Exclusive Gateway (XOR) - Decision -->
<bpmn:exclusiveGateway id="Gateway_[Question]" name="[Question]?" default="Flow_Default">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_ConditionA</bpmn:outgoing>
  <bpmn:outgoing>Flow_ConditionB</bpmn:outgoing>
  <bpmn:outgoing>Flow_Default</bpmn:outgoing>
</bpmn:exclusiveGateway>

<!-- Exclusive Gateway (XOR) - Merge -->
<bpmn:exclusiveGateway id="Gateway_Merge_[Name]">
  <bpmn:incoming>Flow_BranchA</bpmn:incoming>
  <bpmn:incoming>Flow_BranchB</bpmn:incoming>
  <bpmn:outgoing>Flow_Merged</bpmn:outgoing>
</bpmn:exclusiveGateway>

<!-- Parallel Gateway (AND) - Fork -->
<bpmn:parallelGateway id="Gateway_Fork_[Name]" name="Execute Parallel">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Branch1</bpmn:outgoing>
  <bpmn:outgoing>Flow_Branch2</bpmn:outgoing>
</bpmn:parallelGateway>

<!-- Parallel Gateway (AND) - Join -->
<bpmn:parallelGateway id="Gateway_Join_[Name]" name="Synchronize">
  <bpmn:incoming>Flow_Branch1_End</bpmn:incoming>
  <bpmn:incoming>Flow_Branch2_End</bpmn:incoming>
  <bpmn:outgoing>Flow_Synchronized</bpmn:outgoing>
</bpmn:parallelGateway>

<!-- Inclusive Gateway (OR) - Split -->
<bpmn:inclusiveGateway id="Gateway_Inclusive_[Name]" name="Select Applicable" default="Flow_Default">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_OptionA</bpmn:outgoing>
  <bpmn:outgoing>Flow_OptionB</bpmn:outgoing>
  <bpmn:outgoing>Flow_OptionC</bpmn:outgoing>
</bpmn:inclusiveGateway>

<!-- Inclusive Gateway (OR) - Merge -->
<bpmn:inclusiveGateway id="Gateway_InclusiveMerge_[Name]">
  <bpmn:incoming>Flow_OptionA_End</bpmn:incoming>
  <bpmn:incoming>Flow_OptionB_End</bpmn:incoming>
  <bpmn:incoming>Flow_OptionC_End</bpmn:incoming>
  <bpmn:outgoing>Flow_Merged</bpmn:outgoing>
</bpmn:inclusiveGateway>

<!-- Event-Based Gateway -->
<bpmn:eventBasedGateway id="Gateway_EventBased_[Name]" name="Await">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_ToMessageEvent</bpmn:outgoing>
  <bpmn:outgoing>Flow_ToTimerEvent</bpmn:outgoing>
</bpmn:eventBasedGateway>
```

## Sequence Flows

```xml
<!-- Simple Flow -->
<bpmn:sequenceFlow id="Flow_[Source]_[Target]" sourceRef="[SourceId]" targetRef="[TargetId]"/>

<!-- Conditional Flow -->
<bpmn:sequenceFlow id="Flow_[ConditionName]" name="[Human-readable label]" 
                   sourceRef="Gateway_[Name]" targetRef="[TargetId]">
  <bpmn:conditionExpression xsi:type="bpmn:tFormalExpression">
    ${[variable] == '[value]'}
  </bpmn:conditionExpression>
</bpmn:sequenceFlow>

<!-- Default Flow (no condition expression, referenced by gateway's default attribute) -->
<bpmn:sequenceFlow id="Flow_Default_[Name]" name="Otherwise" 
                   sourceRef="Gateway_[Name]" targetRef="[TargetId]"/>
```

## Subprocess

```xml
<!-- Embedded Subprocess -->
<bpmn:subProcess id="SubProcess_[Name]" name="[Subprocess Description]">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>
  
  <bpmn:startEvent id="SubStart_[Name]">
    <bpmn:outgoing>SubFlow_Start</bpmn:outgoing>
  </bpmn:startEvent>
  
  <!-- Subprocess elements -->
  
  <bpmn:endEvent id="SubEnd_[Name]">
    <bpmn:incoming>SubFlow_End</bpmn:incoming>
  </bpmn:endEvent>
</bpmn:subProcess>

<!-- Call Activity (reusable subprocess) -->
<bpmn:callActivity id="Call_[ProcessName]" name="Execute [Process]" calledElement="[ProcessId]">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>
</bpmn:callActivity>
```

## Definitions (Messages, Signals, Errors)

Place before `<bpmn:process>`:

```xml
<bpmn:message id="Message_[Name]" name="[MessageName]"/>
<bpmn:signal id="Signal_[Name]" name="[SignalName]"/>
<bpmn:error id="Error_[Name]" name="[ErrorName]" errorCode="[ERROR_CODE]"/>
```

## Visual Layout (BPMNDiagram)

### Standard Dimensions
| Element | Width | Height |
|---------|-------|--------|
| Start/End Event | 36 | 36 |
| Intermediate Event | 36 | 36 |
| Boundary Event | 36 | 36 |
| Task | 100 | 80 |
| Gateway | 50 | 50 |
| Subprocess | 350 | 200 (min) |

### Shape Templates
```xml
<!-- Event -->
<bpmndi:BPMNShape id="[ElementId]_di" bpmnElement="[ElementId]">
  <dc:Bounds x="[X]" y="[Y]" width="36" height="36"/>
  <bpmndi:BPMNLabel>
    <dc:Bounds x="[X-10]" y="[Y+40]" width="[textWidth]" height="14"/>
  </bpmndi:BPMNLabel>
</bpmndi:BPMNShape>

<!-- Task -->
<bpmndi:BPMNShape id="[ElementId]_di" bpmnElement="[ElementId]">
  <dc:Bounds x="[X]" y="[Y]" width="100" height="80"/>
</bpmndi:BPMNShape>

<!-- Gateway -->
<bpmndi:BPMNShape id="[ElementId]_di" bpmnElement="[ElementId]" isMarkerVisible="true">
  <dc:Bounds x="[X]" y="[Y]" width="50" height="50"/>
  <bpmndi:BPMNLabel>
    <dc:Bounds x="[X-5]" y="[Y-20]" width="[textWidth]" height="14"/>
  </bpmndi:BPMNLabel>
</bpmndi:BPMNShape>

<!-- Lane -->
<bpmndi:BPMNShape id="[LaneId]_di" bpmnElement="[LaneId]" isHorizontal="true">
  <dc:Bounds x="160" y="[Y]" width="[Width]" height="[Height]"/>
</bpmndi:BPMNShape>

<!-- Pool/Participant -->
<bpmndi:BPMNShape id="[ParticipantId]_di" bpmnElement="[ParticipantId]" isHorizontal="true">
  <dc:Bounds x="130" y="[Y]" width="[Width]" height="[Height]"/>
</bpmndi:BPMNShape>
```

### Edge Templates
```xml
<!-- Straight horizontal flow -->
<bpmndi:BPMNEdge id="[FlowId]_di" bpmnElement="[FlowId]">
  <di:waypoint x="[SourceX + SourceWidth]" y="[SourceCenterY]"/>
  <di:waypoint x="[TargetX]" y="[TargetCenterY]"/>
</bpmndi:BPMNEdge>

<!-- Flow with corner (L-shape down then right) -->
<bpmndi:BPMNEdge id="[FlowId]_di" bpmnElement="[FlowId]">
  <di:waypoint x="[GatewayX + 25]" y="[GatewayY + 50]"/>  <!-- Exit bottom -->
  <di:waypoint x="[GatewayX + 25]" y="[TargetCenterY]"/>  <!-- Go down -->
  <di:waypoint x="[TargetX]" y="[TargetCenterY]"/>        <!-- Go right -->
  <bpmndi:BPMNLabel>
    <dc:Bounds x="[LabelX]" y="[LabelY]" width="[Width]" height="14"/>
  </bpmndi:BPMNLabel>
</bpmndi:BPMNEdge>

<!-- Flow with corner (L-shape up then right) -->
<bpmndi:BPMNEdge id="[FlowId]_di" bpmnElement="[FlowId]">
  <di:waypoint x="[GatewayX + 25]" y="[GatewayY]"/>       <!-- Exit top -->
  <di:waypoint x="[GatewayX + 25]" y="[TargetCenterY]"/> <!-- Go up -->
  <di:waypoint x="[TargetX]" y="[TargetCenterY]"/>       <!-- Go right -->
</bpmndi:BPMNEdge>
```

### Coordinate Calculation Rules
```
Task center Y = Task Y + 40
Event center Y = Event Y + 18
Gateway center Y = Gateway Y + 25

Gateway exit points:
  - Right: (X + 50, Y + 25)
  - Bottom: (X + 25, Y + 50)
  - Top: (X + 25, Y)
  - Left: (X, Y + 25)

Task connection points:
  - Right: (X + 100, Y + 40)
  - Left: (X, Y + 40)
```
