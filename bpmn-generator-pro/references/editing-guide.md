# BPMN Editing Guide

Practical guide for modifying existing BPMN diagrams.

## Reading Existing BPMN Files

### Step 1: Parse the Structure

When reading a .bpmn file, identify key sections:

```xml
<bpmn:definitions>
  <!-- Message/Signal/Error definitions -->
  <bpmn:message id="Message_X" name="..."/>

  <!-- Collaboration (if multi-participant) -->
  <bpmn:collaboration id="Collaboration_1">
    <bpmn:participant id="Participant_1" processRef="Process_1"/>
  </bpmn:collaboration>

  <!-- Process definition -->
  <bpmn:process id="Process_1" name="...">
    <bpmn:laneSet>...</bpmn:laneSet>
    <!-- Tasks, Events, Gateways -->
    <!-- Sequence Flows -->
  </bpmn:process>

  <!-- Visual diagram -->
  <bpmndi:BPMNDiagram>
    <bpmndi:BPMNPlane bpmnElement="...">
      <!-- Shapes and Edges -->
    </bpmndi:BPMNPlane>
  </bpmndi:BPMNDiagram>
</bpmn:definitions>
```

### Step 2: Build Element Map

Create mental map of:
- All element IDs and their types
- Flow connections (source → target)
- Visual positions (x, y coordinates)
- Lane assignments

### Step 3: Identify Modification Points

Based on user request, locate:
- Which elements to modify
- Where to insert new elements
- Which flows to update
- Visual layout impact

## Common Editing Scenarios

### Scenario 1: Insert Task in Sequential Flow

**Original:**
```
Task_A → Task_B
```

**Modified:**
```
Task_A → Task_NEW → Task_B
```

**Steps:**
1. Create new task element
2. Update flow from Task_A to point to Task_NEW
3. Create new flow from Task_NEW to Task_B
4. Calculate position: X = (Task_A.x + Task_B.x) / 2
5. Add BPMNShape for Task_NEW

**XML Changes:**
```xml
<!-- Add new task -->
<bpmn:userTask id="Task_NEW" name="New Action">
  <bpmn:incoming>Flow_A_NEW</bpmn:incoming>
  <bpmn:outgoing>Flow_NEW_B</bpmn:outgoing>
</bpmn:userTask>

<!-- Update existing flow -->
<bpmn:sequenceFlow id="Flow_A_B" sourceRef="Task_A" targetRef="Task_NEW"/>

<!-- Add new flow -->
<bpmn:sequenceFlow id="Flow_NEW_B" sourceRef="Task_NEW" targetRef="Task_B"/>

<!-- Update Task_A outgoing -->
<bpmn:userTask id="Task_A" name="...">
  <bpmn:outgoing>Flow_A_NEW</bpmn:outgoing>  <!-- Changed from Flow_A_B -->
</bpmn:userTask>

<!-- Update Task_B incoming -->
<bpmn:userTask id="Task_B" name="...">
  <bpmn:incoming>Flow_NEW_B</bpmn:incoming>  <!-- Changed from Flow_A_B -->
</bpmn:userTask>
```

### Scenario 2: Add Approval Gateway

**Original:**
```
Task_Submit → Task_Process → End
```

**Modified:**
```
Task_Submit → Task_Review → [Approved?]
                              ├─ Yes → Task_Process → End(Approved)
                              └─ No → End(Rejected)
```

**Steps:**
1. Insert Task_Review after Task_Submit
2. Add Exclusive Gateway after Task_Review
3. Add conditional flows (Yes/No)
4. Update existing flow to End
5. Add new End(Rejected) event
6. Calculate layout:
   - Review: between Submit and old Process position
   - Gateway: after Review
   - Process: shift right
   - End(Rejected): below gateway

### Scenario 3: Add Error Boundary Event

**Original:**
```xml
<bpmn:serviceTask id="Task_Payment" name="Process Payment">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Success</bpmn:outgoing>
</bpmn:serviceTask>
```

**Modified:**
```xml
<bpmn:serviceTask id="Task_Payment" name="Process Payment">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Success</bpmn:outgoing>
</bpmn:serviceTask>

<!-- Add error boundary -->
<bpmn:boundaryEvent id="Boundary_PaymentError" name="Payment Failed"
                    attachedToRef="Task_Payment">
  <bpmn:errorEventDefinition errorRef="Error_Payment"/>
  <bpmn:outgoing>Flow_ErrorHandler</bpmn:outgoing>
</bpmn:boundaryEvent>

<!-- Add error definition -->
<bpmn:error id="Error_Payment" name="PaymentError" errorCode="PAYMENT_FAILED"/>

<!-- Add error handler task -->
<bpmn:userTask id="Task_HandleError" name="Handle Payment Failure">
  <bpmn:incoming>Flow_ErrorHandler</bpmn:incoming>
  <bpmn:outgoing>Flow_ToEnd</bpmn:outgoing>
</bpmn:userTask>

<!-- Visual: boundary positioned on bottom-right of task -->
<!-- Handler positioned below task -->
```

### Scenario 4: Convert to Multi-Instance

**Original:**
```xml
<bpmn:userTask id="Task_Review" name="Review Document">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>
</bpmn:userTask>
```

**Modified (Parallel Multi-Instance):**
```xml
<bpmn:userTask id="Task_Review" name="Review Document">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>

  <!-- Add multi-instance -->
  <bpmn:multiInstanceLoopCharacteristics isSequential="false"
      camunda:collection="${reviewers}"
      camunda:elementVariable="reviewer">
  </bpmn:multiInstanceLoopCharacteristics>
</bpmn:userTask>
```

### Scenario 5: Add Timer Escalation

**Original:**
```xml
<bpmn:userTask id="Task_Approval" name="Approve Request">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>
</bpmn:userTask>
```

**Modified:**
```xml
<bpmn:userTask id="Task_Approval" name="Approve Request">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>
</bpmn:userTask>

<!-- Non-interrupting timer (escalation) -->
<bpmn:boundaryEvent id="Boundary_Escalation" name="2 Days Elapsed"
                    attachedToRef="Task_Approval" cancelActivity="false">
  <bpmn:timerEventDefinition>
    <bpmn:timeDuration>P2D</bpmn:timeDuration>
  </bpmn:timerEventDefinition>
  <bpmn:outgoing>Flow_Escalate</bpmn:outgoing>
</bpmn:boundaryEvent>

<bpmn:serviceTask id="Task_NotifyManager" name="Notify Manager">
  <bpmn:incoming>Flow_Escalate</bpmn:incoming>
  <bpmn:outgoing>Flow_EscalationEnd</bpmn:outgoing>
</bpmn:serviceTask>

<bpmn:endEvent id="End_Escalation" name="Manager Notified">
  <bpmn:incoming>Flow_EscalationEnd</bpmn:incoming>
</bpmn:endEvent>
```

### Scenario 6: Split Sequential into Parallel

**Original:**
```
Task_A → Task_B → Task_C → Task_D
```

**Modified:**
```
Task_A → ║Fork║ → Task_B ─────┐
                → Task_C ─────┼→ ║Join║ → Task_D
```

**Steps:**
1. Insert Parallel Gateway (fork) after Task_A
2. Insert Parallel Gateway (join) before Task_D
3. Update flows:
   - Task_A → Gateway_Fork
   - Gateway_Fork → Task_B, Task_C (parallel)
   - Task_B, Task_C → Gateway_Join (sync)
   - Gateway_Join → Task_D
4. Adjust visual layout:
   - Task_B: Y = baseline - 50
   - Task_C: Y = baseline + 50
   - Both end at same X before join gateway

## Visual Layout Preservation

### Minimal Disruption Strategy

When editing, preserve existing layout as much as possible:

1. **Insertions**: Calculate midpoint positions
2. **Additions**: Extend diagram, don't compress
3. **Deletions**: Close gaps or leave if minor
4. **Branch additions**: Use vertical offsets (±100px)

### Coordinate Calculation Helpers

```
Insert between A and B:
  new_X = A_X + ((B_X - A_X) / 2)
  new_Y = A_Y  (same level)

Branch offset from baseline:
  upper_Y = baseline_Y - 100
  lower_Y = baseline_Y + 100

Task right edge:
  right_X = task_X + 100

Task center:
  center_X = task_X + 50
  center_Y = task_Y + 40
```

## ID Management

### Keep Existing IDs When:
- Renaming elements (only change `name` attribute)
- Adding properties
- Changing task types (user → service)

### Generate New IDs When:
- Adding new elements
- Splitting one element into multiple
- Copying elements

### ID Patterns:
```
Tasks:         Task_[VerbNoun]
Events:        Start_[Name], End_[Name], Event_[Name]
Gateways:      Gateway_[Question]
Flows:         Flow_[Source]_[Target]
Boundaries:    Boundary_[EventType]_[Parent]
```

## Validation After Editing

Before saving modified BPMN:

- [ ] All incoming/outgoing refs are valid
- [ ] Every element has corresponding visual shape
- [ ] Every flow has corresponding edge
- [ ] No orphan elements
- [ ] Gateway splits have matching joins (if parallel/inclusive)
- [ ] Boundary events reference valid parent tasks
- [ ] All IDs are unique
- [ ] All flow sourceRef/targetRef point to existing elements

## Testing Edits in Camunda Modeler

After editing:
1. Open file in Camunda Modeler
2. Check for validation errors (red markers)
3. Verify visual layout is correct
4. Test element properties
5. Run simulation if available

If errors appear:
- Check XML syntax
- Verify all references
- Ensure proper namespace usage
- Validate against BPMN 2.0 schema
