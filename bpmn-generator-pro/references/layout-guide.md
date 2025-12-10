# BPMN Layout Guide v3.1

**Purpose:** Intelligent pattern-based layout algorithm for BPMN diagrams with visual layout best practices.

**Replaces:** `intelligent-layout-algorithm-v3.md` + insights from `layout-analysis.md` + `onboarding-layout-analysis.md`

**Based on:** Comprehensive analysis of real-world user corrections and BPMN best practices.

---

## Table of Contents

1. [Applicability](#applicability)
2. [Core Principles](#core-principles)
3. [Spacing Constants](#spacing-constants)
4. [Layout Algorithm](#layout-algorithm)
5. [Real-World Validation](#real-world-validation)
6. [Quick Reference](#quick-reference)
7. [Common Mistakes](#common-mistakes)

---

## Applicability

### When to Use This Algorithm

**✅ Recommended for:**
- Multi-organization processes (different companies/systems with separate pools)
- Lane-based single-org processes (when explicitly requested for legacy compatibility)
- Complex processes where visual separation by role is beneficial
- Editing existing lane-based processes (preserve structure)

**❌ Not Recommended for:**
- **New single-organization processes** → Use role overlay approach instead (see SKILL-v2.md "PMA Approach")
- Simple linear processes → Overhead of lane calculations not justified

### PMA Restriction Note

According to the PMA (Process Modeling Approach) restriction:
- **Lanes should be avoided** for single-organization processes (one company with internal departments)
- **Instead, use role overlays:**
  - Camunda attributes: `camunda:candidateGroups="sales"`
  - Task name prefix: `[Finance] Review Application`
  - Step + Role format: `2.5.1 [Finance] Review Application`

**Exceptions where lanes are OK:**
1. Multi-organization collaboration (different companies)
2. Cross-system integration (independent systems with message flows)
3. User explicitly requests lane-based visualization
4. Editing existing lane-based process

---

## Core Principles

Based on analysis of real-world user corrections:

### 1. Horizontal Compactness is Critical
**Width > Height:** Horizontal compactness is MORE important than vertical compactness.

**User Preference:** Pool width ~1,500-1,900px (NOT 2,500px+)

### 2. Progressive Spacing
**Context-Aware Gaps:** Tight at process start, generous at decision points and end.

**Discovery from user corrections:**
- Standard gap: **50-55px** (NOT 180px!)
- Tight gap: **40px** for simple task chains
- Wider gap: **80px** when starting new sequence

### 3. Content-Driven Dimensions
**Lane Heights:** Based on actual vertical complexity (number of parallel paths), not formula-only.

**User approach:** Minimum viable height, not over-engineered formulas.

### 4. Adaptive Positioning
**Elements adjust** based on process position and complexity.

### 5. End Event Alignment ⭐ CRITICAL
**All end events share same coordinates** (X and Y) to create visual "finish line".

**User requirement:** "End events should be on one horizontal line to visually see where each branch ends."

---

## Spacing Constants

### Horizontal Spacing (Progressive)

```javascript
const HORIZONTAL = {
  // Early Process (first 60% of elements)
  AFTER_START: 52,           // Start → First task
  STANDARD_GAP: 55,          // Default gap between elements
  TASK_CHAIN: 40,            // Simple task → task (sequential)
  NEW_SEQUENCE: 80,          // Major sequence breaks (after subprocess, gateway merge)

  // Late Process (last 40% of elements)
  STANDARD_GAP_LATE: 85,     // Wider for complex decisions
  TASK_CHAIN_LATE: 70,       // More breathing room
  FINAL_STRETCH: 82,         // Task → End event
};
```

**Key Discovery:** User's horizontal spacing = **40-80px**, NOT 180px!

### Vertical Spacing

```javascript
const VERTICAL = {
  // Path Separation (within lane)
  MINOR_PATH: 60,            // Success vs rejection paths
  MAJOR_PATH: 120,           // Parallel branches

  // Lane Heights (Content-Driven)
  SINGLE_LEVEL: 200,         // 1 vertical path
  TWO_LEVELS: 250,           // 2 vertical paths (main + alt)
  THREE_LEVELS: 400,         // 3 vertical paths (upper/main/lower)

  // Formula for 3+ levels
  BASE_HEIGHT: 260,
  INCREMENT_PER_LEVEL: 70,   // Each additional level adds 70px
};
```

**Formula for lane height:**
```javascript
lane_height = BASE_HEIGHT + (vertical_levels - 1) * INCREMENT_PER_LEVEL

Examples:
- 1 level: 200px (fixed minimum)
- 2 levels: 250px (fixed)
- 3 levels: 400px = 260 + (3-1)*70
- 4 levels: 470px = 260 + (4-1)*70
```

### Pool Margins

```javascript
const POOL_MARGIN = {
  LEFT: 120,                 // Left edge
  RIGHT: 50,                 // Right edge (minimal!)
  TOP: 80,                   // Top edge
  BOTTOM: 0,                 // Lanes flush to bottom
};
```

### Element Dimensions (Standard)

```javascript
const ELEMENT_SIZE = {
  TASK: { width: 100, height: 80 },
  GATEWAY: { width: 50, height: 50 },
  EVENT: { width: 36, height: 36 },
  SUBPROCESS: { width: 100, height: 80 },  // Collapsed
  SUBPROCESS_EXPANDED: { width: 350, height: 200 },
};
```

---

## Layout Algorithm

### Phase 1: Process Analysis

**Build flow graph** from start to end events.

**Identify:**
- Logical sequences (chains without branching)
- Decision points (gateways with multiple outgoing)
- Parallel sections (parallel fork → join)
- Exception paths (boundary events, error flows)

**Assign flow levels:**
- Main path = level 0
- Alternative paths = level 1
- Parallel paths = level 2+

### Phase 2: Lane Dimension Calculation

**For EACH lane:**

1. **Count distinct flow levels** (vertical paths that need separation)
2. **Calculate height** using formula:
   ```javascript
   if (levels === 1) height = 200;
   else if (levels === 2) height = 250;
   else height = 260 + (levels - 1) * 70;
   ```
3. **Position lanes** adjacently (no gaps between lanes)

**Example:**
```
Lane_HR: 3 vertical paths → height = 400px, y_start = 80
Lane_IT: 1 vertical path → height = 200px, y_start = 480
Lane_Manager: 2 vertical paths → height = 250px, y_start = 680
```

### Phase 3: Horizontal Layout (X Positioning)

**Build level map** (depth from start):

```javascript
function calculateX(element, levelMap, processPosition) {
  // Determine if early (< 60%) or late (≥ 60%) in process
  const isLateProcess = processPosition >= 0.6;

  // Select gap size based on context
  let gap;
  if (element.type === 'startEvent') {
    gap = 52;  // AFTER_START
  } else if (isSimpleTaskChain(element)) {
    gap = isLateProcess ? 70 : 40;  // TASK_CHAIN
  } else if (isNewSequence(element)) {
    gap = 80;  // NEW_SEQUENCE
  } else {
    gap = isLateProcess ? 85 : 55;  // STANDARD_GAP
  }

  return previous_x + previous_width + gap;
}
```

**Special handling:**
- **End events:** Calculate rightmost position, align all end events to same X
- **Backward flows:** Allowed if space-saving (use waypoints to route cleanly)

### Phase 4: Vertical Layout (Y Positioning)

**Calculate center Y for each flow level within lane:**

```javascript
function calculateY(element, flowLevel, laneHeight, laneStart) {
  if (laneHeight === 200) {
    // Single level: center at lane_start + 90
    return laneStart + 90 - (element.height / 2);
  } else if (laneHeight === 250) {
    // Two levels
    if (flowLevel === 0) {
      return laneStart + 70 - (element.height / 2);  // Main path
    } else {
      return laneStart + 190 - (element.height / 2); // Alt path
    }
  } else {
    // Three+ levels: distribute evenly
    const spacing = laneHeight / (numLevels + 1);
    return laneStart + spacing * (flowLevel + 1) - (element.height / 2);
  }
}
```

**⭐ CRITICAL: Align All End Events**

```javascript
// Calculate average Y of all end events
const endEventYs = endEvents.map(e => e.y);
const alignedY = Math.round(average(endEventYs));

// Set all end events to same Y
endEvents.forEach(e => e.y = alignedY);

// Set all end events to same X (rightmost + margin)
const maxX = Math.max(...endEvents.map(e => e.x));
endEvents.forEach(e => e.x = maxX);
```

**Result:** Visual "finish line" where all outcomes are clearly visible.

### Phase 5: Waypoint Calculation

**For each sequence flow:**

1. **Identify source and target positions**
   ```javascript
   source_exit = { x: source.x + source.width, y: source.y + source.height/2 };
   target_entry = { x: target.x, y: target.y + target.height/2 };
   ```

2. **Determine waypoint count:**
   - **Same lane, same Y:** Direct line (no waypoints)
   - **Same lane, different Y:** L-shape (2 waypoints)
   - **Cross lane:** Horizontal → Vertical (3 waypoints)

3. **Calculate waypoint positions:**
   ```javascript
   // L-shape example (same lane, different Y)
   waypoints = [
     { x: source_exit.x, y: source_exit.y },          // Exit source
     { x: source_exit.x + gap/2, y: source_exit.y },  // Horizontal segment
     { x: source_exit.x + gap/2, y: target_entry.y }, // Vertical segment
     { x: target_entry.x, y: target_entry.y }         // Enter target
   ];
   ```

4. **Minimize waypoint count** (prefer 2-3 waypoints, avoid 4+)

### Phase 6: Collision Detection & Prevention

**Check all element pairs** for overlap:

```javascript
function hasCollision(elem1, elem2, clearance = 20) {
  return !(
    elem1.x + elem1.width + clearance < elem2.x ||
    elem2.x + elem2.width + clearance < elem1.x ||
    elem1.y + elem1.height + clearance < elem2.y ||
    elem2.y + elem2.height + clearance < elem1.y
  );
}
```

**If collision detected:**
1. Adjust Y position (within same lane) by minimum amount
2. If cross-lane, adjust X position
3. Re-validate after adjustments

**Minimum clearance:** 20px between elements

### Phase 7: Pool Dimensions

**Calculate final pool size:**

```javascript
pool_width = rightmost_element_x + POOL_MARGIN.RIGHT;
pool_width = Math.min(pool_width, 2000);  // Max 2000px

pool_height = sum(lane_heights) + POOL_MARGIN.TOP;
```

**Expected results:**
- Pool width: 1,500-1,900px (25% more compact than old approach)
- Pool height: 600-900px (depends on lane count and complexity)

---

## Real-World Validation

### Case Study 1: Online Order Processing

**User corrections revealed:**

#### Pool Dimensions
- My algorithm: 2,530px × 710px
- User's version: **1,900px × 650px**
- **Insight:** User prioritizes horizontal compactness (25% width reduction!)

#### Horizontal Spacing
| Segment | Algorithm Gap | User's Gap | Pattern |
|---------|---------------|------------|---------|
| Start → Validate | 64px | **52px** | Tight |
| Standard tasks | 65-90px | **50-55px** | Tight |
| Simple chain | 90px | **40px** | Very tight |
| New sequence | 90px | **80px** | Moderate |

**Key Discovery:** User's spacing = **40-80px**, NOT 180px!

#### Vertical Positioning
- **Two distinct Y-levels** within lanes:
  - Main path: y = lane_start + 90
  - Alt path: y = lane_start + 150
  - **Vertical offset:** 60px between paths

### Case Study 2: Employee Onboarding

**User corrections revealed:**

#### Content-Driven Lane Heights
| Lane | Vertical Levels | User's Height | Formula |
|------|----------------|---------------|---------|
| HR | 3 levels | **400px** | 260 + (3-1)*70 = 400 |
| IT | 1 level | **200px** | Fixed minimum |
| Manager | 2 levels | **290px** | Fixed (for 2 levels use 250px normally, but user used 290) |

**Key Discovery:** Lane heights are **content-driven**, not fixed.

#### Horizontal Compactness
- My algorithm: 1,750px
- User's version: **1,560px**
- **11% reduction** in width

**Consistent pattern:** User shifts all elements left by ~10px for tighter horizontal spacing.

### Combined Insights

From both case studies:

1. **Horizontal spacing: 40-80px** (NOT 180px)
2. **Lane heights: 200-400px** (content-driven formula)
3. **End event alignment:** Same X and Y (visual finish line)
4. **Pool width: 1,500-1,900px** (NOT 2,500px+)
5. **Vertical separation: 60-120px** between paths

---

## Quick Reference

### Decision Tree: Element Positioning

```
Position element X:
│
├─ Is it Start Event? → x = pool_left + POOL_MARGIN.LEFT
│
├─ Is it End Event? → x = rightmost_element_x + END_EVENT_MARGIN
│                      y = average_Y_of_all_end_events (aligned!)
│
├─ Is it in simple task chain? → gap = 40px (early) or 70px (late)
│
├─ Is it starting new sequence? → gap = 80px
│
└─ Default → gap = 55px (early) or 85px (late)
```

### Spacing Quick Reference

| Situation | Gap Size | Notes |
|-----------|----------|-------|
| Start → First task | 52px | Fixed |
| Simple task → task | 40px (early), 70px (late) | Sequential work |
| Standard gap | 55px (early), 85px (late) | Most elements |
| New sequence | 80px | After subprocess, major break |
| Task → End event | 82px | Final stretch |

### Lane Height Quick Reference

| Vertical Levels | Height | Formula |
|----------------|--------|---------|
| 1 level | 200px | Fixed minimum |
| 2 levels | 250px | Fixed |
| 3 levels | 400px | 260 + (3-1)*70 |
| 4 levels | 470px | 260 + (4-1)*70 |
| N levels (N≥3) | 260 + (N-1)*70 | General formula |

### Expected Results

For typical process (20-30 elements):
- **Pool width:** 1,500-1,900px ✅
- **Pool height:** 600-900px ✅
- **Horizontal gaps:** 40-80px ✅
- **Lane heights:** 200-400px (content-driven) ✅
- **No overlaps:** Guaranteed via collision detection ✅
- **End events aligned:** Same (X, Y) coordinates ✅

---

## Common Mistakes

### Mistake 1: Using 180px Horizontal Gap ❌

**Wrong:**
```javascript
gap = 180;  // Way too wide!
```

**Correct:**
```javascript
gap = isSimpleChain ? 40 : (isNewSequence ? 80 : 55);  // Context-aware
```

**Impact:** 180px gaps create diagrams 2-3x wider than necessary.

### Mistake 2: Fixed Lane Heights Regardless of Content ❌

**Wrong:**
```javascript
lane_height = 300;  // Always 300px
```

**Correct:**
```javascript
if (levels === 1) height = 200;
else if (levels === 2) height = 250;
else height = 260 + (levels - 1) * 70;
```

**Impact:** Wastes vertical space or causes overlaps.

### Mistake 3: Not Aligning End Events ❌

**Wrong:**
```javascript
// End events scattered at different X and Y positions
End_Success: { x: 1200, y: 150 }
End_Rejected: { x: 1050, y: 280 }
End_Cancelled: { x: 1150, y: 350 }
```

**Correct:**
```javascript
// All end events aligned at same X and Y
const alignedX = 1200;
const alignedY = 220;  // Average Y

End_Success: { x: alignedX, y: alignedY }
End_Rejected: { x: alignedX, y: alignedY }
End_Cancelled: { x: alignedX, y: alignedY }
```

**Impact:** User can't easily see all possible outcomes.

### Mistake 4: Over-Engineering Pool Size ❌

**Wrong:**
```javascript
pool_width = 2500;  // Unnecessarily wide
pool_height = 1000; // Unnecessarily tall
```

**Correct:**
```javascript
pool_width = rightmost_element_x + 50;  // Tight right margin
pool_width = Math.min(pool_width, 2000);  // Max cap
pool_height = sum(lane_heights) + 80;  // Just enough
```

**Impact:** Diagram doesn't fit on screen, requires scrolling.

### Mistake 5: Too Many Waypoints ❌

**Wrong:**
```javascript
// 5 waypoints for simple flow
waypoints = [source, wp1, wp2, wp3, wp4, target];
```

**Correct:**
```javascript
// 2-3 waypoints maximum
waypoints = [source, midpoint, target];  // L-shape
```

**Impact:** Cluttered diagram, harder to follow flow.

### Mistake 6: Ignoring Collision Detection ❌

**Wrong:**
```javascript
// Place elements without checking overlaps
```

**Correct:**
```javascript
// Always check and adjust
if (hasCollision(elem1, elem2)) {
  adjustPosition(elem2, minClearance = 20);
}
```

**Impact:** Overlapping elements in Camunda Modeler.

---

## Additional Resources

**Related Documentation:**
- `SKILL-v2.md` - Quick layout rules in Visual Layout section
- `visual-best-practices.md` - Collision prevention techniques, label positioning, boundary events
- `xml-templates.md` - XML syntax for BPMNDiagram and BPMNShape elements

**Back to:** `SKILL-v2.md` Visual Layout section

---

**Version:** 3.1 (Unified from intelligent-layout-algorithm-v3.md + layout-analysis.md + onboarding-layout-analysis.md)
**Token savings:** ~880 lines from original 3 files (1,550 → 670)
**Date:** 2025-12-10
