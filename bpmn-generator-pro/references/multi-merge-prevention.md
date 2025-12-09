# Multi-Merge Prevention Algorithm (Phase 0)

**Reference for:** `SKILL.md` Phase 0 section

**Purpose:** Systematic algorithm to prevent Multi-Merge anti-pattern BEFORE creating XML elements.

## Why This Phase Exists

**Problem:** Multi-Merge anti-pattern (multiple incoming flows to a single element) is the most commonly overlooked error in BPMN modeling.

**Root Cause:** Checking "by memory" or "visually" leads to missed cases, especially in complex processes with many elements.

**Solution:** Systematic tabular analysis that validates EVERY element before XML creation.

## What is Multi-Merge?

**Definition:** When a task, gateway, or event (except End Events) receives multiple incoming sequence flows directly, without an explicit merge gateway.

**Why it's dangerous:**

### After Parallel Gateway (CRITICAL - Execution Error)
```xml
<!-- WRONG: Parallel fork without synchronization -->
<bpmn:parallelGateway id="Gateway_Fork">
  <bpmn:outgoing>Flow_TaskA</bpmn:outgoing>
  <bpmn:outgoing>Flow_TaskB</bpmn:outgoing>
</bpmn:parallelGateway>

<bpmn:sequenceFlow sourceRef="Task_A" targetRef="Task_Final" />
<bpmn:sequenceFlow sourceRef="Task_B" targetRef="Task_Final" />
<!-- ❌ Task_Final receives 2 tokens → executes TWICE! -->
```

**Impact:**
- Task executes **multiple times** (once per token)
- Creates **race conditions**
- **Duplicate data** in system
- **Unpredictable behavior**

### After Exclusive Gateways (Maintenance Issue)
```xml
<!-- Flows from different exclusive gateways -->
<bpmn:userTask id="Task_Notify">
  <bpmn:incoming>Flow_InvalidClaim</bpmn:incoming>    <!-- XOR path 1 -->
  <bpmn:incoming>Flow_ChecksFailed</bpmn:incoming>    <!-- XOR path 2 -->
  <bpmn:incoming>Flow_ManagerReject</bpmn:incoming>   <!-- XOR path 3 -->
</bpmn:userTask>
<!-- ⚠️ Technically valid (only one token) but unclear and hard to maintain -->
```

**Impact:**
- ❌ Not visually clear that flows are mutually exclusive
- ❌ Hard to maintain - easy to break when adding paths
- ❌ Risk of future errors if someone adds parallel path

## The Golden Rule

**RULE:** Only **merge gateways** and **end events** should have multiple incoming sequence flows.

```
✅ Merge Gateway with multiple incoming → OK (this is their purpose)
✅ End Event with multiple incoming → OK (consumes tokens)
❌ Task with multiple incoming → NOT OK (add merge gateway first)
❌ Gateway (non-merge) with multiple incoming → NOT OK (add merge gateway first)
❌ Start Event with multiple incoming → ERROR (invalid BPMN)
```

## Step-by-Step Tabular Analysis (RECOMMENDED)

**Complete this analysis BEFORE creating any XML elements.**

### Step 1: List ALL Elements

Create comprehensive inventory of your process:
- Start event(s)
- All tasks (with step numbers if provided: 2.1.1, 2.2.2, etc.)
- All gateways (decision points, splits, joins)
- All end events
- Any intermediate events

### Step 2: Create Analysis Table

For EACH element, count incoming sequence flows:

| # | Element ID | Element Type | Step # | Incoming Count | Sources | Action Required |
|---|------------|--------------|--------|----------------|---------|-----------------|
| 1 | Event_Start | Start Event | - | 0 | - | ✅ OK (Start has no incoming) |
| 2 | Task_CheckReq | Task | 2.1.1 | 1 | Event_Start | ✅ OK |
| 3 | Gateway_NewReq | Gateway | 2.1.2 | 1 | Task_CheckReq | ✅ OK |
| 4 | Gateway_LibAvail | Gateway | 2.2.1 | 1 | Gateway_NewReq | ✅ OK |
| 5 | Task_UseLib | Task | 2.2.2 | 1 | Gateway_LibAvail(Yes) | ✅ OK |
| 6 | Gateway_Correction | Gateway | 2.2.3 | 1 | Task_UseLib | ✅ OK |
| 7 | Task_CreatePackage | Task | 2.2.4 | **2** | Gateway_LibAvail(No), Gateway_Correction(Yes) | ⚠️ **ADD MERGE GATEWAY** |
| 8 | Gateway_Materials | Gateway | 2.2.5 | **2** | Gateway_Correction(No), Task_CreatePackage | ⚠️ **ADD MERGE GATEWAY** |
| 9 | Task_AddMaterials | Task | 2.2.6 | 1 | Gateway_Materials(Yes) | ✅ OK |
| 10 | Event_Complete | End Event | - | 1 | Task_AddMaterials | ✅ OK (End can have multiple) |

### Step 3: For ANY Element with incoming > 1 (Except End Events)

When you find an element with `incoming.length > 1`:

**a) MANDATORY: Add merge gateway BEFORE this element**
- **ID:** `Gateway_Merge_[Purpose]` (e.g., `Gateway_Merge_BeforePackage`)
- **Type:** Exclusive (XOR) for mutually exclusive paths, Parallel (AND) for synchronization
- **Name:** Leave empty or descriptive (e.g., "Merge Rejection Paths")

**b) Update your element list:**
- Add new merge gateway to the table
- Update target element's incoming count to 1 (now comes from merge gateway)
- Update merge gateway's incoming count to match original target's count

**c) Update flow mapping:**
- All flows that previously pointed to target → now point to merge gateway
- New single flow: merge gateway → target element

**Example correction for Row 7:**

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

**Updated table:**
| # | Element ID | Type | Step # | Incoming Count | Sources | Action |
|---|------------|------|--------|----------------|---------|--------|
| 6a | Gateway_Merge_BeforePackage | Gateway (Merge) | - | **2** | Gateway_LibAvail, Gateway_Correction | ✅ Merge gateway |
| 7 | Task_CreatePackage | Task | 2.2.4 | 1 | Gateway_Merge_BeforePackage | ✅ OK |

### Step 4: Verify Result - Final Validation

Check your updated table:
- ✅ ALL tasks have exactly ONE incoming flow
- ✅ ALL gateways (except merge gateways) have exactly ONE incoming flow
- ✅ ALL events (except End Events) have exactly ONE incoming flow
- ✅ ONLY End Events and Merge Gateways have multiple incoming flows

### Step 5: ONLY AFTER Table is Complete

✅ Now proceed to creating XML elements
✅ You have complete list including all necessary merge gateways
✅ You've prevented Multi-Merge before it enters XML

## Alternative: Per-Flow Algorithm

**Use when:** Creating flows incrementally (not recommended, but sometimes necessary).

**Apply this algorithm to EACH flow you create:**

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
      - ID: Gateway_Merge[Purpose]
      - Name: Descriptive merge name (optional)
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

## Implementation Example (Pseudo-Code)

```javascript
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

## Quick Decision Tree

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

## Real-World Example

**Scenario:** Process with numbered steps from user documentation.

**Requirements Analysis:**
```
2.1.1: Check Requirements (Task)
2.1.2: New Request? (Gateway) → Yes/No
2.2.1: Library Available? (Gateway) → Yes/No
2.2.2: Use Library (Task) - if Yes from 2.2.1
2.2.3: Correction Needed? (Gateway) - after 2.2.2 → Yes/No
2.2.4: Create/Update Package (Task) - if No from 2.2.1 OR Yes from 2.2.3
```

**Phase 0 Analysis:**

Task_CreatePackage (2.2.4) receives flows from:
1. Gateway_LibAvail (No) - when library not available
2. Gateway_Correction (Yes) - when correction needed

**Incoming count = 2 → ⚠️ MULTI-MERGE DETECTED**

**Solution:** Add `Gateway_Merge_BeforePackage` before Task 2.2.4:

```xml
<!-- Merge gateway inserted -->
<bpmn:exclusiveGateway id="Gateway_Merge_BeforePackage">
  <bpmn:incoming>Flow_LibAvail_No</bpmn:incoming>
  <bpmn:incoming>Flow_Correction_Yes</bpmn:incoming>
  <bpmn:outgoing>Flow_ToPackage</bpmn:outgoing>
</bpmn:exclusiveGateway>

<!-- Task now has single incoming -->
<bpmn:userTask id="Task_CreatePackage" name="2.2.4 Create/Update Package">
  <bpmn:incoming>Flow_ToPackage</bpmn:incoming>
  <bpmn:outgoing>Flow_Package_Next</bpmn:outgoing>
</bpmn:userTask>
```

**Result:** Task has exactly 1 incoming flow ✅

## Common Mistakes This Phase Prevents

❌ **Checking only gateways** - Tasks can also have Multi-Merge!
❌ **Checking "by memory"** - Easy to miss elements in complex processes
❌ **Skipping the table** - "I'll just be careful" doesn't work consistently
❌ **Validating after XML creation** - Fixing is harder than preventing

## Validation Checklist

After completing Phase 0:

```
For each row in your table:
□ Start Event: incoming = 0 ✅
□ Tasks: incoming = 1 (except for new merge gateways being added)
□ Gateways (non-merge): incoming = 1
□ Merge Gateways: incoming ≥ 2 ✅
□ End Events: incoming ≥ 1 ✅ (can be any number)
□ No element (except End/Merge) has incoming > 1
```

## Remember

> **"If you're about to create a 2nd incoming flow to a task/event (except end event), STOP and insert a merge gateway first."**

**Trust the systematic process, not your memory.**

---

**Related Anti-Patterns:**
- See `antipatterns-full.md` Section 9 for detailed Multi-Merge examples
- See `gateway-rules-and-antipatterns.md` for token semantics

**Back to:** `SKILL.md` Phase 0 section
