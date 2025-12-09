# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is **BPMN Generator Pro** - a specialized skill system for creating and editing BPMN 2.0 process diagrams that are compatible with Camunda Modeler. The system transforms natural language descriptions into professional BPMN XML files and can modify existing .bpmn files.

## Core Architecture

### Skill System Structure

The main skill is located in `bpmn-generator-pro/`:

```
bpmn-generator-pro/
├── SKILL.md                    # Main instruction file - READ THIS FIRST
├── references/
│   ├── xml-templates.md        # XML snippets for all BPMN elements
│   ├── gateway-rules-and-antipatterns.md       # Gateway balancing rules (English)
│   ├── gateway-cheatsheet-ru.md                # Gateway quick reference (Russian)
│   ├── advanced-patterns.md                    # 10 advanced BPMN patterns
│   ├── editing-guide.md                        # How to modify existing .bpmn files
│   ├── intelligent-layout-algorithm-v3.md      # Layout calculation algorithm
│   └── process-validation-report.md            # Validation results for sample processes
└── assets/
    └── example-loan-approval.bpmn              # Reference example
```

### Critical Files to Understand

1. **SKILL.md** - The brain of the system. Contains:
   - Analysis framework (MANDATORY FIRST STEP before generating any BPMN)
   - Naming conventions (Verb+Object for tasks, questions for gateways)
   - Task type selection matrix
   - Gateway selection decision tree
   - 15 anti-patterns to avoid (with examples)
   - Validation checklist
   - XML ID generation rules (consistent prefixes: `Task_`, `Gateway_`, `Flow_`, etc.)

2. **gateway-rules-and-antipatterns.md** & **gateway-cheatsheet-ru.md** - Comprehensive gateway documentation covering:
   - 5 gateway types: Exclusive (XOR), Parallel (AND), Inclusive (OR), Event-Based, Complex
   - Balancing rules (fork must match join)
   - The Multi-Merge anti-pattern (#9 in SKILL.md)
   - Token semantics

3. **xml-templates.md** - Copy-paste XML snippets for every BPMN element

## Key Principles

### 1. PMA Restriction: NO LANES for Single-Organization Processes 🚫

**CRITICAL:** Lanes (swimlanes) are PROHIBITED for processes within a single organization (one company with internal departments).

**Why lanes are harmful:**
- ❌ Increase diagram size (harder to view)
- ❌ Break the happy path (zigzag pattern across lanes)
- ❌ Reduce readability (must jump between lanes)
- ❌ Obscure process logic with organizational complexity

**Instead of lanes:** Use role overlays on tasks:
```xml
<!-- WRONG: Multiple lanes for departments -->
<bpmn:laneSet>
  <bpmn:lane name="Sales">...</bpmn:lane>
  <bpmn:lane name="Finance">...</bpmn:lane>
</bpmn:laneSet>

<!-- CORRECT: Single pool, roles in task attributes -->
<bpmn:userTask name="Review Application"
               camunda:candidateGroups="finance">
</bpmn:userTask>
```

**Three ways to indicate roles:**
1. **Camunda attributes** (preferred): `camunda:candidateGroups="sales"`
2. **Text annotations**: Add annotation "Role: Finance"
3. **Task name prefix**: `[Finance] Review Application`

**Exception:** Lanes ARE acceptable for:
- Cross-organizational collaboration (different companies)
- Message flows between participants
- Explicit user requirement (with warning about readability)

**Golden rule:** "The happy path should be a straight line, not a zigzag."

See SKILL.md "Process Layout Philosophy (PMA Approach)" for full details.

### 2. BPMN 2.0 XML Structure

All .bpmn files have this structure:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<bpmn:definitions xmlns:bpmn="http://www.omg.org/spec/BPMN/20100524/MODEL"
                   xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI"
                   xmlns:dc="http://www.omg.org/spec/DD/20100524/DC"
                   xmlns:di="http://www.omg.org/spec/DD/20100524/DI"
                   ...>
  <bpmn:process id="Process_..." isExecutable="true">
    <!-- Process elements: tasks, gateways, events, flows -->
  </bpmn:process>

  <bpmndi:BPMNDiagram>
    <bpmndi:BPMNPlane bpmnElement="Process_...">
      <!-- Visual layout: waypoints, bounds, labels -->
    </bpmndi:BPMNPlane>
  </bpmndi:BPMNDiagram>
</bpmn:definitions>
```

**Two parallel structures**: Process logic (`<bpmn:process>`) and Visual layout (`<bpmndi:BPMNDiagram>`). BOTH must be maintained.

### 2. Critical Anti-Patterns (MUST AVOID)

From SKILL.md sections that MUST be enforced:

- **#9 Multi-Merge Anti-Pattern**: Tasks should have ONLY ONE incoming sequence flow. Use explicit merge gateway instead.
  - ❌ WRONG: Multiple flows from different XOR gateways → Task
  - ✅ CORRECT: Multiple flows → Merge Gateway → Single flow → Task
  - **MANDATORY ALGORITHM**: SKILL.md Section 9d contains step-by-step algorithm to prevent Multi-Merge
    - For EVERY sequence flow creation: Check if target already has incoming flow
    - If yes: Insert merge gateway FIRST, then redirect all flows through it
    - Exception: End events can have multiple incoming flows

- **#12 Implicit Gateway**: Tasks should have ONLY ONE outgoing sequence flow. Decisions must use explicit gateway.
  - ❌ WRONG: Task with conditional flows as outgoing
  - ✅ CORRECT: Task → Gateway → Conditional flows

- **Unbalanced Gateways**: Every fork gateway must have matching join gateway
  - Parallel fork → Parallel join (same number of paths)
  - Exclusive fork → Exclusive merge (for clarity)
  - Token synchronization is critical

### 3. Gateway Rules (Golden Rules)

From `gateway-cheatsheet-ru.md`:

**Exclusive Gateway (XOR)**:
- Exactly ONE path activates
- MUST have default flow (to prevent deadlock)
- Other flows have conditions

**Parallel Gateway (AND)**:
- ALL paths activate simultaneously
- Fork and join MUST be balanced
- No conditions on outgoing flows (splits unconditionally)

**Inclusive Gateway (OR)**:
- ONE OR MORE paths activate
- MUST NOT terminate tokens between split and join
- Requires careful token management

### 4. Workflow for Creating/Editing BPMN

**Creating new process:**
1. Analyze using SKILL.md Analysis Framework (section at top)
2. Map actors to lanes, actions to tasks, decisions to gateways
3. Generate XML using templates from `xml-templates.md`
4. Calculate layout using algorithm from `intelligent-layout-algorithm-v3.md`
5. Validate against checklist in SKILL.md
6. Save .bpmn file
7. Open in Camunda Modeler: `open -a "Camunda Modeler" "filename.bpmn"`

**Editing existing process:**
1. Read the .bpmn file
2. Parse both `<bpmn:process>` and `<bpmndi:BPMNDiagram>` sections
3. Identify element by ID (e.g., `Task_Payment`)
4. Make changes to process logic
5. Update visual layout coordinates if needed (preserve existing when possible)
6. Validate changes against anti-patterns
7. Save and open in Camunda Modeler

## Validation Checklist

Before considering any BPMN file complete, verify (from SKILL.md):

**Structural:**
- [ ] Has exactly ONE start event
- [ ] Has at least ONE end event (can have multiple for different outcomes)
- [ ] All elements are connected (no orphaned tasks)
- [ ] All gateways are balanced (fork → join)

**Gateway-Specific:**
- [ ] No Multi-Merge (tasks have single incoming flow)
- [ ] No Implicit Gateway (tasks have single outgoing flow)
- [ ] Exclusive gateways have default flow
- [ ] Parallel gateways synchronize all forked paths

**Naming:**
- [ ] Tasks: Verb + Object (e.g., "Review Application")
- [ ] Gateways: Questions (e.g., "Approved?")
- [ ] End events: Meaningful outcomes (e.g., "Request Rejected", not "End")

**Camunda Compatibility:**
- [ ] Uses correct XML namespaces (bpmn, bpmndi, dc, di, camunda)
- [ ] IDs are unique and use consistent prefixes
- [ ] Visual layout has proper waypoints and bounds

## Camunda Modeler Integration

After creating or editing any .bpmn file, automatically execute:

```bash
open -a "Camunda Modeler" "path/to/file.bpmn"
```

This allows visual inspection. Camunda Modeler is located at `/Applications/Camunda Modeler.app`.

## Common Tasks

### Validate a BPMN file
Read the file and check against the validation checklist in SKILL.md. The `process-validation-report.md` shows examples of validation results for reference processes.

### Add error handling to a task
Use boundary error event pattern from `xml-templates.md`:
```xml
<bpmn:boundaryEvent id="Boundary_Error" attachedToRef="Task_Payment">
  <bpmn:outgoing>Flow_Error_Handler</bpmn:outgoing>
  <bpmn:errorEventDefinition errorRef="Error_PaymentFailed" />
</bpmn:boundaryEvent>
```

### Fix Multi-Merge anti-pattern
Insert explicit merge gateway (see SKILL.md section 9):
```xml
<!-- Add merge gateway -->
<bpmn:exclusiveGateway id="Gateway_Merge">
  <bpmn:incoming>Flow_Path1</bpmn:incoming>
  <bpmn:incoming>Flow_Path2</bpmn:incoming>
  <bpmn:outgoing>Flow_ToTask</bpmn:outgoing>
</bpmn:exclusiveGateway>

<!-- Task now has single incoming -->
<bpmn:userTask id="Task_Action">
  <bpmn:incoming>Flow_ToTask</bpmn:incoming>
</bpmn:userTask>
```

### Calculate visual layout
Use the intelligent layout algorithm from `intelligent-layout-algorithm-v3.md`:
- Pool: 1900×650px (standard size)
- Element spacing: 150px horizontal, 100px vertical
- Lane height: Divide pool height by number of lanes
- Place elements left-to-right in flow order
- Align end events vertically for visual clarity

## Reference Documentation

The `references/` folder contains comprehensive guides. When working on specific aspects:

- **Gateway issues?** → Read `gateway-rules-and-antipatterns.md` (English) or `gateway-cheatsheet-ru.md` (Russian)
- **Need XML syntax?** → Use `xml-templates.md`
- **Complex patterns?** → Check `advanced-patterns.md` (compensation, escalation, multi-instance)
- **Editing existing files?** → Review `editing-guide.md`
- **Layout problems?** → Consult `intelligent-layout-algorithm-v3.md`

## Example Files

Sample BPMN files in the root directory demonstrate correct patterns:
- `online-order-processing.bpmn` - Excellent error handling, subprocess usage (scored 100% in validation)
- `loan-approval-v3.bpmn` - Parallel checks, risk-based routing
- `employee-onboarding.bpmn` - Contains implicit gateway anti-pattern (needs fix)
- `insurance-claim-processing.bpmn` - Demonstrates all gateway types

## Language

The skill system supports both English and Russian. User requests can be in either language. Documentation includes both:
- `gateway-rules-and-antipatterns.md` (English)
- `gateway-cheatsheet-ru.md` (Russian)

## Important Notes

- **Always read SKILL.md first** when working with BPMN generation or editing
- **Preserve visual layout** when editing existing files (don't regenerate coordinates unnecessarily)
- **Use consistent ID prefixes** (Task_, Gateway_, Flow_, Event_, etc.)
- **Every change must be validated** against the anti-pattern checklist
- **Think like a Senior Process Architect** - analyze before generating
