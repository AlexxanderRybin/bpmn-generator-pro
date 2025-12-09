# BPMN Creation & Editing Workflows

**Reference for:** `SKILL.md` Workflow sections

**Purpose:** Step-by-step workflows for creating new BPMN processes and editing existing .bpmn files.

---

## Workflow 1: Creating New BPMN Process

**Use When:** User requests to create/generate/model a new business process from description.

### Step-by-Step Process

#### Phase 0: Pre-Creation Analysis (MANDATORY)

**CRITICAL:** Complete tabular analysis BEFORE generating any XML.

1. **List all elements from user description:**
   - Start events
   - Tasks (with step numbers if provided)
   - Gateways (decision points)
   - End events
   - Any intermediate events

2. **Create Multi-Merge Pre-Check Table:**

| # | Element ID | Type | Step # | Incoming Count | Sources | Action |
|---|------------|------|--------|----------------|---------|--------|
| 1 | Event_Start | Start | - | 0 | - | ✅ OK |
| 2 | Task_Submit | Task | - | 1 | Event_Start | ✅ OK |
| ... | ... | ... | ... | ... | ... | ... |

3. **Identify Multi-Merge cases:**
   - Any element (except End Events) with incoming > 1
   - Add merge gateways to table
   - Update incoming counts

4. **Verify table:**
   - All tasks: incoming = 1 ✅
   - All gateways (non-merge): incoming = 1 ✅
   - Merge gateways: incoming ≥ 2 ✅
   - End events: incoming ≥ 1 ✅

**See:** `multi-merge-prevention.md` for detailed algorithm.

#### Phase 1: Analysis Framework

Complete the 4-part analysis from `SKILL.md`:

**1. Identify Process Boundaries**
- What starts the process? → Start Event name
- What are the end states? → End Event names
- Where does this process end? → Scope limits

**2. Extract Actors & Responsibilities**
- WHO performs each action? → Role overlays (NOT lanes)
- WHICH systems are involved? → Service Tasks
- WHO makes decisions? → User Tasks before Gateways

**3. Map the Happy Path First**
- List ideal scenario steps
- Identify decision points
- Identify exception paths
- Identify timeout/escalation scenarios

**4. Identify Decision Logic**
- Binary (yes/no) → Exclusive Gateway
- Multiple conditions, one result → Exclusive Gateway
- Multiple conditions, multiple results → Inclusive Gateway
- Parallel work → Parallel Gateway
- Wait for external event → Event-Based Gateway

#### Phase 2: Process Design

**1. Apply PMA Restriction:**
- ❌ **NO LANES** for single-organization processes
- ✅ **Use role overlays** on tasks:
  - Option A: Camunda attributes (`camunda:candidateGroups="finance"`)
  - Option B: Task name prefix (`[Finance] Review Application`)
  - Option C: Step + Role format (`2.5.1 [Finance] Review Application`)

**2. Apply Naming Conventions:**
- Tasks: Verb + Object (`Review Application`)
- Gateways: Question format (`Approved?`, `Amount > 1000?`)
- Events: State/Trigger (`Order Received`, `Request Approved`)

**3. Select Task Types:**
- Use Task Type Selection Matrix from `SKILL.md`
- Human work → User Task
- System work → Service Task
- Calculation → Script Task
- Physical work → Manual Task

**4. Select Gateway Types:**
- Use Gateway Selection Decision Tree from `SKILL.md`
- XOR: Exactly one path
- AND: All paths simultaneously
- OR: One or more paths
- Event-Based: Wait for external event

#### Phase 3: XML Generation

**1. Create Process Structure:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<bpmn:definitions xmlns:bpmn="http://www.omg.org/spec/BPMN/20100524/MODEL"
                   xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI"
                   xmlns:dc="http://www.omg.org/spec/DD/20100524/DC"
                   xmlns:di="http://www.omg.org/spec/DD/20100524/DI"
                   xmlns:camunda="http://camunda.org/schema/1.0/bpmn"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   id="Definitions_1"
                   targetNamespace="http://bpmn.io/schema/bpmn">

  <bpmn:process id="Process_[ProcessName]" isExecutable="true">
    <!-- Elements here -->
  </bpmn:process>

  <bpmndi:BPMNDiagram id="BPMNDiagram_1">
    <bpmndi:BPMNPlane id="BPMNPlane_1" bpmnElement="Process_[ProcessName]">
      <!-- Visual layout here -->
    </bpmndi:BPMNPlane>
  </bpmndi:BPMNDiagram>
</bpmn:definitions>
```

**2. Add Elements (Using xml-templates.md):**
- Start Event
- Tasks (in sequence)
- Gateways (from Phase 0 table, including merge gateways)
- End Events
- Sequence Flows

**3. Add Visual Layout (Using intelligent-layout-algorithm-v3.md):**
- Calculate element positions (X, Y)
- Create BPMNShape for each element
- Calculate flow waypoints
- Create BPMNEdge for each flow

**Apply spacing constants:**
- After start: 52px
- Standard gap: 55px
- Task chain: 40px
- New sequence: 80px
- Lane heights: 200-250px (fixed)
- **End events: Same X and Y** (visual finish line)

#### Phase 4: Validation

Run through validation checklist from `SKILL.md`:

**Structural:**
- [ ] Has exactly ONE start event
- [ ] Has at least ONE end event
- [ ] All elements connected
- [ ] All gateways balanced

**Gateway Logic:**
- [ ] XOR: Default flow defined
- [ ] AND: Fork = Join, no conditions on outgoing
- [ ] OR: At least one path guaranteed, no End between split/join
- [ ] No Multi-Merge (checked in Phase 0)
- [ ] No Implicit Gateway

**Naming:**
- [ ] Tasks: Verb + Object
- [ ] Gateways: Questions
- [ ] Events: State/Trigger

**Visual:**
- [ ] Horizontal spacing: 40-80px
- [ ] Lane heights: 200-250px
- [ ] No overlaps (collision detection)
- [ ] End events grouped

#### Phase 5: File Output & Integration

**1. Save File:**
```bash
# Save as [process-name].bpmn
```

**2. Open in Camunda Modeler:**
```bash
open -a "Camunda Modeler" "[process-name].bpmn"
```

**3. Inform User:**
```
✅ Created: [process-name].bpmn
📊 Elements: [count] tasks, [count] gateways, [count] events
🔍 Opening in Camunda Modeler for visual inspection...
```

---

## Workflow 2: Editing Existing BPMN Process

**Use When:** User requests to modify/edit/update existing .bpmn file.

### Step-by-Step Process

#### Phase 1: Read & Parse

**1. Read Existing File:**
```bash
# Read the .bpmn file
```

**2. Parse Structure:**
- Identify existing elements (IDs, names, types)
- Map sequence flows (connections)
- Extract visual layout (coordinates, waypoints)
- Identify collaboration structure (Pools, Lanes if present)

**3. Understand Current State:**
- What is the happy path?
- What are the decision points?
- What are the current issues (if any)?

#### Phase 2: Plan Changes

**1. Clarify User Request:**
- What needs to be added/removed/modified?
- Where in the process?
- Any specific requirements (roles, conditions, error handling)?

**2. Identify Impact:**
- Which elements will be affected?
- Which flows need to be redirected?
- Does layout need adjustment?

**3. Check for Anti-Patterns:**
- Will change create Multi-Merge?
- Will change create Implicit Gateway?
- Will change unbalance gateways?

**If changes create Multi-Merge:**
- Plan to add merge gateway FIRST
- Update flow mappings

#### Phase 3: Apply Changes

**Common Edit Operations:**

### Operation A: Adding New Task

**Steps:**
1. Insert XML element in `<bpmn:process>`:
```xml
<bpmn:userTask id="Task_New" name="New Task Name">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Out</bpmn:outgoing>
</bpmn:userTask>
```

2. Update sequence flows:
   - Redirect existing flow to new task
   - Create new flow from new task

3. Add visual shape in `<bpmndi:BPMNPlane>`:
```xml
<bpmndi:BPMNShape id="Shape_Task_New" bpmnElement="Task_New">
  <dc:Bounds x="[calculated]" y="[calculated]" width="100" height="80" />
</bpmndi:BPMNShape>
```

4. Update flow waypoints for new connections

**Calculate position:** Based on nearby elements (preserve spacing: 40-80px)

### Operation B: Adding Boundary Event

**Steps:**
1. Add boundary event in `<bpmn:process>`:
```xml
<bpmn:boundaryEvent id="Boundary_Error" attachedToRef="Task_Target">
  <bpmn:outgoing>Flow_ToHandler</bpmn:outgoing>
  <bpmn:errorEventDefinition errorRef="Error_Name" />
</bpmn:boundaryEvent>

<bpmn:error id="Error_Name" name="ErrorName" errorCode="ERROR_CODE" />
```

2. Add handler task and flows

3. Add visual shapes:
```xml
<bpmndi:BPMNShape id="Shape_Boundary" bpmnElement="Boundary_Error">
  <dc:Bounds x="[task_x + task_width - 18]" y="[task_y + task_height - 18]"
             width="36" height="36" />
</bpmndi:BPMNShape>
```

4. Position handler task below main flow (Y + 120px)

### Operation C: Changing Gateway Type

**Steps:**
1. Update gateway element type:
```xml
<!-- FROM -->
<bpmn:exclusiveGateway id="Gateway_1" />

<!-- TO -->
<bpmn:parallelGateway id="Gateway_1" />
```

2. Update outgoing flow conditions:
   - Exclusive → Parallel: Remove conditions (parallel always fires all paths)
   - Parallel → Exclusive: Add conditions + default

3. Check for matching join gateway:
   - If changing fork, change join to match

4. Update visual shape (gateway icon changes based on type)

### Operation D: Adding Exception Path

**Steps:**
1. Add boundary event (error/timer) to task
2. Add exception handler task(s)
3. Add exception end event (with semantic name: "Payment Failed")
4. Connect flows: boundary → handler → end
5. Update visual layout (position exception path below or to right)

### Operation E: Converting to Multi-Instance

**Steps:**
1. Add multi-instance characteristics to task:
```xml
<bpmn:userTask id="Task_Review" name="Review Document">
  <bpmn:multiInstanceLoopCharacteristics isSequential="false">
    <bpmn:loopCardinality>#{documents.size()}</bpmn:loopCardinality>
    <bpmn:inputDataItem>${document}</bpmn:inputDataItem>
  </bpmn:multiInstanceLoopCharacteristics>
</bpmn:userTask>
```

2. Update visual marker (add **|||** at task bottom)

### Operation F: Adding Escalation

**Steps:**
1. Add non-interrupting timer boundary event:
```xml
<bpmn:boundaryEvent id="Timer_Escalate" attachedToRef="Task_Review"
  cancelActivity="false">
  <bpmn:timerEventDefinition>
    <bpmn:timeDuration>P2D</bpmn:timeDuration>
  </bpmn:timerEventDefinition>
  <bpmn:outgoing>Flow_ToEscalation</bpmn:outgoing>
</bpmn:boundaryEvent>
```

2. Add escalation task (notify manager)
3. Escalation task flows back to main path (merge) or ends independently

### Operation G: Converting Sequential to Parallel

**Steps:**
1. Identify tasks that can be parallelized
2. Insert Parallel Gateway (fork) before first task
3. Redirect flows to fork gateway
4. Create parallel flows to all tasks
5. Insert Parallel Gateway (join) after all tasks
6. Redirect task outputs to join
7. Continue flow from join
8. Update visual layout (parallel paths at different Y coordinates)

### Operation H: Renaming Elements

**Steps:**
1. Update `name` attribute:
```xml
<bpmn:userTask id="Task_1" name="New Task Name" />
```

2. Preserve `id` attribute (or update all `sourceRef`/`targetRef` if changing ID)

3. No visual layout changes needed (just name)

#### Phase 4: Validation

**After changes, validate:**

1. **Run Multi-Merge check:**
   - Did changes create element with multiple incoming?
   - If yes, add merge gateway

2. **Run Implicit Gateway check:**
   - Does any task have multiple outgoing with conditions?
   - If yes, add explicit gateway

3. **Run Gateway balance check:**
   - Are fork/join gateways matched?
   - Are types consistent?

4. **Run structural check:**
   - All elements still connected?
   - All flows have valid source/target?

5. **Run visual check:**
   - No overlapping elements?
   - Spacing consistent (40-80px)?
   - Waypoints clean (2-3 per flow)?

#### Phase 5: File Output & Integration

**1. Save File:**
```bash
# Overwrite existing [process-name].bpmn
```

**2. Open in Camunda Modeler:**
```bash
open -a "Camunda Modeler" "[process-name].bpmn"
```

**3. Inform User:**
```
✅ Updated: [process-name].bpmn
📝 Changes:
  - [Description of changes made]
🔍 Opening in Camunda Modeler for visual inspection...
```

---

## Workflow 3: Iterative Refinement

**Use When:** User wants to refine process through multiple iterations.

### Recommended Cycle

```
1. User describes process → Claude generates BPMN
   ↓
2. File opens in Camunda Modeler → User reviews visually
   ↓
3. User provides feedback → Claude applies changes
   ↓
4. File reopens in Camunda → User reviews again
   ↓
5. Repeat steps 3-4 until satisfied
   ↓
6. Final polish in Camunda (colors, fonts, final layout tweaks)
```

### Best Practices

**For User:**
- Review visually in Camunda after each generation
- Provide specific feedback ("add error handling to Payment task")
- Test process logic (trace happy path, exception paths)
- Use Camunda's validation (check for errors/warnings)

**For Claude:**
- Always open file in Camunda after changes
- Preserve existing layout when possible (minimal adjustments)
- Validate against anti-patterns after each change
- Explain changes made in each iteration

---

## Workflow 4: From Requirements Document

**Use When:** User provides numbered process steps (e.g., 2.1.1, 2.1.2, 2.2.1).

### Special Considerations

**1. Use Element Numbering:**
- Apply step numbers to task names: `2.5.1 [Finance] Review Application`
- Format: `Step# [Role] Task Name`
- Gateway names: `Step# Question` (e.g., `2.7.1 Approved?`)

**2. Preserve Traceability:**
- Add step numbers as XML comments:
```xml
<!-- Step 2.5.1 -->
<bpmn:userTask id="Task_Review" name="2.5.1 [Finance] Review Application" />
```

**3. Map Requirements to BPMN:**
- User requirements → Phase 0 table
- Each step → Table row
- Detect Multi-Merge from step dependencies

**4. Validate Against Requirements:**
- All numbered steps included?
- Step sequence preserved?
- Decision points mapped to gateways?

---

## Common Pitfalls & How to Avoid

### Pitfall 1: Skipping Phase 0

**Problem:** Generate XML directly → Multi-Merge errors

**Solution:** ALWAYS complete tabular analysis first

### Pitfall 2: Using Lanes for Departments

**Problem:** Single-org process with lanes → zigzag happy path

**Solution:** Apply PMA restriction → role overlays only

### Pitfall 3: Editing Without Reading

**Problem:** Modify .bpmn without understanding current state → break existing logic

**Solution:** ALWAYS read file first, parse structure, then plan changes

### Pitfall 4: Not Validating After Changes

**Problem:** Apply changes → introduce anti-pattern → user sees error in Camunda

**Solution:** Run validation checklist after EVERY change

### Pitfall 5: Regenerating Layout Unnecessarily

**Problem:** Recalculate all coordinates when editing → user's manual adjustments lost

**Solution:** Preserve existing coordinates, adjust minimally only where needed

---

## Quick Reference: Process Creation Checklist

**Before generating XML:**
- [ ] Phase 0 table completed (Multi-Merge check)
- [ ] Analysis Framework completed (4 parts)
- [ ] PMA restriction applied (no lanes for single-org)
- [ ] Naming convention selected (with/without step numbers)
- [ ] Task types selected (User/Service/Script/etc.)
- [ ] Gateway types selected (XOR/AND/OR/Event-Based)

**During XML generation:**
- [ ] Using correct namespaces
- [ ] IDs are unique and consistent (Task_, Gateway_, Flow_, Event_)
- [ ] All elements have both process XML and visual DI
- [ ] Spacing follows algorithm (40-80px horizontal)
- [ ] End events aligned on same line
- [ ] No overlapping elements

**After XML generation:**
- [ ] Validation checklist passed (structural, naming, gateway logic)
- [ ] File saved as [process-name].bpmn
- [ ] File opened in Camunda Modeler
- [ ] User informed of completion

**After user feedback:**
- [ ] Read existing file
- [ ] Plan changes (check for anti-patterns)
- [ ] Apply changes preserving layout
- [ ] Re-validate
- [ ] Reopen in Camunda

---

**Back to:** `SKILL.md` Workflow sections

**See Also:**
- `multi-merge-prevention.md` - Phase 0 detailed algorithm
- `editing-guide.md` - Detailed editing patterns with examples
- `intelligent-layout-algorithm-v3.md` - Layout calculation
- `antipatterns-full.md` - Validation against anti-patterns
