# BPMN Gateway Rules & Anti-Patterns Guide

Complete reference for all gateway types: how to open, split, merge, and close them correctly.

**Date:** 2025-12-08
**Based on:** BPMN 2.0 specification, industry research, and community best practices

---

## Table of Contents

1. [Gateway Types Overview](#gateway-types-overview)
2. [Exclusive Gateway (XOR)](#exclusive-gateway-xor)
3. [Parallel Gateway (AND)](#parallel-gateway-and)
4. [Inclusive Gateway (OR)](#inclusive-gateway-or)
5. [Event-Based Gateway](#event-based-gateway)
6. [Complex Gateway](#complex-gateway)
7. [Gateway Balancing Rules](#gateway-balancing-rules)
8. [Universal Anti-Patterns](#universal-anti-patterns)
9. [Token Semantics](#token-semantics)

---

## Gateway Types Overview

BPMN 2.0 provides **5 types of gateways** to model different routing behaviors:

| Gateway Type | Symbol | Purpose | Behavior |
|-------------|---------|---------|----------|
| **Exclusive (XOR)** | Empty diamond ◇ | Decision point | **ONE** path selected |
| **Parallel (AND)** | Diamond with + | Fork/synchronize | **ALL** paths taken |
| **Inclusive (OR)** | Diamond with ○ | Conditional fork | **ONE OR MORE** paths |
| **Event-Based** | Diamond with pentagon | Wait for event | First event triggers path |
| **Complex** | Diamond with * | Advanced logic | Custom conditions |

---

## Exclusive Gateway (XOR)

### Symbol & Naming
```
    ◇
   /?\    ← Must be phrased as a question
  /   \
```
**Name examples:** "Order Valid?", "Credit Approved?", "Amount > $1000?"

### How It Works

**Split (Decision):**
- Evaluates conditions on outgoing flows
- Takes **ONLY ONE** path (first true condition found)
- Order of evaluation is **non-deterministic** per BPMN spec

**Merge (Convergence):**
- Waits for **one** incoming token
- Immediately passes token through
- **No synchronization needed**

### Split Rules ✅

```xml
<!-- CORRECT: With default flow -->
<bpmn:exclusiveGateway id="Gateway_Decision" name="Amount > $1000?"
  default="Flow_Small">
  <bpmn:outgoing>Flow_Large</bpmn:outgoing>
  <bpmn:outgoing>Flow_Small</bpmn:outgoing>  <!-- default -->
</bpmn:exclusiveGateway>

<bpmn:sequenceFlow id="Flow_Large" name="Yes" sourceRef="Gateway_Decision">
  <bpmn:conditionExpression>${amount > 1000}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<bpmn:sequenceFlow id="Flow_Small" name="No" sourceRef="Gateway_Decision" />
```

**Requirements:**
1. ✅ **ALWAYS define a default flow** (the "else" case)
2. ✅ Conditions must be **mutually exclusive** (only one can be true)
3. ✅ Conditions must be **complete** (cover all possible cases)
4. ✅ Gateway must be named as a **question**

### Common Mistakes ❌

#### 1. **Missing Default Flow** ❌ CRITICAL
```xml
<!-- WRONG: No default flow -->
<bpmn:exclusiveGateway id="Gateway_Risk">
  <bpmn:outgoing>Flow_High</bpmn:outgoing>
  <bpmn:outgoing>Flow_Low</bpmn:outgoing>
</bpmn:exclusiveGateway>

<bpmn:sequenceFlow id="Flow_High" sourceRef="Gateway_Risk">
  <bpmn:conditionExpression>${risk == 'HIGH'}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<bpmn:sequenceFlow id="Flow_Low" sourceRef="Gateway_Risk">
  <bpmn:conditionExpression>${risk == 'LOW'}</bpmn:conditionExpression>
</bpmn:sequenceFlow>
```

**Problem:** If `risk == 'MEDIUM'`, neither condition is true → **DEADLOCK**
**Impact:** Process instance gets stuck, creates incident in engine
**Fix:** Add `default` attribute to one flow (the "catch-all" case)

#### 2. **Non-Mutually Exclusive Conditions** ⚠️
```xml
<!-- WRONG: Overlapping conditions -->
<bpmn:sequenceFlow id="Flow_Student" sourceRef="Gateway_Discount">
  <bpmn:conditionExpression>${age < 25}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<bpmn:sequenceFlow id="Flow_Premium" sourceRef="Gateway_Discount">
  <bpmn:conditionExpression>${loyaltyYears > 5}</bpmn:conditionExpression>
</bpmn:sequenceFlow>
```

**Problem:** A 22-year-old with 6 years loyalty meets **BOTH** conditions
**Impact:** Non-deterministic behavior (which path chosen is unpredictable)
**Fix:** Make conditions mutually exclusive with explicit precedence

#### 3. **Incomplete Conditions** ❌
```xml
<!-- WRONG: Doesn't cover all cases -->
<bpmn:sequenceFlow id="Flow_Child" sourceRef="Gateway_Age">
  <bpmn:conditionExpression>${age < 18}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<bpmn:sequenceFlow id="Flow_Adult" sourceRef="Gateway_Age">
  <bpmn:conditionExpression>${age >= 18 && age < 65}</bpmn:conditionExpression>
</bpmn:sequenceFlow>
<!-- Missing: age >= 65 case! -->
```

**Problem:** Senior citizens (age ≥ 65) have no valid path
**Fix:** Add `Flow_Senior` with default or condition `${age >= 65}`

#### 4. **Conditional Flow from Task** ❌ ANTI-PATTERN
```xml
<!-- WRONG: Task with conditional outgoing flow -->
<bpmn:userTask id="Task_Review">
  <bpmn:outgoing>Flow_Approve</bpmn:outgoing>
  <bpmn:outgoing>Flow_Reject</bpmn:outgoing>  <!-- ❌ -->
</bpmn:userTask>

<bpmn:sequenceFlow id="Flow_Reject" sourceRef="Task_Review">
  <bpmn:conditionExpression>${!approved}</bpmn:conditionExpression>  <!-- ❌ -->
</bpmn:sequenceFlow>
```

**Problem:** **Implicit Gateway** - decision logic hidden in flow
**This is the error we found in employee-onboarding.bpmn!**
**Fix:** Always use explicit gateway after task

```xml
<!-- CORRECT -->
<bpmn:userTask id="Task_Review">
  <bpmn:outgoing>Flow_ToGateway</bpmn:outgoing>  <!-- ✅ Single outgoing -->
</bpmn:userTask>

<bpmn:exclusiveGateway id="Gateway_Approved" name="Approved?">
  <bpmn:incoming>Flow_ToGateway</bpmn:incoming>
  <bpmn:outgoing>Flow_Approve</bpmn:outgoing>
  <bpmn:outgoing>Flow_Reject</bpmn:outgoing>
</bpmn:exclusiveGateway>
```

### Merge Rules ✅

```xml
<!-- CORRECT: Exclusive merge (simple convergence) -->
<bpmn:exclusiveGateway id="Gateway_Merge">
  <bpmn:incoming>Flow_PathA</bpmn:incoming>
  <bpmn:incoming>Flow_PathB</bpmn:incoming>
  <bpmn:incoming>Flow_PathC</bpmn:incoming>
  <bpmn:outgoing>Flow_Continue</bpmn:outgoing>
</bpmn:exclusiveGateway>
```

**Behavior:**
- First token to arrive passes through immediately
- Other paths never send tokens (XOR split guarantees only one)
- **No waiting/synchronization**

**When to use merge gateway:**
- After XOR split when paths converge
- Optional but recommended for clarity
- Can be omitted if flows converge directly into task

---

## Parallel Gateway (AND)

### Symbol & Naming
```
    +
   /|\    ← Named for action: "Start Checks", "Sync Results"
  / | \
```
**Name examples:** "Fork Approvals", "Synchronize Tasks", "Begin Parallel Work"

### How It Works

**Split (Fork):**
- Creates **multiple tokens** (one per outgoing path)
- **ALL** paths executed simultaneously
- No conditions evaluated

**Join (Synchronization):**
- Waits for **ALL** incoming tokens
- Only proceeds when all paths complete
- **Critical for avoiding race conditions**

### Split Rules ✅

```xml
<!-- CORRECT: Parallel fork -->
<bpmn:parallelGateway id="Gateway_Fork" name="Start Background Checks">
  <bpmn:incoming>Flow_ValidApplication</bpmn:incoming>
  <bpmn:outgoing>Flow_ToCreditCheck</bpmn:outgoing>
  <bpmn:outgoing>Flow_ToIncomeVerify</bpmn:outgoing>
  <bpmn:outgoing>Flow_ToEmploymentCheck</bpmn:outgoing>
</bpmn:parallelGateway>
```

**Requirements:**
1. ✅ **No conditions** on outgoing flows (all paths always taken)
2. ✅ Name describes the parallel action
3. ✅ Must have corresponding **join gateway** (99% of cases)

### Join Rules ✅

```xml
<!-- CORRECT: Parallel join (synchronization) -->
<bpmn:parallelGateway id="Gateway_Join" name="Synchronize Checks">
  <bpmn:incoming>Flow_CreditDone</bpmn:incoming>
  <bpmn:incoming>Flow_IncomeDone</bpmn:incoming>
  <bpmn:incoming>Flow_EmploymentDone</bpmn:incoming>
  <bpmn:outgoing>Flow_AllComplete</bpmn:outgoing>
</bpmn:parallelGateway>
```

**Behavior:**
- Waits until **ALL** three tokens arrive
- Only then creates one token on `Flow_AllComplete`
- If one path takes 5ms and another 5 hours, waits for the 5-hour path

### Common Mistakes ❌

#### 1. **Unbalanced Gateways** ❌ DEADLOCK
```xml
<!-- WRONG: Split with Parallel, Join with Exclusive -->
<bpmn:parallelGateway id="Gateway_Fork">  <!-- Creates 3 tokens -->
  <bpmn:outgoing>Flow_A</bpmn:outgoing>
  <bpmn:outgoing>Flow_B</bpmn:outgoing>
  <bpmn:outgoing>Flow_C</bpmn:outgoing>
</bpmn:parallelGateway>

<!-- ... paths A, B, C ... -->

<bpmn:exclusiveGateway id="Gateway_BadJoin">  <!-- ❌ Expects 1 token! -->
  <bpmn:incoming>Flow_A</bpmn:incoming>
  <bpmn:incoming>Flow_B</bpmn:incoming>
  <bpmn:incoming>Flow_C</bpmn:incoming>
  <bpmn:outgoing>Flow_Next</bpmn:outgoing>
</bpmn:exclusiveGateway>
```

**Problem:**
- Parallel split creates **3 tokens**
- Exclusive merge expects **1 token**, passes first arrival through
- **2 tokens remain stuck** → eventual deadlock or duplicate execution
- **This causes 28% of gateway errors per research!**

**Fix:** Use matching parallel join gateway

#### 2. **Missing Synchronization** ❌ RACE CONDITION
```xml
<!-- WRONG: No join gateway -->
<bpmn:parallelGateway id="Gateway_Fork">
  <bpmn:outgoing>Flow_TaskA</bpmn:outgoing>
  <bpmn:outgoing>Flow_TaskB</bpmn:outgoing>
</bpmn:parallelGateway>

<bpmn:sequenceFlow id="Flow_TaskA" targetRef="Task_A" />
<bpmn:sequenceFlow id="Flow_TaskB" targetRef="Task_B" />

<!-- Both tasks flow directly to next task without sync -->
<bpmn:sequenceFlow sourceRef="Task_A" targetRef="Task_Final" />
<bpmn:sequenceFlow sourceRef="Task_B" targetRef="Task_Final" />  <!-- ❌ Multi-merge! -->
```

**Problem:** **Multi-merge** - `Task_Final` receives 2 tokens
**Impact:** Task executes **TWICE** (once per token)
**Fix:** Add parallel join before `Task_Final`

```xml
<!-- CORRECT: With synchronization -->
<bpmn:parallelGateway id="Gateway_Join" name="Sync Tasks">
  <bpmn:incoming>Flow_FromA</bpmn:incoming>
  <bpmn:incoming>Flow_FromB</bpmn:incoming>
  <bpmn:outgoing>Flow_ToFinal</bpmn:outgoing>
</bpmn:parallelGateway>

<bpmn:sequenceFlow sourceRef="Gateway_Join" targetRef="Task_Final" />
```

#### 3. **Asymmetric Paths** ⚠️
```xml
<!-- Risky but sometimes valid -->
<bpmn:parallelGateway id="Gateway_Fork">
  <bpmn:outgoing>Flow_ToTaskA</bpmn:outgoing>
  <bpmn:outgoing>Flow_ToTaskB</bpmn:outgoing>
  <bpmn:outgoing>Flow_ToTaskC</bpmn:outgoing>
</bpmn:parallelGateway>

<!-- Path A: Task_A → End_A (terminates!) -->
<!-- Path B: Task_B → continues -->
<!-- Path C: Task_C → continues -->

<bpmn:parallelGateway id="Gateway_Join">
  <bpmn:incoming>Flow_FromB</bpmn:incoming>
  <bpmn:incoming>Flow_FromC</bpmn:incoming>  <!-- Only 2 paths! -->
  <bpmn:outgoing>Flow_Next</bpmn:outgoing>
</bpmn:parallelGateway>
```

**When this is OK:**
- Path A legitimately ends early (e.g., notification sent independently)
- Join only synchronizes paths that continue

**When this is WRONG:**
- Modeler forgot to connect Path A
- Creates unintended early termination

**Best Practice:** Document why paths are asymmetric with annotations

#### 4. **Conditions on Parallel Flows** ❌
```xml
<!-- WRONG: Conditional flow from parallel gateway -->
<bpmn:parallelGateway id="Gateway_Fork">
  <bpmn:outgoing>Flow_Always</bpmn:outgoing>
  <bpmn:outgoing>Flow_Conditional</bpmn:outgoing>
</bpmn:parallelGateway>

<bpmn:sequenceFlow id="Flow_Conditional" sourceRef="Gateway_Fork">
  <bpmn:conditionExpression>${urgent}</bpmn:conditionExpression>  <!-- ❌ -->
</bpmn:sequenceFlow>
```

**Problem:** Parallel gateway semantics = **all** paths taken
**Impact:** Condition is ignored! Both flows always execute
**Fix:** Use inclusive gateway (OR) instead if you need conditional parallelism

---

## Inclusive Gateway (OR)

### Symbol & Naming
```
    ○
   /?\    ← Must be phrased as a question
  /   \
```
**Name examples:** "Which Reviews Needed?", "Select Applicable Checks?"

### How It Works

**Split (Conditional Fork):**
- Evaluates **ALL** conditions
- Activates **ONE OR MORE** paths (where conditions are true)
- Creates multiple tokens (like parallel), but conditionally

**Join (Conditional Synchronization):**
- Waits for tokens from **all activated paths**
- **Must determine which paths were activated** (complex!)
- Only proceeds when all expected tokens arrive

### Split Rules ✅

```xml
<!-- CORRECT: Inclusive split -->
<bpmn:inclusiveGateway id="Gateway_ReviewTypes"
  name="Which Reviews Needed?">
  <bpmn:outgoing>Flow_LegalReview</bpmn:outgoing>
  <bpmn:outgoing>Flow_FinancialReview</bpmn:outgoing>
  <bpmn:outgoing>Flow_TechnicalReview</bpmn:outgoing>
</bpmn:inclusiveGateway>

<bpmn:sequenceFlow id="Flow_LegalReview" name="Contract involved">
  <bpmn:conditionExpression>${hasContract}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<bpmn:sequenceFlow id="Flow_FinancialReview" name="Amount > $10k">
  <bpmn:conditionExpression>${amount > 10000}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<bpmn:sequenceFlow id="Flow_TechnicalReview" name="New technology" default="true">
  <!-- Default: at least one path must be taken -->
</bpmn:sequenceFlow>
```

**Scenarios:**
- `hasContract=true, amount=5000, newTech=false` → Legal only (1 token)
- `hasContract=false, amount=15000, newTech=true` → Financial + Technical (2 tokens)
- `hasContract=true, amount=20000, newTech=true` → All three (3 tokens)

### Join Rules ✅

```xml
<!-- CORRECT: Inclusive join -->
<bpmn:inclusiveGateway id="Gateway_SyncReviews"
  name="Synchronize Reviews">
  <bpmn:incoming>Flow_LegalDone</bpmn:incoming>
  <bpmn:incoming>Flow_FinancialDone</bpmn:incoming>
  <bpmn:incoming>Flow_TechnicalDone</bpmn:incoming>
  <bpmn:outgoing>Flow_AllReviewsComplete</bpmn:outgoing>
</bpmn:inclusiveGateway>
```

**Behavior:**
- Gateway "remembers" which paths were activated at split
- Waits **only for tokens from activated paths**
- If only Legal was activated, waits only for Legal token

### Common Mistakes ❌

#### 1. **Token Termination Between OR Split and Join** ❌ DEADLOCK
```xml
<!-- WRONG: End event in OR path -->
<bpmn:inclusiveGateway id="Gateway_Split">
  <bpmn:outgoing>Flow_ProcessA</bpmn:outgoing>
  <bpmn:outgoing>Flow_ProcessB</bpmn:outgoing>
</bpmn:inclusiveGateway>

<!-- Path A: leads to End event -->
<bpmn:sequenceFlow sourceRef="Task_A" targetRef="End_PathA" />  <!-- ❌ -->

<!-- Path B: leads to Join -->
<bpmn:sequenceFlow sourceRef="Task_B" targetRef="Gateway_Join" />

<bpmn:inclusiveGateway id="Gateway_Join">  <!-- Waits forever for Path A token! -->
  <bpmn:incoming>Flow_FromA</bpmn:incoming>
  <bpmn:incoming>Flow_FromB</bpmn:incoming>
</bpmn:inclusiveGateway>
```

**Problem:**
- Split activates both paths → creates 2 tokens
- Path A terminates early → token consumed by End event
- Join waits for 2 tokens → **DEADLOCK** (only receives 1)

**This is the #1 error with inclusive gateways!**

**Fix:** Remove end events between OR split and join, or redesign process

#### 2. **Complexity Across Pages** ⚠️
```
Page 1: OR Split
  ↓
(many tasks and gateways)
  ↓
Page 3: OR Join (which paths were activated??)
```

**Problem:** Hard to trace which paths will produce tokens
**Impact:** Difficult to verify correctness, prone to modeling errors
**Recommendation:** **Avoid inclusive gateways in complex models**
**Alternative:** Use parallel gateway + exclusive gateways for clarity

#### 3. **Forgetting Default Path** ❌
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
```

**Problem:** If both `typeA` and `typeB` are false → **NO paths activated** → DEADLOCK
**Fix:** Mark one flow as `default="true"` to ensure at least one path taken

---

## Event-Based Gateway

### Symbol & Naming
```
    ⬡
   /|\    ← Named for waiting action: "Wait for Response"
  / | \
```
**Name examples:** "Wait for Customer Response", "First Event Wins"

### How It Works

**Split (Event Wait):**
- Must be followed by **catching events** (not tasks!)
- Waits for **first event** to occur
- **Only one** path taken (like exclusive, but event-driven)
- Other events are discarded

**No Join:**
- Event-based gateways don't have merges
- Behaves like exclusive gateway after event is caught

### Rules ✅

```xml
<!-- CORRECT: Event-based gateway -->
<bpmn:eventBasedGateway id="Gateway_WaitResponse"
  name="Wait for Customer">
  <bpmn:outgoing>Flow_ToEmailReceived</bpmn:outgoing>
  <bpmn:outgoing>Flow_ToPhoneCall</bpmn:outgoing>
  <bpmn:outgoing>Flow_ToTimeout</bpmn:outgoing>
</bpmn:eventBasedGateway>

<!-- Must be followed by events, NOT tasks -->
<bpmn:intermediateCatchEvent id="Event_EmailReceived" name="Email Received">
  <bpmn:incoming>Flow_ToEmailReceived</bpmn:incoming>
  <bpmn:messageEventDefinition />
</bpmn:intermediateCatchEvent>

<bpmn:intermediateCatchEvent id="Event_PhoneCall" name="Call Received">
  <bpmn:incoming>Flow_ToPhoneCall</bpmn:incoming>
  <bpmn:messageEventDefinition />
</bpmn:intermediateCatchEvent>

<bpmn:intermediateCatchEvent id="Event_Timeout" name="48h Timeout">
  <bpmn:incoming>Flow_ToTimeout</bpmn:incoming>
  <bpmn:timerEventDefinition />
</bpmn:intermediateCatchEvent>
```

### Common Mistakes ❌

#### 1. **Followed by Tasks** ❌
```xml
<!-- WRONG: Event gateway leading to tasks -->
<bpmn:eventBasedGateway id="Gateway_Wait">
  <bpmn:outgoing>Flow_ToTask</bpmn:outgoing>  <!-- ❌ Must be event! -->
</bpmn:eventBasedGateway>

<bpmn:userTask id="Task_ProcessEmail" />
```

**Problem:** BPMN violation - event-based gateway **requires** events
**Fix:** Insert intermediate catch events before tasks

---

## Complex Gateway

### Symbol & Naming
```
    *
   /?\    ← Named for complex condition
  /   \
```

### When to Use

Use complex gateway for advanced synchronization patterns:
- "Wait for any 2 out of 3 approvals"
- "Proceed when either (A and B) or (C) completes"
- Custom m-out-of-n logic

### Example

```xml
<bpmn:complexGateway id="Gateway_Complex"
  name="2 out of 3 Approvals Needed">
  <bpmn:incoming>Flow_ApprovalA</bpmn:incoming>
  <bpmn:incoming>Flow_ApprovalB</bpmn:incoming>
  <bpmn:incoming>Flow_ApprovalC</bpmn:incoming>
  <bpmn:outgoing>Flow_Proceed</bpmn:outgoing>
  <bpmn:activationCondition xsi:type="bpmn:tFormalExpression">
    ${approvalsReceived >= 2}
  </bpmn:activationCondition>
</bpmn:complexGateway>
```

### ⚠️ Warning

**Complex gateways are rarely supported by execution engines!**
- Camunda: Not supported
- Flowable: Limited support
- Most tools: Skip complex gateways

**Recommendation:** Avoid complex gateways. Use combination of parallel + exclusive instead.

---

## Gateway Balancing Rules

### The Balancing Debate

**Traditional Rule:** "Use matching gateway types for split and join"
```
Parallel Split → Parallel Join ✅
Exclusive Split → Exclusive Merge ✅
Inclusive Split → Inclusive Join ✅
```

**Modern Understanding:** Balancing is not strictly required, but **mismatches are dangerous**

### Safe Patterns ✅

#### 1. Balanced Gateways (Safest)
```
[Start] → Parallel Split → Task A → Parallel Join → [Next]
                        → Task B ↗
```
**Why safe:** Clear token flow, easy to verify

#### 2. Parallel Split → Exclusive Merge (Sometimes OK)
```
[Start] → Parallel Split → Task A → End_A
                        → Task B → Exclusive Merge → [Next]
                        → Task C ↗
```
**When OK:** One path terminates independently
**Risk:** Easy to create deadlocks if join expects all tokens

#### 3. Exclusive Split → Parallel Join (NEVER!)
```
[Start] → Exclusive Split → Task A → Parallel Join ← DEADLOCK!
                         → Task B ↗
```
**Why wrong:**
- Exclusive sends 1 token (to A or B)
- Parallel join waits for 2 tokens → **DEADLOCK**

### Unsafe Patterns ❌

#### Multi-Merge (No Synchronization)
```xml
<!-- WRONG -->
<bpmn:parallelGateway id="Gateway_Fork" />
  <!-- Path A → Task_Final -->
  <!-- Path B → Task_Final -->  <!-- ❌ 2 tokens! -->
```
**Fix:** Add parallel join before `Task_Final`

#### Join-Fork (Gateway does both)
```xml
<!-- ANTI-PATTERN -->
<bpmn:parallelGateway id="Gateway_JoinAndFork">
  <bpmn:incoming>Flow_A</bpmn:incoming>
  <bpmn:incoming>Flow_B</bpmn:incoming>  <!-- Join -->
  <bpmn:outgoing>Flow_C</bpmn:outgoing>
  <bpmn:outgoing>Flow_D</bpmn:outgoing>  <!-- Fork -->
</bpmn:parallelGateway>
```
**Why wrong:** Confusing semantics, hard to understand token flow
**Fix:** Use two separate gateways

---

## Universal Anti-Patterns

### 1. Gateway Sending Message ❌
```xml
<!-- WRONG: Gateway with outgoing message flow -->
<bpmn:exclusiveGateway id="Gateway_Decision">
  <bpmn:outgoing>Flow_SequenceToTask</bpmn:outgoing>
  <bpmn:messageFlow id="Flow_ToExternal" />  <!-- ❌ -->
</bpmn:exclusiveGateway>
```
**Problem:** Only activities can send/receive messages, not gateways
**Fix:** Add task after gateway to send message

### 2. Gateway After Gateway (Decision After Decision) ⚠️
```xml
<bpmn:exclusiveGateway id="Gateway_First" />
<bpmn:sequenceFlow sourceRef="Gateway_First" targetRef="Gateway_Second" />
<bpmn:exclusiveGateway id="Gateway_Second" />  <!-- ⚠️ -->
```
**When this is wrong:** Could be combined into one gateway
**When this is OK:** Different decisions at different points (like our loan-approval Low Risk bypass)

### 3. Unbalanced Fork/Join Count ❌
```
Parallel Split (3 paths) → Parallel Join (2 paths)  ❌ Token lost!
```
**Fix:** Ensure same number of paths, or document asymmetry

### 4. Loops Without Conditions ❌
```xml
<bpmn:exclusiveGateway id="Gateway_Loop" default="Flow_Loop">
  <bpmn:outgoing>Flow_Loop</bpmn:outgoing>  <!-- Back to start -->
  <bpmn:outgoing>Flow_Exit</bpmn:outgoing>
</bpmn:exclusiveGateway>

<bpmn:sequenceFlow id="Flow_Loop" targetRef="Previous_Task" />
<!-- No condition! Infinite loop! -->
```
**Fix:** Add condition on loop flow or exit flow

---

## Token Semantics

### What are Tokens?

Tokens represent the **execution state** moving through the process:
- Start event creates 1 token
- Tasks consume 1 token, produce 1 token
- Gateways can create/consume/wait for multiple tokens

### Token Rules by Gateway Type

| Gateway | Split Tokens | Join Behavior |
|---------|-------------|---------------|
| **Exclusive** | 1 token → 1 path | Pass through (no wait) |
| **Parallel** | 1 token → N tokens (all paths) | Wait for N tokens |
| **Inclusive** | 1 token → 1 to N tokens (conditional) | Wait for activated tokens |
| **Event-Based** | 1 token → 1 path (first event) | N/A (like exclusive) |

### Token Flow Examples

#### Exclusive Gateway
```
Start[1 token] → Gateway_XOR[1 token in]
                   ↓
                Path A: 1 token (if condition true)
                Path B: 0 tokens
                   ↓
                Gateway_Merge[1 token] → Next[1 token]
```

#### Parallel Gateway
```
Start[1 token] → Gateway_AND[1 token in]
                   ↓
                Path A: 1 token ────┐
                Path B: 1 token ────┼→ Gateway_Join[waits for 2]
                Path C: 1 token ────┘
                   ↓
                Next[1 token]
```

#### Inclusive Gateway
```
Start[1 token] → Gateway_OR[1 token in, evaluates all conditions]
                   ↓
                Path A: 1 token (condition true) ────┐
                Path B: 0 tokens (condition false)   ├→ Gateway_Join[waits for 1]
                Path C: 1 token (condition true) ────┘
                   ↓
                Next[1 token]
```

### Token Deadlock Scenarios

1. **Missing token:** Join expects 3, receives 2 → waits forever
2. **Terminated token:** Path ends early, join never receives expected token
3. **Wrong gateway type:** Exclusive join after parallel split

---

## Quick Reference Table

| Scenario | Correct Gateway | Wrong Gateway | Consequence |
|----------|----------------|---------------|-------------|
| Choose 1 of N paths | Exclusive (XOR) | Parallel | All paths execute |
| Execute all N paths | Parallel (AND) | Exclusive | Only 1 path executes |
| Execute some of N paths | Inclusive (OR) | Exclusive | Only 1 path executes |
| Wait for first event | Event-Based | Exclusive | All events trigger |
| Join after parallel split | Parallel | Exclusive | Deadlock or multi-execution |
| Join after exclusive split | Exclusive or none | Parallel | Deadlock |
| Join after inclusive split | Inclusive | Parallel | Deadlock |

---

## Validation Checklist

### For Every Exclusive Gateway:
- [ ] Gateway named as question?
- [ ] Has default flow defined?
- [ ] Conditions mutually exclusive?
- [ ] All possible values covered?
- [ ] No conditional flows from tasks?

### For Every Parallel Gateway:
- [ ] Split has matching join?
- [ ] Same number of paths in/out (or documented)?
- [ ] No conditions on outgoing flows?
- [ ] No multi-merge (missing join)?
- [ ] Join waits for ALL paths?

### For Every Inclusive Gateway:
- [ ] Has default flow or guaranteed activation?
- [ ] No end events between split and join?
- [ ] Join matches split paths?
- [ ] Complexity is necessary (can't use simpler pattern)?

### For Every Event-Based Gateway:
- [ ] Followed by events (not tasks)?
- [ ] Events are mutually exclusive?
- [ ] Timeout event included (recommended)?

---

## Sources & References

Based on research from:
- [Visual Paradigm: BPMN Gateway Types](https://www.visual-paradigm.com/guide/bpmn/bpmn-gateway-types/)
- [Heflo: BPMN Gateways Best Practices](https://www.heflo.com/blog/bpmn-gateways)
- [Camunda: BPMN 2.0 Reference](https://camunda.com/bpmn/reference/)
- [Flokzu: Inclusive Gateway Errors](https://docs.flokzu.com/en/article/common-errors-and-recommendations-when-using-inclusive-gateways-and-parallelism-1drj4nh/)
- [BPM Tips: Gateway Token Semantics](https://bpmtips.com/parallel-gateways-and-and-tokens/)
- [ARIS Community: Gateway Split and Merge](https://ariscommunity.com/users/yong-yeow-ching/2021-03-11-gateway-usage-when-come-split-and-merge)
- [Good e-Learning: Fork-Join-Branch-Merge](https://goodelearning.com/bpmn-2-0-terms-explained-fork-join-branch-merge/)
- [Modeling Guidelines: Multi-Merge Absence](https://www.modeling-guidelines.org/guidelines/absence-of-multi-merges/)

---

**Last Updated:** 2025-12-08
**BPMN Version:** 2.0
**Skill Version:** 3.1
