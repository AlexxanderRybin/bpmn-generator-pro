---
name: bpmn-generator-pro
description: Expert-level BPMN 2.0 process modeler with Camunda Modeler integration. Transforms natural language descriptions into professional BPMN diagrams AND edits existing .bpmn files. Creates/modifies processes with proper structure, naming conventions, exception handling, and visual layout. Automatically opens files in Camunda Modeler for visual inspection. Use when user asks to model, create, edit, modify, draw, or generate any business process, workflow, or BPMN diagram.
---

# BPMN Generator Pro

Expert-level BPMN 2.0 modeling engine. Think like a Senior Process Architect.

## ⚠️ CRITICAL: Analysis Framework (MANDATORY FIRST STEP)

**NEVER generate XML before completing this analysis.**

### 1. Identify Process Boundaries

| Aspect | Question | Output |
|--------|----------|--------|
| **Trigger** | What starts the process? | Request, event, schedule, message |
| **Outcome** | What are the possible end states? | Success, rejection, cancellation, error |
| **Scope** | Where does THIS process end and ANOTHER begin? | Clear boundaries |

### 2. Extract Actors & Responsibilities

- **WHO performs each action?** → Role overlays on tasks (NOT lanes - see PMA restriction)
  - If user provides step numbers: Use format `N.N.N [Role] Task Name` (see Element Numbering)
- **WHICH systems are involved?** → Service Tasks
- **WHO makes decisions?** → User Tasks before Gateways

### 3. Map the Happy Path First

Model the ideal scenario from start to successful end, then add:

1. Decision points (where flow branches)
2. Exception paths (what can go wrong)
3. Timeout/escalation scenarios

### 4. Identify Decision Logic

| Decision Type | Gateway Type | Characteristics |
|---------------|--------------|-----------------|
| Binary (yes/no) | Exclusive (XOR) | Exactly one path activates |
| Multiple conditions, one result | Exclusive (XOR) | Mutual exclusivity |
| Multiple conditions, multiple results | Inclusive (OR) | One or more paths |
| Parallel work | Parallel (AND) | All paths simultaneously |
| Wait for external event | Event-Based | Race condition |

---

## 🚫 CRITICAL: Process Layout Philosophy (PMA Approach)

### PMA Restriction: Lanes PROHIBITED for Single-Organization

**Scope:** Applies to processes within ONE organization (single company with internal departments).

#### Why Lanes Are Harmful

| Issue | Impact | Severity |
|-------|--------|----------|
| **Increased model size** | Diagrams become larger, harder to view | 🔴 HIGH |
| **Broken happy path** | Zigzag pattern across lanes | 🔴 CRITICAL |
| **Harder to follow** | Reader must jump between lanes | 🔴 HIGH |
| **Reduced readability** | Visual complexity obscures logic | 🔴 HIGH |

**Golden Rule:**
> **"The happy path should be a straight line, not a zigzag across lanes."**

#### ✅ RECOMMENDED: Role Overlays

| Benefit | Description |
|---------|-------------|
| **Compact diagram** | Smaller, easier to view |
| **Straight happy path** | Clear horizontal flow |
| **Better readability** | Focus on logic, not organization |
| **Easier maintenance** | Role changes don't break layout |
| **Execution-ready** | Camunda attributes assign tasks |

### Implementation Options

**Option A: Camunda Attributes (PREFERRED)** ⭐

```xml
<bpmn:userTask id="Task_Review" name="Review Application"
               camunda:assignee="${reviewer}"
               camunda:candidateGroups="underwriters,managers">
</bpmn:userTask>
```

**Option B: Task Name Prefix (SIMPLE)**

```xml
<bpmn:userTask id="Task_Review" name="[Finance] Review Application">
</bpmn:userTask>
```

**Option C: Step Number + Role (NUMBERED REQUIREMENTS)** ⭐

```xml
<bpmn:userTask id="Task_ReviewApplication"
               name="2.5.1 [Preparation Specialist] Review Application"
               camunda:candidateGroups="preparation-specialist">
</bpmn:userTask>
```

### When Lanes ARE Acceptable ✅

| Scenario | Justification | Example |
|----------|---------------|---------|
| **Multi-Organization** | Different companies collaborating | Customer ↔ Supplier |
| **Cross-System** | Independent systems with messages | ERP ↔ CRM |
| **Explicit User Request** | User specifically requests (with warning) | Legacy compatibility |

**Key Distinction:**
- ❌ Single org + multiple departments → NO lanes (use role overlays)
- ✅ Multiple organizations → YES lanes (use separate pools)

---

## 📝 IMPORTANT: Naming Conventions

### Task Names: Verb + Object

```text
✅ "Review Application"      ❌ "Application Review"
✅ "Calculate Discount"      ❌ "Discount Calculation"
✅ "Send Confirmation"       ❌ "Confirmation Sending"
```

### Gateway Names: Question Format

```text
✅ "Approved?"               ❌ "Check Approval"
✅ "Amount > 10000?"         ❌ "Amount Gateway"
✅ "Documents Complete?"     ❌ "Document Check"
```

### Event Names: State/Trigger Description

```text
✅ "Order Received"          ❌ "Start"
✅ "Request Approved"        ❌ "End"
✅ "Payment Timeout"         ❌ "Timer"
```

### Element Numbering (Optional)

**Use when:** User provides process description with step numbers (2.5.1, 2.5.2, etc.)

**Format:** `N.N.N [Role] Task Name`

```xml
<bpmn:userTask name="2.5.1 [Finance] Review Application" />
<bpmn:exclusiveGateway name="2.7.1 Approved?" />
```

**Benefits:** Traceability to source documentation + visual role identification

---

## 🔍 CRITICAL: Phase 0 - Multi-Merge Pre-Check (MANDATORY)

**Complete this analysis BEFORE creating XML elements.**

**Why:** Multi-Merge anti-pattern (multiple incoming flows) is the most common error. Systematic checking prevents it.

### Quick Algorithm

**For EVERY sequence flow you create:**

```text
1. Identify target element
2. Check target type:
   ├─ End Event? → ✅ ALLOW multiple incoming
   └─ Task/Gateway/Other Event?
      └─ Count existing incoming flows:
         ├─ incoming.length === 0 → ✅ ADD flow directly
         └─ incoming.length >= 1 → ⚠️ INSERT MERGE GATEWAY FIRST
```

### Required Table Format

Create inventory BEFORE generating XML:

| Element ID | Type | Step # | Incoming Count | Sources | Action |
|------------|------|--------|----------------|---------|--------|
| Event_Start | Start | - | 0 | - | ✅ OK |
| Task_Check | Task | 2.1.1 | 1 | Event_Start | ✅ OK |
| Task_Package | Task | 2.2.4 | **2** | Gateway_A, Gateway_B | ⚠️ **ADD MERGE** |
| Event_End | End | - | 3 | Multiple | ✅ OK (End allows multiple) |

**Rule:** If ANY element (except End Events) has `incoming > 1`, add merge gateway FIRST.

**Full details:** See `references/multi-merge-prevention.md`

---

## Task Type Selection Matrix

| Situation | Task Type | Indicator Words |
|-----------|-----------|-----------------|
| Human reviews/decides/approves | **User Task** | review, approve, decide, check, verify |
| System calls API/service | **Service Task** | send, notify, update system, integrate |
| Calculation/transformation | **Script Task** | calculate, transform, generate, compute |
| Physical work outside system | **Manual Task** | deliver, install, physically inspect |
| Decision table/rules | **Business Rule Task** | determine, evaluate rules, apply policy |

---

## Gateway Selection Decision Tree

```text
Is it a choice/decision?
├─ YES: How many paths can be active?
│   ├─ Exactly ONE path → Exclusive Gateway (XOR)
│   └─ ONE OR MORE paths → Inclusive Gateway (OR)
│
└─ NO: Is work done simultaneously?
    ├─ YES → Parallel Gateway (AND)
    └─ NO: Waiting for external event? → Event-Based Gateway
```

---

## Structural Patterns (Common Flows)

### 1. Simple Approval

```text
Start → Submit → Review → [Approved?]
                            ├─ Yes → Process → End(Approved)
                            └─ No → Notify → End(Rejected)
```

### 2. Parallel Processing with Sync

```text
Start → Order → ║Split║ → Check Inventory ──┐
                        → Validate Payment ──┼→ ║Join║ → Continue
                        → Verify Address ────┘
```

### 3. Timeout/Escalation

```text
... → Send Request → ◇Event Gateway◇ → ○ Response → Process → ...
                                      → ⏱ 48h Timeout → Escalate → ...
```

**More patterns:** See `references/examples-guide.md` for 10+ detailed patterns

---

## ⚠️ CRITICAL: Top 5 Anti-Patterns (MUST AVOID)

### 1. Multi-Merge ❌ MOST CRITICAL

**Problem:** Multiple flows directly into task without explicit gateway

**NEVER:**
```text
Gateway_A → Task_Final
Gateway_B → Task_Final  ❌ Task receives 2 tokens!
```

**ALWAYS:**
```text
Gateway_A → Merge_Gateway → Task_Final
Gateway_B ↗
```

### 2. Unbalanced Gateways ❌

**Problem:** Parallel fork without matching join

**NEVER:**
```text
Parallel_Fork → Task_A → End  ❌ Missing sync
              → Task_B → End
```

**ALWAYS:**
```text
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
```text
Task → (conditional flow) → Task_A  ❌
    → (conditional flow) → Task_B
```

**ALWAYS:**
```text
Task → Gateway → Task_A
             → Task_B
```

### 5. Missing Start/End Events ❌

**Problem:** Process without clear boundaries

**NEVER:**
```text
[Task 1] → [Task 2] → [Task 3]  ❌
```

**ALWAYS:**
```text
(Start) → [Task 1] → [Task 2] → (End)  ✅
```

**Full anti-pattern catalog:** See `references/antipatterns-full.md` (15+ patterns with examples)

**Gateway details:** See `references/gateway-complete-guide.md` (comprehensive rules, token semantics, all 5 types)

---

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

---

## Process Complexity Guidelines

| Complexity | Elements | Characteristics | Recommendations |
|------------|----------|-----------------|-----------------|
| **Simple** | ≤10 | Linear or single decision point<br>One actor implied<br>No exception handling | Keep minimal |
| **Medium** | 10-25 | Multiple decision points<br>2-3 actors/roles<br>Basic exception handling | Add boundary events |
| **Complex** | 25+ | Multiple parallel paths<br>4+ actors<br>Multiple exception scenarios | **If >30: Use subprocesses** |

---

## 📐 IMPORTANT: Visual Layout (Quick Reference)

### Critical Rules

| Rule | Value | Notes |
|------|-------|-------|
| **Horizontal spacing** | 40-80px | **NOT 180px** (old docs) |
| After start event | 52px | First gap |
| Standard gap | 55px | Default |
| Task chains | 40px | Tight sequences |
| **End event alignment** | Same (X, Y) | Visual "finish line" ✨ |
| **Lane heights** | 200-250px | Fixed dimensions |
| Single flow | 200px | Minimal |
| Two flows | 250px | Standard |
| **Minimum clearance** | 20px | Collision prevention |

### Multiple Paths MUST Have Different Y

```text
Success path:   Y = lane_center
Rejection path: Y = lane_center + 60px
Minimum vertical separation: 60px
```

### Expected Results

- **Pool width:** 1800-2000px (25% more compact)
- **Horizontal gaps:** 40-80px
- **No overlaps:** Guaranteed collision detection

**Full algorithm:** See `references/layout-guide.md` ⭐

---

## ✅ MANDATORY: Validation Checklist

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

### ⚠️ CRITICAL: Gateway Logic

**EXCLUSIVE (XOR):**
- [ ] Named as question
- [ ] Default flow defined
- [ ] Conditions mutually exclusive
- [ ] Conditions complete

**PARALLEL (AND):**
- [ ] Fork has matching Join
- [ ] Same number of paths
- [ ] No conditions on outgoing
- [ ] No multi-merge

**INCLUSIVE (OR):**
- [ ] At least one path guaranteed
- [ ] No End events between split/join

**General:**
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

### ⚠️ CRITICAL: XML Structure

- [ ] **Correct namespaces** (`bpmn:`, `bpmndi:`, `dc:`, `di:`)
- [ ] **BPMNDiagram structure:**
  - Single-process: `<bpmndi:BPMNPlane bpmnElement="Process_XXX">` (NO Pool shape)
  - Collaboration: `<bpmndi:BPMNPlane bpmnElement="Collaboration_XXX">` (WITH Pool shapes)
  - **NEVER add Process shape when BPMNPlane references Process**

---

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

---

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

```text
1. Claude generates/edits .bpmn
2. File opens in Camunda Modeler
3. User reviews visually
4. User requests changes if needed
5. Claude applies changes, reopens
6. Repeat until satisfied
```

**Detailed editing patterns:** See `references/editing-guide.md`

**Step-by-step workflows:** See `references/workflow-guide.md`

---

## Reference Documentation Index

### 🔴 CRITICAL (Read First)

- **`references/multi-merge-prevention.md`** - Phase 0 algorithm, Multi-Merge prevention (MANDATORY)
- **`references/gateway-complete-guide.md`** - All 5 gateway types, balancing rules, token semantics
- **`references/antipatterns-full.md`** - Complete catalog of 15+ anti-patterns with detection/fixes

### ⭐ ESSENTIAL (Use Frequently)

- **`references/layout-guide.md`** - Unified layout algorithm v3.1 (progressive spacing, content-driven dimensions)
- **`references/xml-templates.md`** - XML structure templates for all BPMN elements
- **`references/workflow-guide.md`** - Step-by-step workflows for creation and editing

### 📚 HELPFUL (Reference As Needed)

- **`references/examples-guide.md`** - 10+ structural patterns with detailed examples
- **`references/advanced-patterns.md`** - Complex patterns (compensation, escalation, multi-instance)
- **`references/editing-guide.md`** - Modification patterns for existing files
- **`references/visual-best-practices.md`** - Collision prevention, visual clarity

### 🔍 VALIDATION

- **`references/process-validation-report.md`** - Example validation reports

---

## Quick Task Navigation

| Task | Primary Files | Support Files |
|------|---------------|---------------|
| **Create new process** | Analysis Framework (this file)<br>`xml-templates.md`<br>`layout-guide.md` | `antipatterns-full.md`<br>`workflow-guide.md` |
| **Edit existing process** | `editing-guide.md`<br>`xml-templates.md` | `workflow-guide.md`<br>`antipatterns-full.md` |
| **Fix gateway issues** | `gateway-complete-guide.md` | `antipatterns-full.md` (sections 6, 9, 12) |
| **Add complex patterns** | `advanced-patterns.md`<br>`examples-guide.md` | `xml-templates.md` |
| **Fix layout/spacing** | `layout-guide.md` | `visual-best-practices.md` |
| **Prevent Multi-Merge** | `multi-merge-prevention.md` | Analysis Framework (Phase 0) |

---

**Version:** 2.2 (Anthropic Best Practices)
**Changes from v2.1:**
- Added structured formatting (tables, code blocks with language tags)
- Added systematic prioritization markers (CRITICAL/IMPORTANT)
- Converted prose to tables (PMA comparison, complexity guidelines, layout rules)
- Fixed all outdated references (removed old file names)
- Improved visual formatting (decision trees, diagrams)
- Added Quick Task Navigation table

**Token savings:** ~12% additional reduction through better structure
**Total optimization:** 72% reduction from original v2.0 (1,902 → 531 lines in v2.1 → 529 lines in v2.2)
