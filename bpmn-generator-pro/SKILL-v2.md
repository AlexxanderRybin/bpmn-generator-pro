---
name: bpmn-generator-pro
description: Expert-level BPMN 2.0 process modeler with Camunda Modeler integration. Transforms natural language descriptions into professional BPMN diagrams AND edits existing .bpmn files. Creates/modifies processes with proper structure, naming conventions, exception handling, and visual layout. Automatically opens files in Camunda Modeler for visual inspection. Use when user asks to model, create, edit, modify, draw, or generate any business process, workflow, or BPMN diagram.
---

# BPMN Generator Pro

Expert-level BPMN 2.0 modeling engine. Think like a Senior Process Architect.

## Analysis Framework (MANDATORY FIRST STEP)

Before generating XML, analyze the process description:

### 1. Identify Process Boundaries
- **Trigger**: What starts the process? (request, event, schedule, message)
- **Outcome**: What are the possible end states? (success, rejection, cancellation, error)
- **Scope**: Where does THIS process end and ANOTHER begin?

### 2. Extract Actors & Responsibilities
- WHO performs each action? → Role overlays on tasks (NOT lanes - see PMA restriction below)
  - **If user provides step numbers**: Use format `N.N.N [Role] Task Name` (see Element Numbering section)
- WHICH systems are involved? → Service Tasks
- WHO makes decisions? → User Tasks before Gateways

### 3. Map the Happy Path First
Model the ideal scenario from start to successful end, then add:
- Decision points (where flow branches)
- Exception paths (what can go wrong)
- Timeout/escalation scenarios

### 4. Identify Decision Logic
For each decision point determine:
- Binary (yes/no) → Exclusive Gateway
- Multiple conditions, one result → Exclusive Gateway
- Multiple conditions, multiple results → Inclusive Gateway
- Parallel work → Parallel Gateway
- Wait for external event → Event-Based Gateway

## Process Layout Philosophy (PMA Approach) 🚫 CRITICAL

### PMA Restriction: Lanes are PROHIBITED for Single-Organization Processes ❌

**Scope:** This restriction applies to processes within a SINGLE organization (one company with internal departments).

**Reason:** Using Lanes (Swimlanes/Pools with multiple lanes) for internal departments causes:
- ❌ **Increased model size** - diagrams become larger and harder to view
- ❌ **Broken happy path** - successful scenario zigzags across lanes instead of flowing straight
- ❌ **Harder to follow** - reader must jump between lanes to track main flow
- ❌ **Reduced readability** - visual complexity obscures the process logic

**Golden Rule:**
> **"The happy path should be a straight line, not a zigzag across lanes."**

### RECOMMENDED Approach: Role Overlays ✅

Indicate roles directly on tasks using Camunda attributes or text annotations.

**Benefits:**
- ✅ **Compact diagram** - smaller, easier to view
- ✅ **Straight happy path** - main flow is a clear horizontal line
- ✅ **Better readability** - focus on process logic, not organizational structure
- ✅ **Easier maintenance** - role changes don't require layout restructuring
- ✅ **Execution-ready** - Camunda attributes directly assign tasks

### Implementation Options

**Option A: Camunda Attributes (Preferred)**
```xml
<bpmn:userTask id="Task_Review" name="Review Application"
               camunda:assignee="${reviewer}"
               camunda:candidateGroups="underwriters,managers">
</bpmn:userTask>
```

**Option B: Task Name Prefix (Simple)**
```xml
<bpmn:userTask id="Task_Review" name="[Finance] Review Application">
</bpmn:userTask>
```

**Option C: ⭐ Step Number + Role (When user provides numbered steps)**
```xml
<bpmn:userTask id="Task_ReviewApplication"
               name="2.5.1 [Preparation Specialist] Review Application"
               camunda:candidateGroups="preparation-specialist">
</bpmn:userTask>
```

### When Lanes ARE Acceptable ✅

Lanes are ALLOWED for:
1. **Multi-Organization Processes** - Different companies collaborating (Customer ↔ Supplier)
2. **Cross-System Collaboration** - Independent systems exchanging messages
3. **Explicit User Requirement** - User specifically requests lanes (with warning)

**Key Distinction:**
- ❌ Single organization, multiple departments → NO lanes (use role overlays)
- ✅ Multiple organizations → YES lanes (use separate pools)

## Naming Conventions (CRITICAL)

### Task Names: Verb + Object
```
✅ "Review Application"      ❌ "Application Review"
✅ "Calculate Discount"      ❌ "Discount Calculation"
✅ "Send Confirmation"       ❌ "Confirmation Sending"
```

### Gateway Names: Question Format
```
✅ "Approved?"               ❌ "Check Approval"
✅ "Amount > 10000?"         ❌ "Amount Gateway"
✅ "Documents Complete?"     ❌ "Document Check"
```

### Event Names: State/Trigger Description
```
✅ "Order Received"          ❌ "Start"
✅ "Request Approved"        ❌ "End"
✅ "Payment Timeout"         ❌ "Timer"
```

### Element Numbering (Optional)

**Use when:** User provides process description with step numbers (2.5.1, 2.5.2, etc.)

**Format:** `N.N.N [Role] Task Name`

**Example:**
```xml
<bpmn:userTask name="2.5.1 [Finance] Review Application" />
<bpmn:exclusiveGateway name="2.7.1 Approved?" />
```

**Benefits:** Traceability to source documentation + visual role identification

## Phase 0: Multi-Merge Pre-Check (MANDATORY) 🔍

**CRITICAL:** Complete this analysis BEFORE creating XML elements.

**Why:** Multi-Merge anti-pattern (multiple incoming flows) is commonly overlooked. Systematic checking prevents errors.

### Quick Algorithm

**For EVERY sequence flow you create:**

1. **Identify target element**
2. **Check target type:**
   - End Event? → ✅ Allow multiple incoming
   - Task/Gateway/Other Event? → Continue to step 3
3. **Count existing incoming flows:**
   - incoming.length === 0 → ✅ Add flow directly
   - incoming.length >= 1 → ⚠️ **INSERT MERGE GATEWAY FIRST**

### Required Table Format

Create inventory BEFORE generating XML:

| Element ID | Type | Step # | Incoming Count | Sources | Action |
|------------|------|--------|----------------|---------|--------|
| Event_Start | Start | - | 0 | - | ✅ OK |
| Task_Check | Task | 2.1.1 | 1 | Event_Start | ✅ OK |
| Task_Package | Task | 2.2.4 | **2** | Gateway_A, Gateway_B | ⚠️ **ADD MERGE** |
| Event_End | End | - | 3 | Multiple | ✅ OK (End allows multiple) |

**Rule:** If ANY element (except End Events) has incoming > 1, add merge gateway FIRST.

**Full details:** See `references/multi-merge-prevention.md`

## Task Type Selection Matrix

| Situation | Task Type | Indicator Words |
|-----------|-----------|-----------------|
| Human reviews/decides/approves | User Task | review, approve, decide, check, verify |
| System calls API/service | Service Task | send, notify, update system, integrate |
| Calculation/transformation | Script Task | calculate, transform, generate, compute |
| Physical work outside system | Manual Task | deliver, install, physically inspect |
| Decision table/rules | Business Rule Task | determine, evaluate rules, apply policy |

## Gateway Selection Decision Tree

```
Is it a choice/decision?
├─ YES: How many paths can be active?
│   ├─ Exactly ONE path → Exclusive Gateway (XOR)
│   └─ ONE OR MORE paths → Inclusive Gateway (OR)
│
└─ NO: Is work done simultaneously?
    ├─ YES → Parallel Gateway (AND)
    └─ NO: Waiting for external event? → Event-Based Gateway
```

## Structural Patterns (Common Flows)

### 1. Simple Approval
```
Start → Submit → Review → [Approved?]
                            ├─ Yes → Process → End(Approved)
                            └─ No → Notify → End(Rejected)
```

### 2. Parallel Processing with Sync
```
Start → Order → ║Split║ → Check Inventory ──┐
                        → Validate Payment ──┼→ ║Join║ → Continue
                        → Verify Address ────┘
```

### 3. Timeout/Escalation
```
... → Send Request → ◇Event Gateway◇ → ○ Response → Process → ...
                                      → ⏱ 48h Timeout → Escalate → ...
```

**More patterns:** See `references/examples-guide.md` for 10+ detailed patterns

## Top 5 Critical Anti-Patterns (MUST AVOID)

### 1. Multi-Merge ❌ MOST CRITICAL
**Problem:** Multiple flows directly into task without explicit gateway

**NEVER:**
```
Gateway_A → Task_Final
Gateway_B → Task_Final  ❌ Task receives 2 tokens!
```

**ALWAYS:**
```
Gateway_A → Merge_Gateway → Task_Final
Gateway_B ↗
```

### 2. Unbalanced Gateways ❌
**Problem:** Parallel fork without matching join

**NEVER:**
```
Parallel_Fork → Task_A → End  ❌ Missing sync
              → Task_B → End
```

**ALWAYS:**
```
Parallel_Fork → Task_A → Parallel_Join → End
              → Task_B ↗
```

### 3. Exclusive Gateway Without Default ❌
**Problem:** No default flow = potential deadlock

**ALWAYS:**
```xml
<bpmn:exclusiveGateway id="Gateway_Risk"
  default="Flow_Default">  <!-- ✅ Default defined -->
</bpmn:exclusiveGateway>
```

### 4. Implicit Gateway ❌
**Problem:** Task with conditional outgoing flows

**NEVER:**
```
Task → (conditional flow) → Task_A  ❌
    → (conditional flow) → Task_B
```

**ALWAYS:**
```
Task → Gateway → Task_A
             → Task_B
```

### 5. Missing Start/End Events ❌
**Problem:** Process without clear boundaries

**NEVER:**
```
[Task 1] → [Task 2] → [Task 3]  ❌
```

**ALWAYS:**
```
(Start) → [Task 1] → [Task 2] → (End)  ✅
```

**Full anti-pattern catalog:** See `references/antipatterns-full.md` (15+ patterns with examples)

**Gateway details:** See `references/gateway-complete-guide.md` (comprehensive rules, token semantics, all 5 types)

## Exception Handling Patterns

### Boundary Error Event
```xml
<bpmn:boundaryEvent id="Error_Payment" attachedToRef="Task_ProcessPayment">
  <bpmn:errorEventDefinition errorRef="Error_Payment"/>
</bpmn:boundaryEvent>
```

### Boundary Timer Event (Escalation)
```xml
<bpmn:boundaryEvent id="Timer_Escalate" attachedToRef="Task_Review"
  cancelActivity="false">
  <bpmn:timerEventDefinition>
    <bpmn:timeDuration>P2D</bpmn:timeDuration>
  </bpmn:timerEventDefinition>
</bpmn:boundaryEvent>
```

**More patterns:** See `references/advanced-patterns.md` (compensation, multi-instance, etc.)

## Process Complexity Guidelines

### Simple (≤10 elements)
- Linear or single decision point
- One actor implied
- No exception handling needed

### Medium (10-25 elements)
- Multiple decision points
- 2-3 actors/roles
- Basic exception handling

### Complex (25+ elements)
- Multiple parallel paths
- 4+ actors
- Multiple exception scenarios
- **Rule:** If > 30 elements, decompose into subprocesses

## Visual Layout (Quick Reference)

### Critical Rules

1. **Compact horizontal spacing (50-80px, NOT 180px)**
   - After start event: 52px
   - Standard gap: 55px
   - Task chains: 40px

2. **Multiple paths MUST have different Y coordinates**
   - Success path: lane_center
   - Rejection path: lane_center + 60px
   - Minimum vertical separation: 60px

3. **End events: Align on SAME line** ✨
   - All end events at same (X, Y) coordinate
   - Creates visual "finish line"

4. **Lane heights: 200-250px (fixed)**
   - Single flow: 200px
   - Two flows: 250px

5. **NO overlapping elements**
   - Run collision detection
   - Minimum 20px clearance

### Expected Results
- Pool width: 1800-2000px (25% more compact than old approach)
- Horizontal gaps: 40-80px
- No overlaps guaranteed

**Full algorithm:** See `references/intelligent-layout-algorithm-v3.md` (progressive spacing, waypoints, collision detection)

## Validation Checklist (MANDATORY)

Before finalizing ANY model, verify:

### Structural Soundness
- [ ] **Start Event present**
- [ ] **End Event(s) present** for each path
- [ ] **All paths lead to End** (no dead ends)
- [ ] **Gateways balanced** (fork = join, same type)
- [ ] **No orphan elements**

### Naming Conventions
- [ ] **Tasks: Verb + Object**
- [ ] **Gateways: Question**
- [ ] **Events: State/Trigger**
- [ ] **Consistent style** throughout

### Gateway Logic (CRITICAL)
- [ ] **EXCLUSIVE (XOR):**
  - [ ] Named as question
  - [ ] Default flow defined
  - [ ] Conditions mutually exclusive
  - [ ] Conditions complete
- [ ] **PARALLEL (AND):**
  - [ ] Fork has matching Join
  - [ ] Same number of paths
  - [ ] No conditions on outgoing
  - [ ] No multi-merge
- [ ] **INCLUSIVE (OR):**
  - [ ] At least one path guaranteed
  - [ ] No End events between split/join
- [ ] **No Implicit Gateways:** Tasks have ONE outgoing flow
- [ ] **No Multi-Merge:**
  - [ ] **PHASE 0 COMPLETED** (tabular analysis)
  - [ ] Tasks have ONE incoming
  - [ ] Gateways have ONE incoming (except merge gateways)
  - [ ] Exception: End events can have multiple

### Process Layout (PMA)
- [ ] **NO LANES** for single-organization processes
- [ ] **Straight happy path**
- [ ] **Roles indicated** on tasks
- [ ] **Compact diagram**

### Complexity
- [ ] **≤ 30 elements** per diagram
- [ ] **Subprocesses** for complex sections

### Error Handling
- [ ] **Boundary events** on critical tasks
- [ ] **Error paths** modeled
- [ ] **Timeouts** where needed

### Visual Layout
- [ ] **Left to right** flow
- [ ] **No overlapping** elements
- [ ] **Minimal crossing** flows
- [ ] **Clean waypoints** (2-3 per flow)

### XML Structure (CRITICAL)
- [ ] **Correct namespaces** (`bpmn:`, `bpmndi:`, `dc:`, `di:`)
- [ ] **BPMNDiagram structure**:
  - Single-process: `<bpmndi:BPMNPlane bpmnElement="Process_XXX">` (NO Pool shape)
  - Collaboration: `<bpmndi:BPMNPlane bpmnElement="Collaboration_XXX">` (WITH Pool shapes)
  - NEVER add Process shape when BPMNPlane references Process

## Output Checklist

Quick verification before delivery:

- [ ] Process has clear start trigger (named appropriately)
- [ ] All end events have semantic names (Approved, Rejected, Cancelled, Error)
- [ ] Every gateway has question-format name
- [ ] All tasks follow Verb + Object naming
- [ ] Parallel gateways balanced (fork = join)
- [ ] No orphan elements
- [ ] Exception paths modeled where critical
- [ ] Visual layout has no overlapping elements

## Camunda Modeler Integration

### Opening Files

**ALWAYS** after creating/editing .bpmn file:

```bash
open -a "Camunda Modeler" "[filename].bpmn"
```

User can then:
- Visually inspect diagram
- Make manual adjustments
- Add visual styling
- Test process execution
- Export as PNG/SVG

### Editing Existing Files

When user requests edits:

1. **Read existing file** first
2. **Parse XML** to identify elements, IDs, layout
3. **Make targeted changes** preserving IDs and layout
4. **Preserve DI elements** for visual consistency
5. **Update incrementally**

### Recommended Workflow

1. Claude generates/edits .bpmn
2. File opens in Camunda Modeler
3. User reviews visually
4. User requests changes if needed
5. Claude applies changes, reopens
6. Repeat until satisfied

**Detailed editing patterns:** See `references/editing-guide.md`

## Reference Documentation Index

### Core Templates
- **`references/xml-templates.md`** - XML structure templates for all BPMN elements (events, tasks, gateways, flows)

### Anti-Patterns & Quality
- **`references/antipatterns-full.md`** - Complete catalog of 15+ anti-patterns with detailed examples
- **`references/multi-merge-prevention.md`** - In-depth Phase 0 algorithm and Multi-Merge prevention

### Gateway Rules (CRITICAL)
- **`references/gateway-complete-guide.md`** ⭐ - Unified comprehensive guide (English + Russian terms) covering all 5 gateway types, balancing rules, token semantics, and anti-patterns

### Layout & Positioning
- **`references/intelligent-layout-algorithm-v3.md`** - Primary layout algorithm v3.0 (progressive spacing, waypoints)
- **`references/visual-best-practices.md`** - Collision prevention and visual clarity techniques

### Patterns & Examples
- **`references/examples-guide.md`** - 10+ structural patterns with detailed examples and use cases
- **`references/advanced-patterns.md`** - Complex patterns (compensation, escalation, multi-instance, subprocesses)

### Editing & Workflows
- **`references/editing-guide.md`** - Guide for modifying existing .bpmn files (adding tasks, changing gateways)
- **`references/workflow-guide.md`** - Step-by-step workflows for creation and editing processes

### Validation
- **`references/process-validation-report.md`** - Example validation reports for sample processes

### Quick Navigation

**Creating new process:**
1. Read: Analysis Framework (this file)
2. Use: `xml-templates.md` (syntax)
3. Use: `intelligent-layout-algorithm-v3.md` (positioning)
4. Check: `antipatterns-full.md` (avoid mistakes)

**Editing existing process:**
1. Read: `editing-guide.md` (modification patterns)
2. Use: `xml-templates.md` (syntax reference)
3. Check: `antipatterns-full.md` (validate changes)

**Gateway problems:**
1. Read: `gateway-rules-and-antipatterns.md` (English) OR `gateway-cheatsheet-ru.md` (Russian)
2. Check: `antipatterns-full.md` sections 6, 9, 12

**Complex patterns:**
1. Use: `advanced-patterns.md` (compensation, escalation, multi-instance)
2. Use: `examples-guide.md` (structural patterns)

---

**Version:** 2.1 (Optimized)
**Token savings:** ~9,000 tokens (66% reduction from v2.0)
**Maintained:** All critical rules and guidelines
