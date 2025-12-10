# BPMN Gateway Complete Guide

**Purpose:** Comprehensive reference for all gateway types with rules, anti-patterns, and token semantics.

**Replaces:** `gateway-rules-and-antipatterns.md` (EN) + `gateway-cheatsheet-ru.md` (RU)

**Languages:** English (primary) with Russian terms where helpful

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
10. [Quick Reference](#quick-reference)

---

## Gateway Types Overview

BPMN 2.0 provides **5 types of gateways** (шлюзы):

| Type | Symbol | When to Use | Behavior |
|------|--------|-------------|----------|
| **Exclusive (XOR)** | ◇ empty | Decision point - choose ONE path | Exactly **ONE** path activates |
| **Parallel (AND)** | ◇ with + | Fork/synchronize - ALL paths | **ALL** paths execute simultaneously |
| **Inclusive (OR)** | ◇ with ○ | Conditional branching | **ONE OR MORE** paths activate |
| **Event-Based** | ◇ with ⬡ | Wait for external event | First event determines path |
| **Complex** | ◇ with * | Advanced custom logic | ⚠️ NOT supported by engines! |

---

## Exclusive Gateway (XOR)

**Purpose:** Decision point where **exactly ONE** outgoing path is taken based on conditions.

**Russian:** Эксклюзивный шлюз, выбор одного пути

### Symbol & Naming
```
    ◇
   /?\    ← MUST be phrased as a question
  /   \
```
**Name examples:** "Approved?", "Amount > $1000?", "Documents Complete?"

### How It Works

**Split (Decision) / Разветвление:**
- Evaluates conditions on outgoing flows
- Takes **ONLY ONE** path (first true condition, order is non-deterministic)
- **Default flow** taken if no condition is true

**Merge (Convergence) / Слияние:**
- Waits for **one** incoming token
- Immediately passes token through
- **No synchronization** (unlike parallel join)

### XML Structure

```xml
<!-- CORRECT: With default flow -->
<bpmn:exclusiveGateway id="Gateway_Decision"
  name="Amount > $1000?"
  default="Flow_Default">  <!-- ✅ MANDATORY: default flow -->
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Large</bpmn:outgoing>
  <bpmn:outgoing>Flow_Default</bpmn:outgoing>
</bpmn:exclusiveGateway>

<!-- Conditional flow -->
<bpmn:sequenceFlow id="Flow_Large" name="Yes"
  sourceRef="Gateway_Decision" targetRef="Task_Process">
  <bpmn:conditionExpression xsi:type="bpmn:tFormalExpression">
    ${amount > 1000}
  </bpmn:conditionExpression>
</bpmn:sequenceFlow>

<!-- Default flow (NO condition) -->
<bpmn:sequenceFlow id="Flow_Default" name="No"
  sourceRef="Gateway_Decision" targetRef="Task_Approve" />
```

### Mandatory Rules ✅

1. **ALWAYS define default flow** (`default="Flow_ID"` attribute)
2. **Conditions must be mutually exclusive** (only one can be true at a time)
3. **Conditions must be complete** (cover all possible values)
4. **Name as question** ("Approved?", not "Check Approval")

### Critical Mistakes ❌

#### Mistake 1: Missing Default Flow → DEADLOCK

```xml
<!-- ❌ WRONG: No default flow -->
<bpmn:exclusiveGateway id="Gateway_Risk">
  <bpmn:outgoing>Flow_High</bpmn:outgoing>
  <bpmn:outgoing>Flow_Low</bpmn:outgoing>
</bpmn:exclusiveGateway>

<bpmn:sequenceFlow id="Flow_High">
  <bpmn:conditionExpression>${risk == 'HIGH'}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<bpmn:sequenceFlow id="Flow_Low">
  <bpmn:conditionExpression>${risk == 'LOW'}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<!-- If risk='MEDIUM', NO condition matches → PROCESS STUCK! -->
```

**Fix:** Add `default` attribute pointing to one flow (the "else" case).

#### Mistake 2: Overlapping Conditions → Non-Deterministic

```xml
<!-- ❌ WRONG: Both conditions can be true -->
<bpmn:sequenceFlow id="Flow_Student">
  <bpmn:conditionExpression>${age < 25}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<bpmn:sequenceFlow id="Flow_Premium">
  <bpmn:conditionExpression>${loyaltyYears > 5}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<!-- If age=22 AND loyaltyYears=6, BOTH conditions true! Which path? -->
```

**Fix:** Make conditions mutually exclusive:
```xml
<bpmn:conditionExpression>${age < 25 && loyaltyYears <= 5}</bpmn:conditionExpression>
<bpmn:conditionExpression>${loyaltyYears > 5}</bpmn:conditionExpression>
```

#### Mistake 3: Incomplete Conditions → Missing Cases

```xml
<!-- ❌ WRONG: Doesn't cover age >= 65 -->
<bpmn:conditionExpression>${age < 18}</bpmn:conditionExpression>
<bpmn:conditionExpression>${age >= 18 && age < 65}</bpmn:conditionExpression>
<!-- Where are seniors?! -->
```

**Fix:** Add missing condition OR use default flow for last case.

---

## Parallel Gateway (AND)

**Purpose:** Fork execution into multiple parallel paths, then synchronize them back together.

**Russian:** Параллельный шлюз, все пути одновременно

### Symbol & Naming
```
    ◇
   /+\    ← Plus sign indicates parallelism
  /   \
```
**Name examples:** "Start Background Checks", "Synchronize Results" (or leave empty for merge)

### How It Works

**Split (Fork) / Разветвление:**
- Creates **multiple tokens** (one per outgoing path)
- **ALL** paths execute simultaneously
- **NO conditions** on outgoing flows (unconditional split)

**Join (Synchronization) / Синхронизация:**
- Waits for **ALL** incoming tokens
- Once all tokens arrive, produces **ONE** outgoing token
- **Blocks** until synchronization complete

### XML Structure

```xml
<!-- FORK: Creates multiple tokens -->
<bpmn:parallelGateway id="Gateway_Fork" name="Start Checks">
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_ToCredit</bpmn:outgoing>
  <bpmn:outgoing>Flow_ToIncome</bpmn:outgoing>
  <bpmn:outgoing>Flow_ToCollateral</bpmn:outgoing>
</bpmn:parallelGateway>
<!-- NO conditions! All 3 paths ALWAYS execute -->

<!-- Parallel tasks -->
<bpmn:userTask id="Task_CheckCredit" name="Check Credit" />
<bpmn:userTask id="Task_CheckIncome" name="Verify Income" />
<bpmn:userTask id="Task_CheckCollateral" name="Assess Collateral" />

<!-- JOIN: Waits for ALL 3 tokens -->
<bpmn:parallelGateway id="Gateway_Join" name="Synchronize">
  <bpmn:incoming>Flow_CreditDone</bpmn:incoming>
  <bpmn:incoming>Flow_IncomeDone</bpmn:incoming>
  <bpmn:incoming>Flow_CollateralDone</bpmn:incoming>
  <bpmn:outgoing>Flow_Continue</bpmn:outgoing>
</bpmn:parallelGateway>
```

### Mandatory Rules ✅

1. **Use in pairs:** Fork → ... → Join
2. **Match path counts:** 3 outgoing → 3 incoming (or document asymmetry)
3. **NO conditions on outgoing flows** (all paths always execute)
4. **NO multi-merge:** Don't skip synchronization (see Universal Anti-Patterns)

### Critical Mistakes ❌

#### Mistake 1: Unbalanced Fork/Join → DEADLOCK

```xml
<!-- ❌ WRONG: 3 paths out, only 2 paths in -->
<bpmn:parallelGateway id="Gateway_Fork">
  <bpmn:outgoing>Flow_A</bpmn:outgoing>
  <bpmn:outgoing>Flow_B</bpmn:outgoing>
  <bpmn:outgoing>Flow_C</bpmn:outgoing>
</bpmn:parallelGateway>

<bpmn:parallelGateway id="Gateway_Join">
  <bpmn:incoming>Flow_A</bpmn:incoming>
  <bpmn:incoming>Flow_B</bpmn:incoming>
  <!-- Missing Flow_C! Join waits forever for 3rd token → DEADLOCK -->
</bpmn:parallelGateway>
```

**Fix:** Ensure join has same number of incoming paths as fork has outgoing.

**Exception:** Asymmetric join is OK if one path terminates independently (e.g., fire-and-forget notification), but MUST be documented.

#### Mistake 2: Mismatched Gateway Types → DEADLOCK

```xml
<!-- ❌ WRONG: Parallel fork → Exclusive join -->
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
<!-- First token passes, 2 tokens stuck forever → DEADLOCK -->
```

**Fix:** Use matching gateway types (Parallel fork → Parallel join).

#### Mistake 3: Missing Synchronization (Multi-Merge)

```xml
<!-- ❌ WRONG: No join gateway -->
<bpmn:parallelGateway id="Gateway_Fork">
  <bpmn:outgoing>Flow_ToA</bpmn:outgoing>
  <bpmn:outgoing>Flow_ToB</bpmn:outgoing>
</bpmn:parallelGateway>

<!-- Both tasks flow directly to next task -->
<bpmn:sequenceFlow sourceRef="Task_A" targetRef="Task_Final" />
<bpmn:sequenceFlow sourceRef="Task_B" targetRef="Task_Final" />
<!-- ❌ Task_Final receives 2 tokens → executes TWICE! -->
```

**Fix:** Add parallel join before Task_Final (see `multi-merge-prevention.md`).

---

## Inclusive Gateway (OR)

**Purpose:** Conditional parallelism - activate one or more paths based on conditions.

**Russian:** Включающий шлюз, один или несколько путей

**⚠️ CAUTION:** Most complex gateway, easy to misuse. Prefer Parallel + Exclusive when possible.

### Symbol & Naming
```
    ◇
   /○\    ← Circle indicates "inclusive"
  /   \
```
**Name examples:** "Which Reviews Needed?", "Select Delivery Options"

### How It Works

**Split (Fork) / Разветвление:**
- Evaluates conditions on each outgoing flow
- Activates **ONE OR MORE** paths (all with true conditions)
- **At least one path MUST activate** (use default flow to guarantee)

**Join (Synchronization) / Синхронизация:**
- Waits for **ALL activated tokens** from the split
- **Knows** which paths were activated at split
- Complex synchronization logic (engine-dependent)

### XML Structure

```xml
<!-- SPLIT: Conditional parallelism -->
<bpmn:inclusiveGateway id="Gateway_ReviewTypes"
  name="Which Reviews Needed?"
  default="Flow_Technical">  <!-- ✅ Guarantees at least one path -->
  <bpmn:incoming>Flow_In</bpmn:incoming>
  <bpmn:outgoing>Flow_Legal</bpmn:outgoing>
  <bpmn:outgoing>Flow_Financial</bpmn:outgoing>
  <bpmn:outgoing>Flow_Technical</bpmn:outgoing>
</bpmn:inclusiveGateway>

<!-- Conditional flows -->
<bpmn:sequenceFlow id="Flow_Legal" name="Contract Involved">
  <bpmn:conditionExpression>${hasContract}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<bpmn:sequenceFlow id="Flow_Financial" name="Amount > $10k">
  <bpmn:conditionExpression>${amount > 10000}</bpmn:conditionExpression>
</bpmn:sequenceFlow>

<!-- Default (ensures at least one path) -->
<bpmn:sequenceFlow id="Flow_Technical" name="New Technology" />

<!-- Review tasks -->
<bpmn:userTask id="Task_LegalReview" />
<bpmn:userTask id="Task_FinancialReview" />
<bpmn:userTask id="Task_TechnicalReview" />

<!-- JOIN: Waits for ALL activated paths -->
<bpmn:inclusiveGateway id="Gateway_ReviewJoin" name="All Reviews Complete">
  <bpmn:incoming>Flow_LegalDone</bpmn:incoming>
  <bpmn:incoming>Flow_FinancialDone</bpmn:incoming>
  <bpmn:incoming>Flow_TechnicalDone</bpmn:incoming>
  <bpmn:outgoing>Flow_Continue</bpmn:outgoing>
</bpmn:inclusiveGateway>
```

### Mandatory Rules ✅

1. **MUST have default flow** (or ensure at least one condition is always true)
2. **NO End events between split and join** (causes deadlock)
3. **Match split and join paths** (all potential paths must reach join)
4. **Use sparingly** - prefer Parallel + Exclusive for clarity

### Critical Mistakes ❌

#### Mistake 1: Token Termination Between Split/Join → DEADLOCK

```xml
<!-- ❌ WRONG: Path terminates before join -->
<bpmn:inclusiveGateway id="Gateway_Split">
  <bpmn:outgoing>Flow_ProcessA</bpmn:outgoing>
  <bpmn:outgoing>Flow_ProcessB</bpmn:outgoing>
</bpmn:inclusiveGateway>

<!-- Path A terminates early -->
<bpmn:sequenceFlow sourceRef="Task_A" targetRef="End_PathA" />  <!-- ❌ -->

<!-- Path B goes to join -->
<bpmn:inclusiveGateway id="Gateway_Join">
  <bpmn:incoming>Flow_FromA</bpmn:incoming>  <!-- Never arrives! -->
  <bpmn:incoming>Flow_FromB</bpmn:incoming>
</bpmn:inclusiveGateway>
<!-- If both paths activated, join waits forever → DEADLOCK -->
```

**Fix:** All paths must go to join, then single End after join.

#### Mistake 2: No Default Flow → Deadlock if All Conditions False

```xml
<!-- ❌ WRONG: All conditions could be false -->
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
```

**Fix:** Add `default` attribute to one flow.

---

## Event-Based Gateway

**Purpose:** Wait for external events (messages, timers, signals). First event to occur determines path.

**Russian:** Событийный шлюз, ожидание события

### Symbol
```
    ◇
   /⬡\    ← Pentagon indicates event-based
  /   \
```

### How It Works

- Gateway **blocks** and waits
- **First event** to occur determines which path is taken
- Other events are **ignored** (not consumed)
- Only **ONE** path activates (like Exclusive gateway)

### XML Structure

```xml
<bpmn:eventBasedGateway id="Gateway_Event" name="Wait for Response or Timeout">
  <bpmn:incoming>Flow_RequestSent</bpmn:incoming>
  <bpmn:outgoing>Flow_ToResponse</bpmn:outgoing>
  <bpmn:outgoing>Flow_ToTimeout</bpmn:outgoing>
</bpmn:eventBasedGateway>

<!-- Catch events (not tasks!) -->
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
```

### Rules ✅

1. **Outgoing flows MUST connect to events** (not tasks)
2. **Only catch events** (Message, Timer, Signal, Conditional)
3. **First event wins** (others ignored)
4. **Use for asynchronous scenarios** (waiting for external signals)

---

## Complex Gateway

**Purpose:** Advanced custom routing logic with complex expressions.

**Russian:** Комплексный шлюз

### Symbol
```
    ◇
   /*\    ← Asterisk indicates complexity
  /   \
```

### ⚠️ WARNING

**Complex Gateway is NOT supported by most BPMN engines** (including Camunda, Flowable, Activiti).

**Recommendation:** **DO NOT USE.** Instead:
- Use combination of Exclusive + Parallel + Inclusive gateways
- Use Script Tasks for complex logic
- Use Business Rule Tasks with DMN decision tables

---

## Gateway Balancing Rules

**Golden Rule:** **Every fork gateway MUST have a matching join gateway of the same type.**

### Safe Patterns ✅

| Fork Type | Join Type | Status |
|-----------|-----------|--------|
| Exclusive → Exclusive | ✅ SAFE | Merge is optional (for clarity) |
| Parallel → Parallel | ✅ REQUIRED | MUST synchronize |
| Inclusive → Inclusive | ✅ REQUIRED | MUST synchronize |

### Dangerous Patterns ❌

| Fork Type | Join Type | Problem |
|-----------|-----------|---------|
| Parallel → Exclusive | ❌ DEADLOCK | Join expects 1 token, gets N |
| Exclusive → Parallel | ❌ DEADLOCK | Join expects N tokens, gets 1 |
| Inclusive → Parallel | ❌ DEADLOCK | Join may get wrong token count |
| Parallel → Inclusive | ❌ UNDEFINED | Behavior is engine-dependent |

### Path Count Matching

**Parallel fork with 3 outgoing → Parallel join with 3 incoming**

```
Fork(3) → A ──┐
          B ──┼→ Join(3) → Continue
          C ──┘
```

**Exception:** Asymmetric join acceptable if:
- One path terminates independently (fire-and-forget)
- Documented with comment
- Example: Audit log path that doesn't need synchronization

---

## Universal Anti-Patterns

Anti-patterns that apply to ALL gateway types.

### Anti-Pattern #1: Multi-Merge ❌

**Problem:** Multiple flows converge directly into task/event without merge gateway.

**See:** `multi-merge-prevention.md` and `antipatterns-full.md` Section 9 for details.

**Quick Fix:** Insert merge gateway before element with multiple incoming.

### Anti-Pattern #2: Implicit Gateway ❌

**Problem:** Task with multiple outgoing flows (decision hidden in task).

**See:** `antipatterns-full.md` Section 12 for details.

**Quick Fix:** Insert explicit gateway after task with single outgoing.

### Anti-Pattern #3: Gateway Sending Messages ❌

**Problem:** Gateway directly connected to message throw event.

**Wrong:**
```
Gateway → ○ Message Event
```

**Correct:**
```
Gateway → Task → ○ Message Event
```

**Reason:** Gateways route tokens, they don't perform actions. Sending messages is an action (must be in task).

### Anti-Pattern #4: Unbalanced Gateways ❌

**Problem:** Fork without matching join (or mismatched types).

**See:** [Gateway Balancing Rules](#gateway-balancing-rules) above.

---

## Token Semantics

Understanding token flow is critical for debugging deadlocks.

### Token Basics

**Token:** Conceptual object representing execution flow (like a ball rolling through pipes).

**Rules:**
- Process **starts** with one token at Start Event
- Sequence flows **transfer** tokens
- Tasks **consume** incoming token, **produce** outgoing token
- End events **consume** token (no output)

### Gateway Token Behavior

| Gateway Type | Split Behavior | Join Behavior |
|-------------|----------------|---------------|
| **Exclusive (XOR)** | 1 token in → 1 token out (one path) | 1 token in → 1 token out (pass through) |
| **Parallel (AND)** | 1 token in → N tokens out (all paths) | N tokens in → 1 token out (synchronize) |
| **Inclusive (OR)** | 1 token in → 1-N tokens out (conditional) | 1-N tokens in → 1 token out (sync activated) |
| **Event-Based** | 1 token in → 1 token out (first event) | N/A (no join) |

### Common Deadlock Scenarios

**Scenario 1: Parallel fork without join**
```
Fork(3) → A → Task_Final
          B → Task_Final  ❌ Task receives 3 tokens → executes 3 times
          C → Task_Final
```

**Scenario 2: Mismatched fork/join**
```
Parallel Fork(3) → ... → Exclusive Join  ❌ Join expects 1, gets 3 → 2 stuck
```

**Scenario 3: Inclusive with termination**
```
OR Fork → Path A → End  ❌ Token consumed
        → Path B → OR Join  ❌ Waits forever for Path A
```

---

## Quick Reference

### Decision Tree: Which Gateway?

```
Need to route execution?
│
├─ All paths MUST execute simultaneously?
│  └─ YES → Parallel Gateway (AND)
│
├─ Wait for external event?
│  └─ YES → Event-Based Gateway
│
├─ Exactly ONE path based on condition?
│  └─ YES → Exclusive Gateway (XOR)
│
└─ ONE OR MORE paths based on conditions?
   └─ YES → Inclusive Gateway (OR) - or use Parallel + Exclusive instead
```

### Checklist: Before Using Gateways

**Exclusive (XOR):**
- [ ] Named as question?
- [ ] Default flow defined?
- [ ] Conditions mutually exclusive?
- [ ] Conditions complete (cover all values)?

**Parallel (AND):**
- [ ] Used in pairs (fork + join)?
- [ ] Path counts match (or documented)?
- [ ] NO conditions on outgoing flows?
- [ ] NO multi-merge (missing sync)?

**Inclusive (OR):**
- [ ] Default flow OR always-true condition?
- [ ] NO End events between split/join?
- [ ] Really need OR? (Could use Parallel + Exclusive?)

**Event-Based:**
- [ ] Outgoing flows connect to catch events (not tasks)?
- [ ] Only one event will fire?

### Common XML Mistakes

| Mistake | Detection | Fix |
|---------|-----------|-----|
| Missing `default` on XOR | No `default="Flow_ID"` attribute | Add default attribute |
| Conditions on Parallel out | `<bpmn:conditionExpression>` on AND outgoing | Remove conditions |
| Multi-merge | Element with `<bpmn:incoming>` count > 1 | Insert merge gateway |
| Unbalanced paths | Fork count ≠ Join count | Match counts or document |
| Token termination in OR | `<bpmn:endEvent>` between OR split/join | Remove End, route to join |

---

## Additional Resources

**Related Documentation:**
- `antipatterns-full.md` - Complete anti-pattern catalog (Sections 6-9 cover gateways)
- `multi-merge-prevention.md` - Phase 0 algorithm to prevent multi-merge
- `SKILL-v2.md` - Gateway Selection Decision Tree
- `xml-templates.md` - XML syntax templates for all gateway types

**Back to:** `SKILL-v2.md` Gateway section

---

**Version:** 1.0 (Unified from gateway-rules-and-antipatterns.md + gateway-cheatsheet-ru.md)
**Token savings:** ~1,000 lines from original files
**Date:** 2025-12-10
