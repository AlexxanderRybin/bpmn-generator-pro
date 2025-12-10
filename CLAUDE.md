# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is **BPMN Generator Pro** - a specialized skill system for creating and editing BPMN 2.0 process diagrams that are compatible with Camunda Modeler. The system transforms natural language descriptions into professional BPMN XML files and can modify existing .bpmn files.

## Core Architecture

### Skill System Structure

The main skill is located in `bpmn-generator-pro/`:

```
bpmn-generator-pro/
├── SKILL-v2.md                                 # ⭐ Main instruction v2.2 (529 lines, optimized with tables)
├── references/
│   ├── xml-templates.md                        # XML snippets for all BPMN elements
│   ├── gateway-complete-guide.md               # ⭐ Unified gateway guide (all 5 types, bilingual)
│   ├── layout-guide.md                         # ⭐ Unified layout algorithm v3.1
│   ├── multi-merge-prevention.md               # ⭐ Phase 0 detailed algorithm (MANDATORY)
│   ├── antipatterns-full.md                    # ⭐ Complete 15+ anti-pattern catalog
│   ├── workflow-guide.md                       # ⭐ Step-by-step creation/editing workflows
│   ├── examples-guide.md                       # ⭐ 10+ structural patterns with examples
│   ├── advanced-patterns.md                    # Complex patterns (compensation, escalation)
│   ├── editing-guide.md                        # How to modify existing .bpmn files
│   ├── visual-best-practices.md                # Collision prevention techniques
│   └── process-validation-report.md            # Validation results for sample processes
└── examples/                                    # Sample BPMN files demonstrating patterns
    ├── example-loan-approval.bpmn              # Reference example
    ├── employee-onboarding.bpmn                # Contains implicit gateway anti-pattern
    ├── insurance-claim-processing.bpmn         # Demonstrates all gateway types
    ├── loan-approval-v3.bpmn                   # Parallel checks, risk-based routing
    └── online-order-processing.bpmn            # Excellent error handling (100% validation)
```

### Critical Files to Understand

1. **SKILL-v2.md** ⭐ (OPTIMIZED v2.2) - The brain of the system. Contains:
   - **Size:** 529 lines (72% reduction from original 1,902 lines)
   - **Content:** Core rules and quick references with structured tables and prioritization
   - Analysis framework (MANDATORY FIRST STEP before generating any BPMN)
   - Naming conventions (Verb+Object for tasks, questions for gateways)
   - Task type selection matrix
   - Gateway selection decision tree
   - Top 5 critical anti-patterns (brief)
   - Validation checklist
   - Links to detailed reference files
   - XML ID generation rules (consistent prefixes: `Task_`, `Gateway_`, `Flow_`, etc.)

2. **multi-merge-prevention.md** ⭐ (NEW) - Comprehensive Phase 0 algorithm:
   - Why Multi-Merge is the most common error
   - Step-by-step tabular analysis method
   - Per-flow algorithm for incremental creation
   - Real-world examples and detection rules
   - Golden rule: Only merge gateways and end events can have multiple incoming flows

3. **antipatterns-full.md** ⭐ (NEW) - Complete anti-pattern catalog:
   - All 15+ anti-patterns with detailed explanations
   - XML examples of WRONG vs CORRECT approaches
   - Impact analysis (deadlocks, execution errors, maintenance issues)
   - Quick detection signals and fixes
   - Based on research analyzing thousands of BPMN models

4. **examples-guide.md** ⭐ (NEW) - 10 structural pattern examples:
   - Simple approval, multi-level approval, parallel processing
   - Timeout/escalation, loops, subprocesses
   - Inclusive gateway, error handling, multi-instance, event-driven
   - Complete XML examples with use cases
   - Pattern selection guide

5. **workflow-guide.md** ⭐ (NEW) - Step-by-step workflows:
   - Workflow 1: Creating new BPMN process (5 phases)
   - Workflow 2: Editing existing BPMN (4 phases)
   - Workflow 3: Iterative refinement cycle
   - Workflow 4: From requirements document (with step numbers)
   - Common pitfalls and how to avoid them

6. **gateway-complete-guide.md** ⭐ (NEW - UNIFIED) - Comprehensive gateway documentation:
   - **Replaces:** gateway-rules-and-antipatterns.md + gateway-cheatsheet-ru.md
   - **Size:** 672 lines (61% reduction from 1,710 lines total)
   - All 5 gateway types: Exclusive (XOR), Parallel (AND), Inclusive (OR), Event-Based, Complex
   - Balancing rules (fork must match join)
   - Token semantics and deadlock prevention
   - Universal anti-patterns for gateways
   - Bilingual support (English primary, Russian terms where helpful)
   - Quick reference tables and decision trees

7. **xml-templates.md** - Copy-paste XML snippets for every BPMN element

8. **layout-guide.md** ⭐ (NEW - UNIFIED) - Visual layout calculation:
   - **Replaces:** intelligent-layout-algorithm-v3.md + layout-analysis.md + onboarding-layout-analysis.md
   - **Size:** 566 lines (63% reduction from 1,550 lines total)
   - Progressive spacing (40-80px horizontal, NOT 180px)
   - Content-driven lane heights (200-400px formula)
   - End event alignment (same X and Y coordinate - visual finish line)
   - Collision detection and prevention
   - Real-world validation (2 case studies from user corrections)

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

See SKILL-v2.md "Process Layout Philosophy (PMA Approach)" for full details.

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

From SKILL-v2.md sections that MUST be enforced:

- **#9 Multi-Merge Anti-Pattern**: Tasks should have ONLY ONE incoming sequence flow. Use explicit merge gateway instead.
  - ❌ WRONG: Multiple flows from different XOR gateways → Task
  - ✅ CORRECT: Multiple flows → Merge Gateway → Single flow → Task
  - **MANDATORY PHASE 0**: Complete tabular analysis BEFORE creating XML (see `multi-merge-prevention.md`)
    - Create table of ALL elements with incoming flow counts
    - ANY element (except End Events) with incoming > 1 → Add merge gateway
    - Only merge gateways and end events can have multiple incoming flows
  - **MANDATORY ALGORITHM**: For EVERY sequence flow creation:
    - Check if target already has incoming flow
    - If yes: Insert merge gateway FIRST, then redirect all flows through it
    - Exception: End events can have multiple incoming flows
  - **FULL DETAILS**: See `references/multi-merge-prevention.md` and `antipatterns-full.md` Section 9

- **#12 Implicit Gateway**: Tasks should have ONLY ONE outgoing sequence flow. Decisions must use explicit gateway.
  - ❌ WRONG: Task with conditional flows as outgoing
  - ✅ CORRECT: Task → Gateway → Conditional flows

- **Unbalanced Gateways**: Every fork gateway must have matching join gateway
  - Parallel fork → Parallel join (same number of paths)
  - Exclusive fork → Exclusive merge (for clarity)
  - Token synchronization is critical

### 3. Gateway Rules (Golden Rules)

From `gateway-complete-guide.md`:

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

**FULL WORKFLOWS:** See `references/workflow-guide.md` for complete step-by-step instructions.

**Creating new process (5 phases):**
1. **Phase 0 (MANDATORY):** Complete tabular Multi-Merge pre-check analysis
   - List ALL elements with incoming flow counts
   - Identify elements with multiple incoming (except End Events)
   - Add merge gateways BEFORE creating XML
   - See `multi-merge-prevention.md` for algorithm
2. **Phase 1:** Analyze using SKILL-v2.md Analysis Framework
   - Identify boundaries, actors, happy path, decision logic
3. **Phase 2:** Process design
   - Apply PMA restriction (no lanes for single-org)
   - Apply naming conventions
   - Select task/gateway types
4. **Phase 3:** Generate XML
   - Use templates from `xml-templates.md`
   - Calculate layout using `layout-guide.md`
   - Apply Phase 0 table (includes merge gateways)
5. **Phase 4:** Validate against checklist in SKILL-v2.md
6. **Phase 5:** Save and open in Camunda Modeler: `open -a "Camunda Modeler" "filename.bpmn"`

**Editing existing process (4 phases):**
1. **Phase 1:** Read and parse the .bpmn file
   - Identify existing elements, IDs, layout
   - Map sequence flows
2. **Phase 2:** Plan changes
   - Check if changes create Multi-Merge
   - Check if changes create Implicit Gateway
   - Plan merge gateway insertions if needed
3. **Phase 3:** Apply changes
   - Add/modify/remove elements
   - Update flows (preserve layout when possible)
   - Add merge gateways where needed
4. **Phase 4:** Validate and reopen
   - Validate against anti-patterns
   - Save and open in Camunda Modeler

**See:** `workflow-guide.md` for detailed instructions on common operations:
- Adding tasks, boundary events, changing gateway types
- Adding exception paths, converting to multi-instance
- Adding escalation, converting sequential to parallel

## Validation Checklist

Before considering any BPMN file complete, verify (from SKILL-v2.md):

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
Read the file and check against the validation checklist in SKILL-v2.md. The `process-validation-report.md` shows examples of validation results for reference processes.

### Add error handling to a task
Use boundary error event pattern from `xml-templates.md`:
```xml
<bpmn:boundaryEvent id="Boundary_Error" attachedToRef="Task_Payment">
  <bpmn:outgoing>Flow_Error_Handler</bpmn:outgoing>
  <bpmn:errorEventDefinition errorRef="Error_PaymentFailed" />
</bpmn:boundaryEvent>
```

### Fix Multi-Merge anti-pattern
Insert explicit merge gateway (see `multi-merge-prevention.md`):
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
Use the intelligent layout algorithm from `layout-guide.md`:
- Pool: 1900×650px (standard size)
- Element spacing: 150px horizontal, 100px vertical
- Lane height: Divide pool height by number of lanes
- Place elements left-to-right in flow order
- Align end events vertically for visual clarity

## Reference Documentation

The `references/` folder contains comprehensive guides. **New optimized structure** with specialized files:

### Quick Navigation by Task

**Creating new process:**
1. Start: Read SKILL-v2.md Analysis Framework
2. Phase 0: Use `multi-merge-prevention.md` (tabular analysis)
3. Patterns: Reference `examples-guide.md` (10+ patterns)
4. XML: Use `xml-templates.md` (syntax)
5. Layout: Use `layout-guide.md` ⭐ (unified positioning algorithm)
6. Validate: Check `antipatterns-full.md` (avoid mistakes)
7. Workflow: Follow `workflow-guide.md` (step-by-step)

**Editing existing process:**
1. Workflow: Read `workflow-guide.md` Workflow 2
2. Patterns: Reference `editing-guide.md` (modification patterns)
3. XML: Use `xml-templates.md` (syntax reference)
4. Validate: Check `antipatterns-full.md` (validate changes)

**Gateway problems:**
1. Read `gateway-complete-guide.md` ⭐ (unified guide, bilingual)
2. Multi-Merge: Check `antipatterns-full.md` Section 9 OR `gateway-complete-guide.md` Anti-Patterns
3. Unbalanced: Check `antipatterns-full.md` Section 6d OR `gateway-complete-guide.md` Balancing Rules
4. Token deadlocks: Check `gateway-complete-guide.md` Token Semantics section

**Complex patterns:**
1. Structural: Use `examples-guide.md` (10 common patterns)
2. Advanced: Use `advanced-patterns.md` (compensation, escalation, multi-instance, subprocesses)

**Multi-Merge prevention:**
1. Algorithm: Read `multi-merge-prevention.md`
2. Examples: Check `antipatterns-full.md` Section 9
3. Phase 0: Follow SKILL-v2.md Phase 0 section

### File Organization by Category

**Core Instructions:**
- `SKILL-v2.md` (529 lines) - Optimized main reference v2.2 with tables and prioritization

**Anti-Patterns & Quality:**
- `antipatterns-full.md` ⭐ - Complete 15+ anti-pattern catalog
- `multi-merge-prevention.md` ⭐ - Phase 0 algorithm in detail

**Patterns & Examples:**
- `examples-guide.md` ⭐ - 10 structural patterns with full XML
- `advanced-patterns.md` - Complex patterns (compensation, escalation, etc.)

**Workflows:**
- `workflow-guide.md` ⭐ - Creation/editing workflows (4 workflows)
- `editing-guide.md` - Editing patterns and techniques

**Gateway Documentation:**
- `gateway-complete-guide.md` ⭐ - Unified comprehensive guide (all 5 types, bilingual, 61% reduction)

**Layout & Visual:**
- `layout-guide.md` ⭐ - Unified layout guide (7-phase algorithm, 63% reduction)
- `visual-best-practices.md` - Collision prevention and visual clarity

**Templates & Validation:**
- `xml-templates.md` - XML syntax for all BPMN elements
- `process-validation-report.md` - Example validation results

## Example Files

Sample BPMN files in `bpmn-generator-pro/examples/` demonstrate correct patterns:
- `online-order-processing.bpmn` - Excellent error handling, subprocess usage (scored 100% in validation)
- `loan-approval-v3.bpmn` - Parallel checks, risk-based routing
- `employee-onboarding.bpmn` - Contains implicit gateway anti-pattern (needs fix)
- `insurance-claim-processing.bpmn` - Demonstrates all gateway types
- `example-loan-approval.bpmn` - Reference example with proper structure

All examples are located in `bpmn-generator-pro/examples/` directory.

## Language

The skill system supports both English and Russian. User requests can be in either language. All unified documentation files (`gateway-complete-guide.md`, `layout-guide.md`) include bilingual support with English primary content and Russian terms where helpful.

## Important Notes

- **Always read SKILL-v2.md first** when working with BPMN generation or editing (529 lines, optimized v2.2 with structured tables)
- **Complete Phase 0 Multi-Merge analysis** BEFORE generating any XML (see `multi-merge-prevention.md`)
- **Preserve visual layout** when editing existing files (don't regenerate coordinates unnecessarily)
- **Use consistent ID prefixes** (Task_, Gateway_, Flow_, Event_, etc.)
- **Every change must be validated** against the anti-pattern checklist in `antipatterns-full.md`
- **Follow workflows** in `workflow-guide.md` for step-by-step guidance
- **Think like a Senior Process Architect** - analyze before generating

## Version History

**v2.1 (2025-12-10) - OPTIMIZATION RELEASE:**

**Phase 1 - Main File Restructuring:**
- ✅ **SKILL-v2.md**: Reduced from 1,902 to 531 lines (72% reduction, ~9,000 tokens saved)
- ✅ **New reference files** for detailed content:
  - `multi-merge-prevention.md` - Phase 0 algorithm in detail
  - `examples-guide.md` - 10 structural patterns with full examples
  - `antipatterns-full.md` - Complete 15+ anti-pattern catalog
  - `workflow-guide.md` - Step-by-step creation/editing workflows
- ✅ **Improved modularity**: Core rules in main file, details in specialized references
- ✅ **Better organization**: Quick navigation by task, file organization by category

**Phase 2 - Gateway Documentation Merge:**
- ✅ **gateway-complete-guide.md**: Unified guide from 2 files
  - Combined: gateway-rules-and-antipatterns.md (829 lines) + gateway-cheatsheet-ru.md (881 lines)
  - Result: gateway-complete-guide.md (672 lines)
  - **61% reduction** (1,038 lines saved, ~7,000 tokens)
- ✅ **Bilingual support**: English primary with Russian terms where helpful
- ✅ **Complete coverage**: All 5 gateway types, balancing, token semantics, anti-patterns

**Phase 3 - Layout Documentation Merge:**
- ✅ **layout-guide.md**: Unified guide from 3 files
  - Combined: intelligent-layout-algorithm-v3.md (748 lines) + layout-analysis.md (420 lines) + onboarding-layout-analysis.md (382 lines)
  - Result: layout-guide.md (566 lines)
  - **63% reduction** (984 lines saved, ~7,000 tokens)
- ✅ **Real-world validation**: Integrated insights from user corrections
- ✅ **Complete algorithm**: 7 phases with formulas, examples, quick reference

**Phase 4 - Anthropic Best Practices:**
- ✅ **SKILL-v2.md → v2.2**: Applied Anthropic documentation standards
  - Structured formatting: All code blocks have language tags (xml, text, bash)
  - Systematic prioritization: CRITICAL (🔴⚠️), IMPORTANT (📝📐), ESSENTIAL (⭐)
  - Prose to tables: PMA comparison, complexity guidelines, layout rules, task navigation
  - Removed redundancies: Fixed all outdated file references
  - Improved visuals: Better decision trees, diagrams with consistent formatting
  - **~12% additional reduction** through better structure (531 → 529 lines)
- ✅ **Better scanability**: Tables allow quick lookup without reading full paragraphs
- ✅ **Clear hierarchy**: Visual markers indicate priority levels at a glance
- ✅ **No content loss**: All information preserved with improved accessibility

**Phase 5 - Repository Cleanup:**
- ✅ **Removed deprecated files** (6 files, 5,162 lines):
  - SKILL.md (1,902 lines) - replaced by SKILL-v2.md
  - gateway-rules-and-antipatterns.md (829 lines) - merged into gateway-complete-guide.md
  - gateway-cheatsheet-ru.md (881 lines) - merged into gateway-complete-guide.md
  - intelligent-layout-algorithm-v3.md (692 lines) - merged into layout-guide.md
  - layout-analysis.md (458 lines) - merged into layout-guide.md
  - onboarding-layout-analysis.md (400 lines) - merged into layout-guide.md
- ✅ **Updated all references** in CLAUDE.md to point to new unified files
- ✅ **Clean repository**: Only active, optimized files remain

**Total Optimization (Phases 1-5):**
- ✅ **Token efficiency**: ~52% reduction in total skill size
- ✅ **Line reduction**: 4,573 lines saved (SKILL + Gateways + Layout + Formatting)
- ✅ **Improved structure**: Tables, markers, consistent formatting throughout
- ✅ **Maintained completeness**: All critical information preserved and enhanced

**v2.0 (2025-12-08):**
- Added Camunda Modeler integration
- Added editing support for existing .bpmn files
- Created `editing-guide.md`
