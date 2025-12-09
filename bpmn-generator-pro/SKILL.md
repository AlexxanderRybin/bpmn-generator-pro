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

**WRONG Approach (Using Lanes):**
```xml
<bpmn:process id="Process_1">
  <bpmn:laneSet>
    <bpmn:lane id="Lane_Sales" name="Sales Department">
      <bpmn:flowNodeRef>Task_CreateQuote</bpmn:flowNodeRef>
    </bpmn:lane>
    <bpmn:lane id="Lane_Finance" name="Finance Department">
      <bpmn:flowNodeRef>Task_ApprovePrice</bpmn:flowNodeRef>
    </bpmn:lane>
    <bpmn:lane id="Lane_Operations" name="Operations">
      <bpmn:flowNodeRef>Task_FulfillOrder</bpmn:flowNodeRef>
    </bpmn:lane>
  </bpmn:laneSet>
  <!-- Happy path zigzags: Sales → Finance → Operations → Sales... -->
</bpmn:process>
```

**Visual problem:**
```
┌─────────────────────────────────────────────┐
│ Sales       │ Create Quote → ↓             │
├─────────────┼──────────────────────────────┤
│ Finance     │              Approve → ↓     │  ❌ Zigzag path
├─────────────┼──────────────────────────────┤
│ Operations  │                      Fulfill │
└─────────────┴──────────────────────────────┘
```

### RECOMMENDED Approach: Role Overlays ✅

**Instead:** Indicate roles directly on tasks using Camunda attributes or text annotations.

**CORRECT Approach (No Lanes):**
```xml
<bpmn:process id="Process_1">
  <!-- No laneSet - single flat structure -->

  <bpmn:userTask id="Task_CreateQuote" name="Create Quote" camunda:assignee="${salesRep}" camunda:candidateGroups="sales">
    <!-- Role indicated via attributes -->
  </bpmn:userTask>

  <bpmn:userTask id="Task_ApprovePrice" name="Approve Price" camunda:assignee="${financeManager}" camunda:candidateGroups="finance">
    <!-- Role indicated via attributes -->
  </bpmn:userTask>

  <bpmn:userTask id="Task_FulfillOrder" name="Fulfill Order" camunda:assignee="${opsTeam}" camunda:candidateGroups="operations">
    <!-- Role indicated via attributes -->
  </bpmn:userTask>
</bpmn:process>
```

**Visual benefit:**
```
Start → Create Quote → Approve Price → Fulfill Order → End
        [Sales]        [Finance]        [Operations]        ✅ Straight line
```

**Benefits of Role Overlays:**
- ✅ **Compact diagram** - smaller, easier to view
- ✅ **Straight happy path** - main flow is a clear horizontal line
- ✅ **Better readability** - focus on process logic, not organizational structure
- ✅ **Easier maintenance** - role changes don't require layout restructuring
- ✅ **Execution-ready** - Camunda attributes directly assign tasks

### Implementation Guidelines

**1. Use Single Pool (No Lanes):**
```xml
<bpmn:collaboration id="Collaboration_1">
  <bpmn:participant id="Participant_Process" name="Order Processing" processRef="Process_OrderHandling" />
  <!-- Single participant, no lane subdivision -->
</bpmn:collaboration>
```

**2. Indicate Roles on Tasks:**

**Option A: Camunda Attributes (Preferred for executable processes)**
```xml
<bpmn:userTask id="Task_Review" name="Review Application"
               camunda:assignee="${reviewer}"
               camunda:candidateGroups="underwriters,managers">
  <!-- Role: underwriters or managers -->
</bpmn:userTask>
```

**Option B: Text Annotations (For documentation diagrams)**
```xml
<bpmn:userTask id="Task_Review" name="Review Application">
  <!-- Add text annotation in visual layer -->
</bpmn:userTask>

<bpmn:textAnnotation id="Annotation_Role">
  <bpmn:text>Role: Underwriter</bpmn:text>
</bpmn:textAnnotation>

<bpmn:association sourceRef="Task_Review" targetRef="Annotation_Role" />
```

**Option C: Task Name Prefix (Simple documentation)**
```xml
<bpmn:userTask id="Task_Review" name="[Finance] Review Application">
  <!-- Role indicated in task name -->
</bpmn:userTask>
```

**Option D: ⭐ Step Number + Role (When user provides numbered steps)**
```xml
<bpmn:userTask id="Task_Review" name="2.5.1 [Finance] Review Application"
               camunda:candidateGroups="finance">
  <!-- Combines step traceability with role overlay -->
</bpmn:userTask>
```
**When to use:** User provides process documentation with step numbers (2.5.1, 2.5.2, etc.)
**See:** Element Numbering section for full format specification

**3. Layout Tasks Left-to-Right:**
```
Position elements in happy path sequence:
X=200 → X=350 → X=500 → X=650 (straight line)

NOT zigzagging across lanes:
Y=100 → Y=300 → Y=500 → Y=100 (broken line)
```

### When Lanes ARE Acceptable ✅

Lanes are ALLOWED (and recommended) for:

1. **Multi-Organization Processes** - Different companies/organizations collaborating
   - Example: Customer ↔ Supplier ↔ Logistics Provider
   - Each organization = separate Pool with its own lanes
   - Connected via Message Flows

2. **Cross-System Collaboration** - Independent systems exchanging messages
   - Example: E-commerce Platform ↔ Payment Gateway ↔ Shipping Service
   - Each system = separate Participant

3. **Explicit User Requirement** - User specifically requests lane-based visualization
   - Provide warning about readability impact
   - Suggest role overlay alternative

**Key Distinction:**
- ❌ Single organization, multiple departments → NO lanes (use role overlays)
- ✅ Multiple organizations → YES lanes (use separate pools)

**Example of valid lane use (different organizations):**
```xml
<bpmn:collaboration>
  <bpmn:participant id="Customer" name="Customer" processRef="Process_Customer" />
  <bpmn:participant id="Supplier" name="Supplier" processRef="Process_Supplier" />
  <!-- Message flows between different organizations -->
  <bpmn:messageFlow sourceRef="Task_SendOrder" targetRef="Task_ReceiveOrder" />
</bpmn:collaboration>
```

### Decision Tree: Lanes vs. Role Overlays

```
Need to show WHO performs tasks?
│
├─ Different organizations/systems?
│  └─ YES → Use separate Participants (pools) + message flows
│
├─ Single organization, different departments?
│  └─ User explicitly requested lanes?
│     ├─ YES → Use lanes (but warn about readability impact)
│     └─ NO → ✅ Use Role Overlays (recommended)
│
└─ Just documenting process flow?
   └─ ✅ Use Role Overlays (keep it simple)
```

### Validation Rule

```
✅ CORRECT: Single pool, no lanes, roles in task attributes
❌ WRONG: Multiple lanes for different departments in same organization
⚠️  ACCEPTABLE (with justification): Lanes for cross-organizational collaboration
```

**Remember:**
> **"The happy path should be a straight line, not a zigzag across lanes."**

## Naming Conventions (CRITICAL)

### Task Names: Verb + Object
```
✅ "Review Application"      ❌ "Application Review"
✅ "Calculate Discount"      ❌ "Discount Calculation"  
✅ "Send Confirmation"       ❌ "Confirmation Sending"
✅ "Verify Documents"        ❌ "Document Verification"
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
✅ "Process Cancelled"       ❌ "Cancel End"
```

### Flow Labels: Condition Result
```
✅ "Yes" / "No"              ❌ "approved=true"
✅ "VIP Customer"            ❌ "if VIP"
✅ "Amount ≤ 1000"           ❌ "small amount flow"
```

### Element Numbering (Optional - Use when user provides numbered steps)

**When to use:** If user provides process description with step numbers (like 1.1.1, 1.1.2, 1.2.1), preserve this numbering for traceability.

**How to apply:**

1. **As XML Comments** (Keeps diagram clean):
```xml
<!-- Step 1.1.1 -->
<bpmn:userTask id="Task_IdentifyWork" name="Identify Work Requirement">
  ...
</bpmn:userTask>

<!-- Step 1.1.2 - Decision by Review Committee -->
<bpmn:exclusiveGateway id="Gateway_CheckClear" name="Notification Clear?">
  ...
</bpmn:exclusiveGateway>
```

2. **As Prefix in Task Name** (Visual traceability):
```xml
<bpmn:userTask id="Task_IdentifyWork" name="1.1.1 Identify Work Requirement">
  ...
</bpmn:userTask>
```

3. **⭐ RECOMMENDED: Step Number + Role + Name** (Best for complex processes):

**Format:** `N.N.N [Role] Task Name`

This combines step numbering with role overlays for maximum traceability and clarity.

**For Tasks:**
```xml
<!-- English example -->
<bpmn:userTask id="Task_ReviewApplication"
               name="2.5.1 [Preparation Specialist] Review Application"
               camunda:candidateGroups="preparation-specialist">
  <bpmn:incoming>Flow_1</bpmn:incoming>
  <bpmn:outgoing>Flow_2</bpmn:outgoing>
</bpmn:userTask>

<!-- Russian example -->
<bpmn:userTask id="Task_CreateWorkOrder"
               name="2.5.3 [Специалист по подготовке] Создать рабочий наряд"
               camunda:candidateGroups="preparation-specialist">
  <bpmn:incoming>Flow_3</bpmn:incoming>
  <bpmn:outgoing>Flow_4</bpmn:outgoing>
</bpmn:userTask>
```

**For Gateways:**
```xml
<!-- Gateways: Only step number + question (no role needed) -->
<bpmn:exclusiveGateway id="Gateway_MaterialsRequired"
                        name="2.8.2 Требуются материалы или услуги?"
                        default="Flow_Default">
  <bpmn:incoming>Flow_5</bpmn:incoming>
  <bpmn:outgoing>Flow_Yes</bpmn:outgoing>
  <bpmn:outgoing>Flow_No</bpmn:outgoing>
</bpmn:exclusiveGateway>
```

**When to use this format:**
- ✅ User provides numbered process steps (2.5.1, 2.5.2, etc.)
- ✅ Multiple roles/actors in single organization (no lanes per PMA)
- ✅ Need visual role indication on diagram
- ✅ Process documentation requires step-to-BPMN mapping

**Structure:**
- **Tasks**: `Step# [Role] Action` → `2.5.1 [Sales] Create Quote`
- **Gateways**: `Step# Question` → `2.7.1 Approved?`
- **Events**: Natural language (no numbering needed) → `Получена заявка`

**Benefits:**
- ✅ Step number = Traceability to source documentation
- ✅ [Role] = Visual actor identification (aligns with PMA role overlays)
- ✅ Action/Question = Clear task purpose
- ✅ Single line contains all critical metadata
- ✅ No need for separate lanes or annotations

4. **In Documentation Element** (Most formal):
```xml
<bpmn:userTask id="Task_IdentifyWork" name="Identify Work Requirement">
  <bpmn:documentation>Process Step: 1.1.1</bpmn:documentation>
  ...
</bpmn:userTask>
```

**Benefits:**
- ✅ Easy mapping between requirements and BPMN
- ✅ Simplifies validation and reviews
- ✅ Helps identify missing steps
- ✅ Combines role visibility with step tracking

**Default Approach:**
- **Simple processes (single role)**: Use Option 1 (XML comments)
- **Complex processes (multiple roles + numbered steps)**: Use Option 3 ⭐ (Step + Role + Name)
- **Formal documentation**: Use Option 4 (Documentation element)

## Phase 0: Multi-Merge Pre-Check Analysis (MANDATORY) 🔍

**CRITICAL:** Complete this systematic analysis BEFORE creating any XML elements.

**Why this phase exists:** Multi-Merge anti-pattern (multiple incoming flows to a single element) is the most commonly overlooked error. Checking "by memory" leads to missed cases. This phase ensures EVERY element is validated.

### Step-by-Step Tabular Analysis:

**1. List ALL elements in the process:**

Create a comprehensive inventory:
- Start event
- All tasks (with their step numbers if provided: 2.1.1, 2.2.2, etc.)
- All gateways (decision points, splits, joins)
- All end events
- Any intermediate events

**2. For EACH element, count incoming sequence flows:**

Create a table in this exact format:

| Element ID | Element Type | Step # | Incoming Flows Count | Sources | Action Required |
|------------|--------------|--------|---------------------|---------|-----------------|
| Event_Start | Start Event | - | 0 | - | ✅ OK (Start events have no incoming) |
| Task_CheckReq | Task | 2.1.1 | 1 | Event_Start | ✅ OK |
| Gateway_NewReq | Gateway | 2.1.2 | 1 | Task_CheckReq | ✅ OK |
| Gateway_LibAvail | Gateway | 2.2.1 | 1 | Gateway_NewReq | ✅ OK |
| Task_UseLib | Task | 2.2.2 | 1 | Gateway_LibAvail | ✅ OK |
| Gateway_Correction | Gateway | 2.2.3 | 1 | Task_UseLib | ✅ OK |
| Task_CreatePackage | Task | 2.2.4 | **2** | Gateway_LibAvail(No), Gateway_Correction(Yes) | ⚠️ **ADD MERGE GATEWAY** |
| Gateway_Materials | Gateway | 2.2.5 | **2** | Gateway_Correction(No), Task_CreatePackage | ⚠️ **ADD MERGE GATEWAY** |
| Event_Complete | End Event | - | 1 | Task_CreatePlan | ✅ OK (End events can have multiple) |

**3. For ANY element with incoming count > 1 (except End Events):**

When you find an element with `incoming.length > 1`:

a) **MANDATORY: Add merge gateway BEFORE this element**
   - ID: `Gateway_Merge_[Purpose]` (e.g., `Gateway_Merge_BeforePackage`)
   - Type: Exclusive (XOR) for mutually exclusive paths, Parallel (AND) for synchronization
   - Name: Leave empty (merge gateways typically have no label)

b) **Update your element list:**
   - Add new merge gateway to the table
   - Update target element's incoming count to 1 (now comes from merge gateway)
   - Update merge gateway's incoming count to match original target's count

c) **Update flow mapping:**
   - All flows that previously pointed to target → now point to merge gateway
   - New single flow: merge gateway → target element

**Example correction:**

BEFORE (Multi-Merge detected):
```
Gateway_LibAvail --No--> Task_CreatePackage
Gateway_Correction --Yes--> Task_CreatePackage
(Task_CreatePackage has 2 incoming flows ❌)
```

AFTER (Merge gateway added):
```
Gateway_LibAvail --No--> Gateway_Merge_BeforePackage
Gateway_Correction --Yes--> Gateway_Merge_BeforePackage
Gateway_Merge_BeforePackage ---> Task_CreatePackage
(Task_CreatePackage has 1 incoming flow ✅)
```

**4. Verify result - Final validation:**

Check your updated table:
- ✅ ALL tasks have exactly ONE incoming flow
- ✅ ALL gateways (except merge gateways) have exactly ONE incoming flow
- ✅ ALL events (except End Events and merge gateways) have exactly ONE incoming flow
- ✅ ONLY End Events and Merge Gateways have multiple incoming flows

**5. ONLY AFTER this table is complete and validated:**
   - ✅ Proceed to creating XML elements
   - ✅ You now have a complete list including all necessary merge gateways
   - ✅ You've prevented Multi-Merge anti-pattern before it enters the XML

### Common Mistakes This Phase Prevents:

❌ **Checking only gateways** - Tasks can also have Multi-Merge!
❌ **Checking "by memory"** - Easy to miss elements in complex processes
❌ **Skipping the table** - "I'll just be careful" doesn't work consistently
❌ **Validating after XML creation** - Fixing is harder than preventing

### Real Example from pm-preparation.bpmn:

**What went wrong:** Task_CreateUpdatePackage (2.2.4) had 2 incoming flows:
1. From Gateway_LibraryAvailable (No)
2. From Gateway_CorrectionNeeded (Yes)

**Why it was missed:** Checked gateways 2.2.5 and 2.2.7 for Multi-Merge, but didn't check task 2.2.4 systematically.

**How Phase 0 would have prevented it:** Tabular analysis would show:
```
Task_CreateUpdatePackage | Task | 2.2.4 | 2 | Gateway_LibAvail, Gateway_Correction | ⚠️ ADD MERGE GATEWAY
```

**Lesson:** Trust the systematic process, not your memory.

## Task Type Selection Matrix

| Situation | Task Type | Indicator Words |
|-----------|-----------|-----------------|
| Human reviews/decides/approves | User Task | review, approve, decide, check, verify (by person) |
| System calls API/service | Service Task | send, notify, update system, integrate, call |
| Calculation/transformation | Script Task | calculate, transform, generate, compute |
| Physical work outside system | Manual Task | deliver, install, physically inspect |
| Decision table/rules | Business Rule Task | determine, evaluate rules, apply policy |
| Send message to participant | Send Task | notify external party, send to customer |
| Wait for external response | Receive Task | wait for reply, await confirmation |

## Gateway Selection Decision Tree

```
Is it a choice/decision?
├─ YES: How many paths can be active?
│   ├─ Exactly ONE path → Exclusive Gateway (XOR)
│   │   Examples: approved/rejected, valid/invalid
│   │
│   └─ ONE OR MORE paths → Inclusive Gateway (OR)
│       Examples: select delivery options, choose add-ons
│
└─ NO: Is work done simultaneously?
    ├─ YES → Parallel Gateway (AND)
    │   Examples: notify HR AND Finance, check stock AND credit
    │
    └─ NO: Waiting for external event?
        └─ YES → Event-Based Gateway
            Examples: wait for payment OR timeout, response OR cancel
```

## Structural Patterns

### Pattern 1: Simple Approval
```
Start → Submit Request → Review Request → [Approved?]
                                            ├─ Yes → Process Request → Notify Success → End(Approved)
                                            └─ No → Notify Rejection → End(Rejected)
```

### Pattern 2: Multi-Level Approval
```
Start → Submit → Manager Review → [Approved?]
                                    ├─ No → End(Rejected)
                                    └─ Yes → [Amount > Threshold?]
                                              ├─ No → Process → End
                                              └─ Yes → Director Review → [Approved?]
                                                                          ├─ Yes → Process → End
                                                                          └─ No → End(Rejected)
```

### Pattern 3: Parallel Processing with Sync
```
Start → Receive Order → ║Split║ → Check Inventory ──────┐
                                → Validate Payment ──────┼→ ║Join║ → [All OK?] → ...
                                → Verify Address ────────┘
```

### Pattern 4: Timeout/Escalation
```
... → Send Request → ◇Event Gateway◇ → ○ Response Received → Process Response → ...
                                      → ⏱ 48h Timeout → Escalate → ...
```

### Pattern 5: Loop with Exit Condition
```
Start → Prepare Document → Review → [Approved?]
                             ↑         ├─ Yes → Finalize → End
                             │         └─ No → [Attempts < 3?]
                             │                   ├─ Yes → Revise ─┘
                             │                   └─ No → End(Failed)
                             └─────────────────────────────┘
```

### Pattern 6: Subprocess for Reusable Logic
```
Main: Start → Collect Data → [Payment Subprocess] → Ship Order → End

Payment Subprocess: Start → Calculate Total → Process Payment → [Success?]
                                                                  ├─ Yes → End
                                                                  └─ No → Retry Logic → ...
```

## Anti-Patterns (NEVER DO)

### ❌ Implicit Gateway
```
BAD:  Task → Task A (if approved)
           → Task B (if rejected)
GOOD: Task → Gateway → Task A
                     → Task B
```

### ❌ Unbalanced Gateways
```
BAD:  Gateway(fork) → A → End
                    → B → End
GOOD: Gateway(fork) → A → Gateway(merge) → End
                    → B ↗
```

### ❌ Multiple End Events Without Semantic Difference
```
BAD:  Branch A → End1
      Branch B → End2  (both mean "success")
GOOD: Branch A → Merge Gateway → End(Success)
      Branch B ↗
```

### ❌ Decision After Decision Without Task
```
BAD:  Gateway1 → Gateway2 → ...
GOOD: Gateway1 → Clarifying Task → Gateway2 → ...
```

### ❌ Lanes Without Purpose
```
BAD:  Single lane containing everything
GOOD: No lanes (simple process) OR meaningful role separation
```

### ❌ Vague Naming
```
BAD:  "Process Data", "Handle Request", "Do Something"
GOOD: "Validate Customer Data", "Approve Loan Request", "Calculate Shipping Cost"
```

### ❌ Logically Incomplete Loop (Loop without State Change)
```
BAD:  Gateway "Data Valid?" → No → Task "Get More Data" → Gateway "Data Valid?"
      ❌ PROBLEM: Who updates the data? How does state change?

GOOD: Gateway "Data Valid?" → No → Task "Get More Data"
                                  → Task "Update Record with New Data"
                                  → Gateway "Data Valid?"
      ✅ State changes, loop makes logical sense
```

**CRITICAL VALIDATION for EVERY Loop:**

When creating a loop pattern (any flow that goes backwards), VERIFY:

1. ✅ **State Change**: Is there a clear action that modifies the condition?
   - **BAD**: `Check → [Invalid?] → Get Info → Check`
     - *Who updates what? Magic loop!*
   - **GOOD**: `Check → [Invalid?] → Get Info → Update Document → Check`
     - *Clear: document is updated between checks*

2. ✅ **Actor Transition**: If different actors are involved, is the handoff explicit?
   - **BAD**: `[Committee Reviews] → Not Clear → [Requester Gets Info] → [Committee Reviews]`
     - *How does committee know requester is done?*
   - **GOOD**: `[Committee Reviews] → Not Clear → [Requester Gets Info] → [Requester Resubmits] → [Committee Reviews]`
     - *Explicit resubmission triggers new review*

3. ✅ **Exit Condition**: Can the loop terminate? Is there a max attempts counter?
   - **BAD**: Infinite loop without escape
   - **GOOD**: Loop with attempt counter or timeout

**Loop Pattern Template:**
```
Gateway_Decision (Actor A)
  ↓ Condition Not Met
Task_ActionToFix (Actor B)
  ↓
Task_StateChange (Actor B - update/resubmit/notify)
  ↓
Gateway_Decision (Actor A - re-evaluate)
```

**Example from Real Process:**
```
❌ WRONG:
  Gateway "Clear?" (Committee) → No → Task "Get Info" (Requester) → Gateway "Clear?" (Committee)
  Problem: No explicit resubmission!

✅ CORRECT:
  Gateway "Clear?" (Committee)
    → No → Task "Get Info" (Requester)
    → Task "Update and Resubmit Notification" (Requester)
    → Gateway "Clear?" (Committee)
  Fixed: Explicit state change + resubmission!
```

**Detection Rule:**
- If you see loop pattern and can't answer "What changed between iterations?" → **NOTIFY USER** (DO NOT auto-fix!)

**CRITICAL: When Detecting Logically Incomplete Loop:**

🚫 **DO NOT:**
- Automatically add missing tasks
- Modify the process structure without user approval
- Make assumptions about what the missing step should be

✅ **DO:**
1. **Detect**: Identify the loop pattern and missing state change
2. **Document**: Explain what's missing and why it's a problem
3. **Report**: Create clear message to user showing:
   - Current structure (with the gap)
   - What's missing (state-changing task)
   - Why it's problematic (magic transition)
   - Suggested fix (example of what could be added)
4. **Wait**: Let user decide whether and how to fix it

**Example Report to User:**
```
⚠️ Logically Incomplete Loop Detected

Current structure:
  Gateway "Clear?" → No → Task "Get Info" → Gateway "Clear?"

Problem: No explicit state change between iterations.

Question: What happens between getting info and re-checking?

Suggested fix (for your consideration):
  Add task: "Update Notification and Resubmit"

Would you like me to add this task, or handle it differently?
```

## Exception Handling Patterns

### Boundary Error Event
Attach to tasks that can fail:
```xml
<bpmn:boundaryEvent id="Error_PaymentFailed" attachedToRef="Activity_ProcessPayment">
  <bpmn:errorEventDefinition errorRef="Error_Payment"/>
  <bpmn:outgoing>Flow_ToErrorHandler</bpmn:outgoing>
</bpmn:boundaryEvent>
```

### Boundary Timer Event
For SLA/timeout requirements:
```xml
<bpmn:boundaryEvent id="Timer_Escalation" attachedToRef="Activity_Review" cancelActivity="false">
  <bpmn:timerEventDefinition>
    <bpmn:timeDuration>P2D</bpmn:timeDuration>
  </bpmn:timerEventDefinition>
  <bpmn:outgoing>Flow_ToEscalation</bpmn:outgoing>
</bpmn:boundaryEvent>
```

### Compensation Pattern
For rollback scenarios, use compensation events and handlers.

## Process Complexity Guidelines

### Simple (no lanes, ≤10 elements)
- Linear or single decision point
- One actor implied
- No exception handling needed

### Medium (optional lanes, 10-25 elements)
- Multiple decision points
- 2-3 actors/roles
- Basic exception handling

### Complex (lanes required, 25+ elements)
- Multiple parallel paths
- 4+ actors
- Multiple exception scenarios
- Consider subprocess decomposition

**Rule**: If a process exceeds 30 elements, decompose into subprocesses.

## Visual Layout Algorithm v2.0

**CRITICAL**: Use intelligent layout calculation based on real-world pattern analysis.

**Primary Reference**: `references/intelligent-layout-algorithm-v3.md` - Latest v3.0 algorithm with progressive spacing
**Supporting Docs**:
- `references/layout-analysis.md` - Real-world pattern analysis (online order processing)
- `references/onboarding-layout-analysis.md` - Real-world pattern analysis (employee onboarding)
- `references/visual-best-practices.md` - Collision prevention techniques

### Critical Rules (MUST FOLLOW)

1. **Use COMPACT horizontal spacing (50-80px, NOT 180px)**
   - After start event: 52px
   - Standard gap: 55px (tasks, gateways)
   - Task chains: 40px (simple sequential tasks)
   - New sequences: 80px (after subprocesses, major sections)

2. **Multiple paths MUST have different Y coordinates**
   - Success path: y = lane_center_primary
   - Rejection path: y = lane_center_primary + 60px
   - Parallel branches: y = lane_center_primary + 120px
   - Minimum vertical separation: 60px (minor), 120px (major)

3. **Lane heights: Fixed sizes based on content (200-250px)**
   - Single flow: 200px
   - Two flows: 250px
   - NOT formula-driven: user prefers compact fixed heights

4. **End events: Align on SAME HORIZONTAL LINE** ✨ CRITICAL
   - All end events share one X position (same vertical line)
   - All end events share one Y position (same horizontal line)
   - Visual clarity: User instantly sees all possible endings
   - Saves space and reduces cognitive load

   **Example:**
   ```
                        → End_Success
   Happy Path → XOR ──→ End_Rejected   ← All at same (X, Y) baseline
                        → End_Cancelled
   ```

   **User Requirement (2025-12-09):**
   > "End events should be on one horizontal line to visually see where each branch ends."

5. **Subprocesses: ALWAYS collapsed by default**
   ```xml
   <bpmn:subProcess isExpanded="false">  <!-- Collapsed = 100x80px -->
   ```

6. **Backward flows are ALLOWED for compactness**
   - Elements can be positioned left of their predecessors
   - Use waypoints to route cleanly
   - Prioritize overall compactness over strict left-to-right

7. **NO elements with identical (x, y) coordinates**
   - Run collision detection before finalizing
   - Auto-correct with minimum adjustments

### Spacing Constants v2.0 (MANDATORY)

```javascript
// HORIZONTAL SPACING (Updated from user analysis)
AFTER_START_EVENT: 52px       // Start → First task
STANDARD_GAP: 55px            // Default between elements
TASK_CHAIN_GAP: 40px          // Simple task → task
NEW_SEQUENCE_GAP: 80px        // Major sequence breaks
END_EVENT_MARGIN: 80px        // Task → End event

// VERTICAL SPACING
PATH_SEPARATION_MINOR: 60px   // Success vs rejection paths
PATH_SEPARATION_MAJOR: 120px  // Parallel branches

// LANE SIZING
LANE_HEIGHT_SIMPLE: 200px     // 1 flow
LANE_HEIGHT_MEDIUM: 250px     // 2 flows
LANE_PADDING: 50px            // Top and bottom

// POOL MARGINS
POOL_MARGIN_LEFT: 130px
POOL_MARGIN_RIGHT: 130px
POOL_MARGIN_TOP: 50px
```

### Layout Algorithm Steps

**Phase 1: Process Analysis**
1. Build flow graph from start to end events
2. Identify logical sequences (chains without branching)
3. Assign flow levels (main=0, exception=1, parallel=2+)

**Phase 2: Lane Dimensions**
1. Count distinct flow levels per lane
2. Assign heights: 200px (1 flow), 250px (2 flows), or 200+(n*50)
3. Position lanes adjacently (no gaps)

**Phase 3: Horizontal Layout (X)**
1. Build level map (depth from start)
2. Calculate X for each element:
   - After start: +52px
   - After task: +40px (chain) or +55px (standard)
   - After gateway: +55px
   - New sequence: +80px
3. Optimize end events to same X coordinate (vertical line)
4. Allow backward flows if space-saving

**Phase 4: Vertical Layout (Y)**
1. Calculate center Y for each flow level:
   - 200px lane: main=+90, alt=+150
   - 250px lane: main=+70, branch=+190
2. Position elements centered at flow level
3. **Align all end events to same Y coordinate (horizontal line)** ✨ NEW
   - Calculate average Y of all end events
   - Set all end events to this Y
   - Creates visual "finish line"
3. Position boundary events at task bottom-right

**Phase 5: Waypoints**
1. Calculate exit point (gateway edges, task right-center)
2. Same-lane: direct or L-shape (2 waypoints)
3. Cross-lane: horizontal → vertical (3 waypoints)
4. Minimize waypoint count

**Phase 6: Collision Detection**
1. Check all element pairs for overlap (20px clearance)
2. Resolve by adjusting Y (same lane) or X (cross-lane)
3. Re-validate after adjustments

**Phase 7: Pool Dimensions**
1. Width = rightmost_element_x + 130px (max 2000px)
2. Height = sum(lane_heights) + 50px

### Practical Formulas v2.0

```javascript
// Horizontal positioning
next_x = current_x + current_width + gap_size
gap_size = isTaskChain ? 40 : (isNewSequence ? 80 : 55)

// Vertical positioning (200px lane)
main_path_y = lane_start + 90 - (element_height / 2)
alt_path_y = lane_start + 150 - (element_height / 2)

// Vertical positioning (250px lane)
main_path_y = lane_start + 70 - (element_height / 2)
branch_path_y = lane_start + 190 - (element_height / 2)

// End event grouping
all_end_events_x = max(predecessor_tasks.x + width) + 80

// Boundary event position
boundary_x = task_x + task_width - 18
boundary_y = task_y + task_height - 18
```

### Expected Results

For typical process (20-30 elements):
- **Pool width**: 1800-2000px (vs old 2500px = **25% more compact**)
- **Pool height**: 600-700px (vs old 800px)
- **Horizontal gaps**: 40-80px (vs old 180px)
- **Lane heights**: 200-250px (fixed, not formula)
- **No overlaps**: Guaranteed via collision detection

### Validation Checks

Before generating XML, verify:
- [ ] Horizontal spacing: 40-80px (NOT 180px)
- [ ] Lane heights: 200-250px (fixed values)
- [ ] End events grouped at same X
- [ ] No elements overlap (min 20px clearance)
- [ ] All elements inside their lanes
- [ ] Flows use 2-3 waypoints (not 4+)
- [ ] Total width < 2000px
- [ ] **CRITICAL**: Correct XML namespaces:
  - Process elements: `<bpmn:...>`
  - Visual shapes: `<bpmndi:BPMNShape>`
  - Visual edges: `<bpmndi:BPMNEdge>`
  - Diagram: `<bpmndi:BPMNDiagram>`
  - Bounds/waypoints: `<dc:Bounds>` and `<di:waypoint>`
- [ ] **CRITICAL**: BPMNDiagram structure (prevents "multiple DI elements" error):
  - **Single-process diagram** (99% of cases): `<bpmndi:BPMNPlane bpmnElement="Process_XXX">` → **NO Pool shape**
  - **Collaboration diagram** (rare): `<bpmndi:BPMNPlane bpmnElement="Collaboration_XXX">` → Pool shapes for participants
  - **NEVER** add `<bpmndi:BPMNShape bpmnElement="Process_XXX">` when BPMNPlane already references Process_XXX
  - See `references/xml-templates.md` section "⚠️ CRITICAL: BPMNDiagram Structure Rules" for details

## BPMN Anti-Patterns & Common Mistakes (CRITICAL)

**MUST AVOID** these top errors identified in research analyzing thousands of BPMN models:

### 1. Wrong Usage of Connecting Objects ❌ MOST COMMON (48%)

**NEVER:**
- Sequence Flow crossing Pool boundaries
- Message Flow inside a Pool
- Activities not connected within Pool

**ALWAYS:**
```
INSIDE Pool → Sequence Flow (solid arrow)
BETWEEN Pools → Message Flow (dashed arrow)
```

### 2. Missing Start/End Events ❌ (25%)

**NEVER:**
```
[Task 1] → [Task 2] → [Task 3]  ❌ No beginning/end
```

**ALWAYS:**
```
(Start) → [Task 1] → [Task 2] → (End)  ✅
```

### 3. Redundant Event Naming ❌

**NEVER:**
```
(Start) "Process Start"  ❌ Redundant
(End) "Process End"      ❌ Redundant
```

**ALWAYS:**
```
(Start) "Order Received"    ✅ Business event
(End) "Order Completed"     ✅ Result state
```

### 4. Inconsistent Naming ❌ VERY COMMON (35%)

**NEVER:**
```
"Validation"              ❌ Noun
"Review application"      ❌ Lowercase
"approval of request"     ❌ Different style
```

**ALWAYS:**
```
"Validate Application"    ✅ Verb + Object
"Review Application"      ✅ Consistent
"Approve Request"         ✅ Same pattern
```

### 5. Overcomplicated Models ❌ (32%)

**NEVER:**
- 50+ elements on one diagram
- "Everything on one page" style
- Deep nesting without subprocesses

**ALWAYS:**
- Max 20-30 elements per diagram
- Use Subprocesses for complexity
- Use Call Activities for reusable logic

### 6. Gateway Mistakes ❌ (28%) - CRITICAL RULES

#### 6a. EXCLUSIVE GATEWAY (XOR) - Decision Point

**Rules:**
- **ALWAYS** name as question: "Approved?", "Amount > 1000?"
- **ALWAYS** define default flow (the "else" case)
- Conditions must be **mutually exclusive** (only one can be true)
- Conditions must be **complete** (cover all possible values)

**XML Requirements:**
```xml
<bpmn:exclusiveGateway id="Gateway_Decision" name="Amount > 1000?"
  default="Flow_Small">  <!-- ✅ Default flow defined -->
  <bpmn:outgoing>Flow_Large</bpmn:outgoing>
  <bpmn:outgoing>Flow_Small</bpmn:outgoing>
</bpmn:exclusiveGateway>

<bpmn:sequenceFlow id="Flow_Large" name="Yes" sourceRef="Gateway_Decision">
  <bpmn:conditionExpression xsi:type="bpmn:tFormalExpression">
    ${amount > 1000}
  </bpmn:conditionExpression>
</bpmn:sequenceFlow>

<!-- Default flow has NO condition -->
<bpmn:sequenceFlow id="Flow_Small" name="No" sourceRef="Gateway_Decision"
  targetRef="Task_Next" />
```

**CRITICAL MISTAKES:**

**❌ Missing Default Flow:**
```xml
<!-- WRONG: No default = DEADLOCK if no condition matches -->
<bpmn:exclusiveGateway id="Gateway_Risk">
  <bpmn:outgoing>Flow_High</bpmn:outgoing>
  <bpmn:outgoing>Flow_Low</bpmn:outgoing>
</bpmn:exclusiveGateway>
<!-- If risk='MEDIUM', neither condition true → process stuck! -->
```

**❌ Overlapping Conditions:**
```xml
<!-- WRONG: Both could be true -->
<bpmn:sequenceFlow id="Flow_Student">
  <bpmn:conditionExpression>${age < 25}</bpmn:conditionExpression>
</bpmn:sequenceFlow>
<bpmn:sequenceFlow id="Flow_Premium">
  <bpmn:conditionExpression>${loyaltyYears > 5}</bpmn:conditionExpression>
</bpmn:sequenceFlow>
<!-- What if age=22 AND loyaltyYears=6? → Non-deterministic! -->
```

**❌ Incomplete Conditions:**
```xml
<!-- WRONG: Doesn't cover age >= 65 -->
<bpmn:sequenceFlow id="Flow_Child">
  <bpmn:conditionExpression>${age < 18}</bpmn:conditionExpression>
</bpmn:sequenceFlow>
<bpmn:sequenceFlow id="Flow_Adult">
  <bpmn:conditionExpression>${age >= 18 && age < 65}</bpmn:conditionExpression>
</bpmn:sequenceFlow>
<!-- Missing: seniors! -->
```

#### 6b. PARALLEL GATEWAY (AND) - Fork/Synchronize

**Rules:**
- **ALWAYS** use in pairs: Fork → ... → Join
- **NEVER** put conditions on outgoing flows (all paths always execute)
- **ALWAYS** match number of paths (3 out → 3 in)
- **ALWAYS** synchronize tokens (avoid multi-merge)

**XML Requirements:**
```xml
<!-- Fork: Creates multiple tokens -->
<bpmn:parallelGateway id="Gateway_Fork" name="Start Background Checks">
  <bpmn:outgoing>Flow_ToCredit</bpmn:outgoing>
  <bpmn:outgoing>Flow_ToIncome</bpmn:outgoing>
  <bpmn:outgoing>Flow_ToEmployment</bpmn:outgoing>
</bpmn:parallelGateway>
<!-- NO conditions! All 3 paths always execute -->

<!-- Join: Waits for ALL tokens -->
<bpmn:parallelGateway id="Gateway_Join" name="Synchronize Checks">
  <bpmn:incoming>Flow_CreditDone</bpmn:incoming>
  <bpmn:incoming>Flow_IncomeDone</bpmn:incoming>
  <bpmn:incoming>Flow_EmploymentDone</bpmn:incoming>
  <bpmn:outgoing>Flow_AllComplete</bpmn:outgoing>
</bpmn:parallelGateway>
```

**CRITICAL MISTAKES:**

**❌ Unbalanced Gateway Types (DEADLOCK!):**
```xml
<!-- WRONG: Parallel split → Exclusive join -->
<bpmn:parallelGateway id="Gateway_Fork">  <!-- Creates 3 tokens -->
  <bpmn:outgoing>Flow_A</bpmn:outgoing>
  <bpmn:outgoing>Flow_B</bpmn:outgoing>
  <bpmn:outgoing>Flow_C</bpmn:outgoing>
</bpmn:parallelGateway>

<bpmn:exclusiveGateway id="Gateway_BadJoin">  <!-- ❌ Expects 1 token! -->
  <bpmn:incoming>Flow_A</bpmn:incoming>
  <bpmn:incoming>Flow_B</bpmn:incoming>
  <bpmn:incoming>Flow_C</bpmn:incoming>
</bpmn:exclusiveGateway>
<!-- Result: First token passes, 2 tokens stuck → DEADLOCK -->
```

**❌ Missing Synchronization (Multi-Merge):**
```xml
<!-- WRONG: No join gateway -->
<bpmn:parallelGateway id="Gateway_Fork">
  <bpmn:outgoing>Flow_TaskA</bpmn:outgoing>
  <bpmn:outgoing>Flow_TaskB</bpmn:outgoing>
</bpmn:parallelGateway>

<!-- Both tasks flow directly to next task -->
<bpmn:sequenceFlow sourceRef="Task_A" targetRef="Task_Final" />
<bpmn:sequenceFlow sourceRef="Task_B" targetRef="Task_Final" />
<!-- ❌ Task_Final receives 2 tokens → executes TWICE! -->

<!-- FIX: Add parallel join before Task_Final -->
```

**❌ Unbalanced Path Count:**
```xml
<!-- WRONG: 3 paths out, 2 paths in -->
<bpmn:parallelGateway id="Gateway_Fork">
  <bpmn:outgoing>Flow_A</bpmn:outgoing>
  <bpmn:outgoing>Flow_B</bpmn:outgoing>
  <bpmn:outgoing>Flow_C</bpmn:outgoing>
</bpmn:parallelGateway>

<bpmn:parallelGateway id="Gateway_Join">
  <bpmn:incoming>Flow_A</bpmn:incoming>
  <bpmn:incoming>Flow_B</bpmn:incoming>
  <!-- Missing Flow_C! → Join waits forever for 3rd token → DEADLOCK -->
</bpmn:parallelGateway>
```

#### 6c. INCLUSIVE GATEWAY (OR) - Conditional Parallelism

**Rules:**
- **ALWAYS** ensure at least one path activates (use default flow)
- **NEVER** terminate tokens between OR-split and OR-join (no End events!)
- **AVOID** in complex models (prefer Parallel + Exclusive instead)
- Match split and join paths exactly

**XML Requirements:**
```xml
<bpmn:inclusiveGateway id="Gateway_ReviewTypes"
  name="Which Reviews Needed?">
  <bpmn:outgoing>Flow_Legal</bpmn:outgoing>
  <bpmn:outgoing>Flow_Financial</bpmn:outgoing>
  <bpmn:outgoing>Flow_Technical</bpmn:outgoing>
</bpmn:inclusiveGateway>

<bpmn:sequenceFlow id="Flow_Legal" name="Contract involved">
  <bpmn:conditionExpression>${hasContract}</bpmn:conditionExpression>
</bpmn:sequenceFlow>
<bpmn:sequenceFlow id="Flow_Financial" name="Amount > $10k">
  <bpmn:conditionExpression>${amount > 10000}</bpmn:conditionExpression>
</bpmn:sequenceFlow>
<!-- Default: ensures at least one path taken -->
<bpmn:sequenceFlow id="Flow_Technical" name="New technology" default="true" />
```

**CRITICAL MISTAKES:**

**❌ Token Termination Between Split/Join (DEADLOCK!):**
```xml
<!-- WRONG: End event in OR path -->
<bpmn:inclusiveGateway id="Gateway_Split">
  <bpmn:outgoing>Flow_ProcessA</bpmn:outgoing>
  <bpmn:outgoing>Flow_ProcessB</bpmn:outgoing>
</bpmn:inclusiveGateway>

<!-- Path A terminates early -->
<bpmn:sequenceFlow sourceRef="Task_A" targetRef="End_PathA" />  <!-- ❌ -->

<!-- Path B goes to join -->
<bpmn:inclusiveGateway id="Gateway_Join">
  <bpmn:incoming>Flow_FromA</bpmn:incoming>
  <bpmn:incoming>Flow_FromB</bpmn:incoming>
  <!-- Waits forever for Path A token that was terminated! → DEADLOCK -->
</bpmn:inclusiveGateway>
```

**❌ No Default Path (All Conditions False):**
```xml
<!-- WRONG: All conditions could be false -->
<bpmn:inclusiveGateway id="Gateway_Optional">
  <bpmn:outgoing>Flow_A</bpmn:outgoing>
  <bpmn:outgoing>Flow_B</bpmn:outgoing>
</bpmn:inclusiveGateway>
<bpmn:sequenceFlow id="Flow_A">
  <bpmn:conditionExpression>${typeA}</bpmn:conditionExpression>
</bpmn:sequenceFlow>
<bpmn:sequenceFlow id="Flow_B">
  <bpmn:conditionExpression>${typeB}</bpmn:conditionExpression>
</bpmn:sequenceFlow>
<!-- If typeA=false AND typeB=false → NO paths activate → DEADLOCK -->
<!-- FIX: Mark one flow as default="true" -->
```

#### 6d. Gateway Type Balancing Rules

**Safe Patterns ✅:**
```
Exclusive Split → Exclusive Merge (or no merge) ✅
Parallel Split → Parallel Join ✅
Inclusive Split → Inclusive Join ✅
```

**DANGEROUS Patterns ⚠️:**
```
Parallel Split → Exclusive Join ❌ DEADLOCK (join gets N tokens, expects 1)
Exclusive Split → Parallel Join ❌ DEADLOCK (join expects N tokens, gets 1)
Inclusive Split → Parallel Join ❌ DEADLOCK (join may get wrong token count)
```

**When Asymmetry is OK:**
```
Parallel Split (3 paths) → 1 path ends early → Parallel Join (2 paths) ✅
  IF documented and intentional (e.g., send notification independently)
```

### 7. Gateway Sending Messages ❌

**NEVER:**
```
Gateway → Message Event  ❌
```

**ALWAYS:**
```
Task → Message Event  ✅
```

### 8. Inclusive Gateway Token Issues ❌

**NEVER:**
```
Inclusive_Fork → Path A → End           ❌
               → Path B → Inclusive_Join  ❌ Waiting for A!
```

**ALWAYS:**
```
Inclusive_Fork → Path A → Inclusive_Join → End  ✅
               → Path B ↗
```

### 9. Multi-Merge Anti-Pattern ❌ CRITICAL

**Problem:** Multiple flows converging directly into task/event without explicit merge gateway

**Context:** Research shows this is a common anti-pattern that creates unclear semantics and maintenance issues.

#### 9a. DANGEROUS Multi-Merge (after Parallel Gateway) ❌

**NEVER:**
```xml
<!-- WRONG: Parallel fork without synchronization -->
<bpmn:parallelGateway id="Gateway_Fork">
  <bpmn:outgoing>Flow_TaskA</bpmn:outgoing>
  <bpmn:outgoing>Flow_TaskB</bpmn:outgoing>
</bpmn:parallelGateway>

<!-- Both tasks flow directly to final task -->
<bpmn:sequenceFlow sourceRef="Task_A" targetRef="Task_Final" />
<bpmn:sequenceFlow sourceRef="Task_B" targetRef="Task_Final" />
<!-- ❌ Task_Final receives 2 tokens → executes TWICE! -->
```

**Why this is CRITICAL:**
- Parallel gateway creates **multiple tokens** (one per path)
- Task receives **all tokens** without synchronization
- **Task executes multiple times** (once per token)
- Creates **race conditions** and **duplicate data**

**ALWAYS:**
```xml
<!-- CORRECT: Explicit synchronization -->
<bpmn:parallelGateway id="Gateway_Fork">
  <bpmn:outgoing>Flow_ToA</bpmn:outgoing>
  <bpmn:outgoing>Flow_ToB</bpmn:outgoing>
</bpmn:parallelGateway>

<!-- Tasks complete independently -->
<bpmn:sequenceFlow sourceRef="Task_A" targetRef="Gateway_Join" />
<bpmn:sequenceFlow sourceRef="Task_B" targetRef="Gateway_Join" />

<!-- Parallel join synchronizes tokens -->
<bpmn:parallelGateway id="Gateway_Join">
  <bpmn:incoming>Flow_FromA</bpmn:incoming>
  <bpmn:incoming>Flow_FromB</bpmn:incoming>
  <bpmn:outgoing>Flow_ToFinal</bpmn:outgoing>
</bpmn:parallelGateway>

<!-- Task receives single synchronized token -->
<bpmn:userTask id="Task_Final">
  <bpmn:incoming>Flow_ToFinal</bpmn:incoming>  <!-- ✅ One token -->
</bpmn:userTask>
```

#### 9b. Technically Valid BUT NOT RECOMMENDED Multi-Merge ⚠️

**Context:** When flows are mutually exclusive (from different XOR gateways)

**Technically allowed but discouraged:**
```xml
<!-- Flows from exclusive gateways (only one can be active) -->
<bpmn:serviceTask id="Task_NotifyRejection">
  <bpmn:incoming>Flow_InvalidClaim</bpmn:incoming>      <!-- XOR path 1 -->
  <bpmn:incoming>Flow_ChecksFailed</bpmn:incoming>      <!-- XOR path 2 -->
  <bpmn:incoming>Flow_ManagerReject</bpmn:incoming>     <!-- XOR path 3 -->
  <bpmn:incoming>Flow_PaymentFailed</bpmn:incoming>     <!-- XOR path 4 -->
</bpmn:serviceTask>
<!-- ⚠️ Technically valid (only one token arrives) but unclear -->
```

**Why this is problematic:**
- ❌ **Not visually clear** that flows are mutually exclusive
- ❌ **Hard to maintain** - easy to break when adding paths
- ❌ **Violates guideline** - "activities should have one incoming flow"
- ❌ **Risk of future errors** - if someone adds parallel path by mistake

**BEST PRACTICE - Use Explicit Merge Gateway:**
```xml
<!-- CORRECT: Explicit merge makes semantics clear -->
<bpmn:exclusiveGateway id="Gateway_MergeRejections">
  <bpmn:incoming>Flow_InvalidClaim</bpmn:incoming>
  <bpmn:incoming>Flow_ChecksFailed</bpmn:incoming>
  <bpmn:incoming>Flow_ManagerReject</bpmn:incoming>
  <bpmn:incoming>Flow_PaymentFailed</bpmn:incoming>
  <bpmn:outgoing>Flow_ToNotify</bpmn:outgoing>
</bpmn:exclusiveGateway>

<bpmn:serviceTask id="Task_NotifyRejection">
  <bpmn:incoming>Flow_ToNotify</bpmn:incoming>  <!-- ✅ Single incoming -->
  <bpmn:outgoing>Flow_Notify_End</bpmn:outgoing>
</bpmn:serviceTask>
```

**Benefits of explicit merge gateway:**
- ✅ **Visually clear** - obvious merge point
- ✅ **Easy to maintain** - add new paths to gateway
- ✅ **Prevents errors** - impossible to accidentally create multi-token execution
- ✅ **Follows best practices** - one incoming flow per activity
- ✅ **Better documentation** - gateway can be named to explain merge

#### 9c. The Golden Rule

**RULE:** Only **gateways** should have multiple incoming sequence flows

**Exception:** End events can have multiple incoming flows (they consume tokens)

```
✅ Gateway with multiple incoming → OK (this is their purpose)
❌ Task with multiple incoming → NOT OK (use merge gateway first)
❌ Start Event with multiple incoming → ERROR (invalid BPMN)
✅ End Event with multiple incoming → OK (each token terminates independently)
```

**Visual pattern to always follow:**
```
Multiple Paths → Merge Gateway → Single Flow → Task/Event
```

#### 9d. MANDATORY Algorithm to Prevent Multi-Merge ⚙️

**CRITICAL:** Execute this algorithm for EVERY sequence flow you create.

**RECOMMENDED APPROACH: Complete Phase 0 tabular analysis BEFORE creating XML**

Before implementing flows one-by-one, create a complete inventory table (see Phase 0 section):

**Example Pre-Creation Table for 10-element process:**

| # | Element ID | Type | Step # | Incoming Count | Sources | Action |
|---|------------|------|--------|----------------|---------|--------|
| 1 | Event_Start | Start | - | 0 | - | ✅ OK |
| 2 | Task_CheckReq | Task | 2.1.1 | 1 | Event_Start | ✅ OK |
| 3 | Gateway_NewReq | Gateway | 2.1.2 | 1 | Task_CheckReq | ✅ OK |
| 4 | Gateway_LibAvail | Gateway | 2.2.1 | 1 | Gateway_NewReq | ✅ OK |
| 5 | Task_UseLib | Task | 2.2.2 | 1 | Gateway_LibAvail(Yes) | ✅ OK |
| 6 | Gateway_Correction | Gateway | 2.2.3 | 1 | Task_UseLib | ✅ OK |
| 7 | Task_CreatePackage | Task | 2.2.4 | **2** | Gateway_LibAvail(No), Gateway_Correction(Yes) | ⚠️ **ADD MERGE** |
| 8 | Gateway_Materials | Gateway | 2.2.5 | **2** | Gateway_Correction(No), Task_CreatePackage | ⚠️ **ADD MERGE** |
| 9 | Task_AddMaterials | Task | 2.2.6 | 1 | Gateway_Materials(Yes) | ✅ OK |
| 10 | Event_Complete | End | - | 1 | Task_AddMaterials | ✅ OK (End can have multiple) |

**Key insight:** Rows 7 and 8 show Multi-Merge BEFORE creating any XML. Add merge gateways to fix:
- Insert `Gateway_Merge_BeforePackage` before Task_CreatePackage
- Insert `Gateway_Merge_BeforeMaterials` before Gateway_Materials

**CRITICAL:** Do NOT skip any element in this table! Checking "by memory" leads to missed cases.

---

**Alternative: Per-Flow Algorithm (if not using Phase 0 table):**

If you're creating flows incrementally, apply this algorithm to EACH flow:

**Step-by-Step Process:**

```
FOR EACH sequence flow you want to create:

1. IDENTIFY target element (where flow points to)

2. CHECK target element type:
   ├─ Is it an End Event?
   │  └─ YES → ✅ ALLOW multiple incoming (skip to step 6)
   │
   └─ Is it a Task, Gateway, or Start/Intermediate Event?
      └─ YES → CONTINUE to step 3

3. COUNT existing incoming flows to target:
   ├─ incoming.length === 0
   │  └─ ✅ SAFE: Add flow directly (skip to step 6)
   │
   └─ incoming.length >= 1
      └─ ⚠️ MULTI-MERGE DETECTED → GO TO step 4

4. INSERT MERGE GATEWAY (MANDATORY):

   a) Determine merge type:
      ├─ All paths from Parallel Gateway splits?
      │  └─ Use Parallel Gateway (synchronization)
      │
      ├─ Paths are mutually exclusive?
      │  └─ Use Exclusive Gateway (XOR merge)
      │
      └─ Paths can overlap (one or more active)?
         └─ Use Inclusive Gateway (OR merge)

   b) Create merge gateway:
      - ID: Gateway_Merge[Purpose]  (e.g., Gateway_MergeRejections)
      - Name: Descriptive merge name (e.g., "Merge Rejection Paths")
      - Place BEFORE target element

   c) Redirect ALL incoming flows:
      - Change targetRef of existing flow(s) → merge gateway
      - Change targetRef of new flow → merge gateway

   d) Create single outflow:
      - sourceRef: merge gateway
      - targetRef: original target element

5. UPDATE visual layout:
   - Position gateway 110px before target (X - 110)
   - Update all flow waypoints
   - Ensure clean visual routing

6. VALIDATE result:
   ✅ Target element has EXACTLY ONE incoming flow
   ✅ Merge gateway (if added) has multiple incoming
   ✅ Semantics are visually clear

DONE ✅
```

**Implementation Example:**

```javascript
// PSEUDO-CODE for safe flow creation

function createSequenceFlow(sourceId, targetId, flowId) {
  const target = getElement(targetId);

  // Step 2: Check if target allows multiple incoming
  if (target.type === 'endEvent') {
    // ✅ End events can have multiple incoming
    return addFlowDirectly(sourceId, targetId, flowId);
  }

  // Step 3: Count existing incoming flows
  const existingIncoming = target.incoming || [];

  if (existingIncoming.length === 0) {
    // ✅ No existing flows - safe to add directly
    return addFlowDirectly(sourceId, targetId, flowId);
  }

  // Step 4: MULTI-MERGE DETECTED - insert merge gateway
  console.warn(`⚠️ Multi-Merge detected at ${targetId}`);

  // Step 4a: Determine merge type
  const mergeType = determineMergeType(sourceId, existingIncoming);

  // Step 4b: Create merge gateway
  const mergeGateway = createGateway({
    id: `Gateway_Merge_${targetId}`,
    type: mergeType,
    name: `Merge paths to ${target.name}`
  });

  // Step 4c: Redirect all flows to gateway
  existingIncoming.forEach(flow => {
    flow.targetRef = mergeGateway.id;
  });

  // Add new flow to gateway
  addFlowDirectly(sourceId, mergeGateway.id, flowId);

  // Step 4d: Create single flow from gateway to target
  const mergeToTargetFlow = {
    id: `Flow_${mergeGateway.id}_${targetId}`,
    sourceRef: mergeGateway.id,
    targetRef: targetId
  };
  addFlowDirectly(mergeGateway.id, targetId, mergeToTargetFlow.id);

  // Step 5: Update visual layout
  positionGateway(mergeGateway, target);

  // Step 6: Validate
  assert(target.incoming.length === 1, "Target must have exactly 1 incoming");

  return mergeToTargetFlow;
}
```

**Quick Decision Tree:**

```
Creating flow to element X?
│
├─ X is End Event? → ✅ Add flow directly
│
├─ X has 0 incoming flows? → ✅ Add flow directly
│
└─ X has 1+ incoming flows? → ⚠️ INSERT MERGE GATEWAY FIRST
   │
   ├─ After Parallel split? → Use Parallel Gateway
   ├─ Mutually exclusive? → Use Exclusive Gateway
   └─ Can overlap? → Use Inclusive Gateway
```

**Real-World Application:**

```xml
<!-- SCENARIO: Adding 2nd path to Task_Notify -->

<!-- BEFORE: Task has 1 incoming -->
<bpmn:userTask id="Task_Notify">
  <bpmn:incoming>Flow_Success</bpmn:incoming>  <!-- 1st flow exists -->
</bpmn:userTask>

<!-- ALGORITHM DETECTS: incoming.length = 1 → INSERT GATEWAY -->

<!-- AFTER: Algorithm auto-inserts merge gateway -->
<bpmn:exclusiveGateway id="Gateway_MergeNotify">
  <bpmn:incoming>Flow_Success</bpmn:incoming>      <!-- Redirected -->
  <bpmn:incoming>Flow_Alternative</bpmn:incoming>  <!-- New flow -->
  <bpmn:outgoing>Flow_Merge_Notify</bpmn:outgoing>
</bpmn:exclusiveGateway>

<bpmn:userTask id="Task_Notify">
  <bpmn:incoming>Flow_Merge_Notify</bpmn:incoming>  <!-- ✅ Still 1 incoming -->
</bpmn:userTask>
```

**Validation Checklist (Run After EVERY Flow Creation):**

```
For each Task/Event (except End Events):
□ Has exactly ONE incoming sequence flow
□ If multiple paths converge → merge gateway present
□ Gateway type matches path semantics (XOR/AND/OR)
□ Visual layout shows clear merge point
```

**Common Mistakes to Avoid:**

```
❌ "I'll add the merge gateway later" → NO! Add it NOW
❌ "Only 2 flows, doesn't need gateway" → WRONG! Still needs gateway
❌ "Flows are exclusive, it's fine" → NOT OK! Use explicit merge
❌ "End event can merge" → ONLY for end events, not tasks
```

**Remember:**
> **"If you're about to create a 2nd incoming flow to a task/event (except end event), STOP and insert a merge gateway first."**

### 10. No Error Handling ❌

**NEVER:**
```
[Payment Task] → [Success]  ❌ What if error?
```

**ALWAYS:**
```
[Payment Task] → [Success]
     ⚡ Boundary Error → [Handle] → End  ✅
```

### 10. Multiple Ends Without Semantic Difference ❌

**NEVER:**
```
Path A → (End) "Finished"  ❌
Path B → (End) "Done"      ❌ Both mean success
```

**ALWAYS:**
```
Path A → Merge → (End) "Success"  ✅
Path B ↗
```

### 11. Decision After Decision ❌

**NEVER:**
```
Gateway_1 → Gateway_2  ❌ No task between
```

**ALWAYS:**
```
Gateway_1 → [Task] → Gateway_2  ✅
```

### 12. Implicit Gateways ❌ CRITICAL ANTI-PATTERN

**Problem:** Decision logic hidden in sequence flows instead of explicit gateway

**NEVER do this:**
```xml
<!-- WRONG: Task with conditional outgoing flows -->
<bpmn:userTask id="Task_RequestDocuments">
  <bpmn:outgoing>Flow_Continue</bpmn:outgoing>
  <bpmn:outgoing>Flow_Cancel</bpmn:outgoing>  <!-- ❌ Two outgoing! -->
</bpmn:userTask>

<bpmn:sequenceFlow id="Flow_Continue"
  sourceRef="Task_RequestDocuments"
  targetRef="Task_PrepareDocuments" />

<bpmn:sequenceFlow id="Flow_Cancel"
  sourceRef="Task_RequestDocuments"
  targetRef="End_Cancelled">
  <bpmn:conditionExpression xsi:type="bpmn:tFormalExpression">
    ${cancelled}  <!-- ❌ Condition on flow from task! -->
  </bpmn:conditionExpression>
</bpmn:sequenceFlow>
```

**Why this is wrong:**
- Tasks should have **ONLY ONE** outgoing sequence flow
- Decision logic must be **VISIBLE** in a gateway
- Creates "invisible" decision point (hard to understand process)
- Violates BPMN best practices

**Impact:**
- Unclear process logic
- Hard to maintain
- Difficult to trace decision points

**ALWAYS do this instead:**
```xml
<!-- CORRECT: Explicit gateway after task -->
<bpmn:userTask id="Task_RequestDocuments">
  <bpmn:outgoing>Flow_ToGateway</bpmn:outgoing>  <!-- ✅ Single outgoing -->
</bpmn:userTask>

<bpmn:exclusiveGateway id="Gateway_CancelDecision"
  name="Cancel Process?"
  default="Flow_Continue">
  <bpmn:incoming>Flow_ToGateway</bpmn:incoming>
  <bpmn:outgoing>Flow_Continue</bpmn:outgoing>
  <bpmn:outgoing>Flow_Cancel</bpmn:outgoing>
</bpmn:exclusiveGateway>

<bpmn:sequenceFlow id="Flow_Continue" name="No"
  sourceRef="Gateway_CancelDecision"
  targetRef="Task_PrepareDocuments" />

<bpmn:sequenceFlow id="Flow_Cancel" name="Yes"
  sourceRef="Gateway_CancelDecision"
  targetRef="End_Cancelled">
  <bpmn:conditionExpression>${cancelled}</bpmn:conditionExpression>
</bpmn:sequenceFlow>
```

**Benefits:**
- ✅ Decision point clearly visible
- ✅ Gateway shows question: "Cancel Process?"
- ✅ Easy to understand and maintain
- ✅ Follows BPMN best practices

**Detection Rule:**
- If a task/activity has more than one outgoing flow → **IMPLICIT GATEWAY ERROR**
- Exception: Boundary events (error/timeout) are OK

## Validation Checklist (MANDATORY)

Before finalizing ANY model, verify:

### Structural Soundness
- [ ] **Start Event present** (not implicit)
- [ ] **End Event(s) present** for each path
- [ ] **All paths lead to End** (no dead ends)
- [ ] **Gateways balanced** (fork = join, same type)
- [ ] **No orphan elements** (all connected)

### Naming Conventions
- [ ] **Tasks: Verb + Object** ("Validate Application")
- [ ] **Gateways: Question** ("Approved?", "Amount > 1000?")
- [ ] **Events: State/Trigger** ("Order Received", "Payment Failed")
- [ ] **Flows: Condition** ("Yes", "No", "High Risk")
- [ ] **Consistent style** throughout

### Connection Rules
- [ ] **Sequence Flow ONLY within Pool**
- [ ] **Message Flow ONLY between Pools**
- [ ] **No flows crossing Pool boundaries** (except Message Flow)
- [ ] **Gateways don't send/receive messages**

### Gateway Logic (CRITICAL)
- [ ] **EXCLUSIVE (XOR) Gateways:**
  - [ ] Named as question ("Approved?", "Amount > 1000?")
  - [ ] Default flow defined (`default="Flow_ID"` attribute)
  - [ ] Conditions mutually exclusive (only one can be true)
  - [ ] Conditions complete (all values covered)
  - [ ] No conditional flows from tasks (use gateway instead)
- [ ] **PARALLEL (AND) Gateways:**
  - [ ] Fork has matching Join (paired)
  - [ ] Same number of paths (3 out → 3 in, or documented asymmetry)
  - [ ] No conditions on outgoing flows (all paths always execute)
  - [ ] No multi-merge (missing join before converging task)
- [ ] **INCLUSIVE (OR) Gateways:**
  - [ ] At least one path guaranteed (default flow or always-true condition)
  - [ ] No End events between OR-split and OR-join
  - [ ] Split and Join paths match
- [ ] **Gateway Type Matching:**
  - [ ] No Parallel Split → Exclusive Join (DEADLOCK!)
  - [ ] No Exclusive Split → Parallel Join (DEADLOCK!)
  - [ ] No mixed types unless documented and intentional
- [ ] **No Implicit Gateways:**
  - [ ] Tasks have only ONE outgoing sequence flow
  - [ ] All decisions use explicit gateways (not conditional flows from tasks)
- [ ] **No Multi-Merge Anti-Pattern:**
  - [ ] **PHASE 0 COMPLETED:** Created tabular analysis of ALL elements with incoming flow counts
  - [ ] **SYSTEMATIC CHECK:** Verified EVERY element (not just gateways) for incoming.length
  - [ ] Tasks have only ONE incoming sequence flow (use merge gateway)
  - [ ] Gateways (except merge gateways) have only ONE incoming sequence flow
  - [ ] Events (except End) have only ONE incoming sequence flow
  - [ ] ALL path convergences use explicit merge gateway
  - [ ] Exception: End events can have multiple incoming (token termination)
  - [ ] Rule: "Multiple paths → Merge Gateway → Single flow → Element"
  - [ ] **Verification method:** Table shows no element (except End Events and Merge Gateways) with incoming > 1

### Complexity
- [ ] **≤ 30 elements** per diagram
- [ ] **Subprocesses** for complex sections
- [ ] **Call Activities** for reusable logic
- [ ] **No deep nesting** (max 2 levels)

### Error Handling
- [ ] **Boundary events** on critical tasks
- [ ] **Error paths** modeled
- [ ] **Timeouts** where needed
- [ ] **Compensation** for transactions

### Semantic Clarity
- [ ] **Different End Events** have different meanings
- [ ] **Each path** has clear outcome
- [ ] **Business-readable** labels
- [ ] **No technical jargon** (unless tech doc)

### Visual Layout
- [ ] **Left to right** flow
- [ ] **No overlapping** elements
- [ ] **Minimal crossing** flows
- [ ] **Clean waypoints** (2-3 per flow)

### Process Layout (PMA Approach)
- [ ] **NO LANES** for single-organization processes (use role overlays)
- [ ] **Straight happy path** (not zigzagging across lanes)
- [ ] **Roles indicated** on tasks (Camunda attributes or name prefix)
  - [ ] If numbered steps provided: Use format `N.N.N [Role] Task Name` (see Element Numbering)
- [ ] **Lanes ONLY for** cross-organizational collaboration (different companies)
- [ ] **Compact diagram** - minimize vertical size

## Output Checklist

Before finalizing, verify:

- [ ] Process has clear start trigger (named appropriately)
- [ ] All end events have semantic names (Approved, Rejected, Cancelled, Error)
- [ ] Every gateway has a question-format name
- [ ] All tasks follow Verb + Object naming
- [ ] Parallel gateways are balanced (fork has matching join)
- [ ] No orphan elements (everything connected)
- [ ] Lanes reflect actual role separation (or omitted if single-actor)
- [ ] Exception paths are modeled where business-critical
- [ ] Flow labels on conditional paths are human-readable
- [ ] Visual layout has no overlapping elements

## Camunda Modeler Integration

### Opening Files in Camunda Modeler

**ALWAYS** after creating or editing a .bpmn file, offer to open it in Camunda Modeler:

```bash
open -a "Camunda Modeler" "[filename].bpmn"
```

User can then:
- Visually inspect the generated diagram
- Make manual adjustments if needed
- Add visual styling (colors, fonts)
- Test the process with execution
- Export as PNG/SVG for documentation

### Editing Existing BPMN Files

When user requests to edit/modify an existing .bpmn file:

1. **Read the existing file** first to understand current structure
2. **Parse the XML** to identify:
   - Existing elements (tasks, gateways, events)
   - Current IDs and naming
   - Visual layout coordinates
   - Existing flows and connections
3. **Make targeted changes** preserving:
   - Element IDs (unless renaming required)
   - Visual layout (adjust minimally)
   - Existing flows (unless modification requested)
4. **Preserve DI (Diagram Interchange)** elements to maintain visual layout
5. **Update incrementally** to avoid breaking the diagram

### Common Edit Operations

#### Adding a New Task
- Insert XML element in `<bpmn:process>`
- Add corresponding `<bpmndi:BPMNShape>` in diagram section
- Update sequence flows to connect new task
- Calculate coordinates based on nearby elements

#### Modifying a Gateway
- Update gateway type if needed (exclusive → parallel)
- Adjust outgoing flows and conditions
- Update visual shape if gateway type changes

#### Adding Exception Handling
- Add boundary event attached to existing task
- Create error/timer event definition
- Add handler task and flows
- Position handler below main flow

#### Renaming Elements
- Update `name` attribute in element
- Preserve `id` attribute (or update all references if changing)
- No need to update visual layout for name changes

### Workflow with Camunda Modeler

**Recommended Process:**
1. Claude generates/edits .bpmn file
2. File automatically opens in Camunda Modeler
3. User reviews visually in Camunda
4. User requests further changes if needed
5. Claude applies changes, reopens file
6. Repeat until satisfied

## File Output

Generate complete BPMN 2.0 XML. Save as `[process-name].bpmn`.

**After saving, ALWAYS offer to open in Camunda Modeler** using:
```bash
open -a "Camunda Modeler" "[process-name].bpmn"
```

## Reference Documentation Index

### Core Templates & Patterns
- `references/xml-templates.md` - XML structure templates for all BPMN elements
- `references/advanced-patterns.md` - Advanced BPMN patterns (compensation, escalation, multi-instance)
- `references/editing-guide.md` - Guide for editing existing .bpmn files

### Gateway Rules (Critical)
- `references/gateway-rules-and-antipatterns.md` - Comprehensive gateway guide (English)
- `references/gateway-cheatsheet-ru.md` - Gateway quick reference cheatsheet (Russian)

### Layout & Positioning
- `references/intelligent-layout-algorithm-v3.md` - Primary layout algorithm (v3.0)
- `references/layout-analysis.md` - Real-world pattern analysis (online order processing)
- `references/onboarding-layout-analysis.md` - Real-world pattern analysis (employee onboarding)
- `references/visual-best-practices.md` - Collision prevention and visual clarity

### Validation & Quality
- `references/process-validation-report.md` - Example validation reports for sample processes

### Usage
Refer to these documents when:
- **Creating processes**: Use xml-templates.md for syntax, layout algorithm for positioning
- **Editing processes**: Use editing-guide.md for modification patterns
- **Gateway logic**: Use gateway-rules-and-antipatterns.md for all gateway types
- **Troubleshooting**: Check process-validation-report.md for examples of common issues
- **Advanced features**: Use advanced-patterns.md for complex scenarios
