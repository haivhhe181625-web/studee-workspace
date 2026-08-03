# Contract: Template Selection Resolver (Enrollment)

- **Loại:** Nội bộ module (adaptive service → learning-map service)
- **Bên cung cấp (provider):** `modules/learning-map/template-resolver.js`
- **Bên tiêu thụ (consumer):** `modules/adaptive/adaptive.service.js` (ensureActivePath)
- **Trạng thái:** Live (commit 6600767)

---

## Overview

Resolver matches a learner's **placement outcome** (program + CEFR band) to the right **CourseMapTemplate**, so admin-authored templates auto-reach the correct learner cohort. Replaces hardcoded `demo-foundation` (`ensureActivePath` line 76-99 trước).

**Input:** StudentModel (learner's target program + cefr band).
**Output:** 1 CourseMapTemplate or `[]` (fallback to demo-foundation).
**Guarantee:** Existing learner's LearnerPath.segments (pinned templateKey,version) never changes retroactively.

---

## Resolver signature

```javascript
/**
 * Match learner to eligible templates by program & band.
 * 
 * @param {StudentModel} sm — learner's model (target.program, target.cefr)
 * @returns {Promise<CourseMapTemplate|null>} — best-match template or null (fallback to demo)
 * 
 * Eligibility:
 *   1. Template status ∈ {published, undefined/legacy} (not draft/archived)
 *   2. Program: exact match (+2) > universal (+1) > skip
 *   3. Band: cefrRange covers sm.target.cefr:
 *      - null cefrRange = universal (covers all bands)
 *      - [floor, ceiling] = learner band must fall in range
 *      - mismatch = skip (NEVER force learner into wrong band)
 *   4. Entry node unlocked (≥1 node with gatePolicy=NONE or prerequisite=[])
 *   5. Rank: program-exact+band-fit > program-universal+band-fit > fallback
 * 
 * Fallback: demo-foundation (hardcoded key "demo-foundation", latest published)
 *   if no match or no templates exist.
 * 
 * Learner pinning: LearnerPath.segments already ref (templateKey, version) at creation
 *   → resolver only affects NEW path enrollment, never retroactive.
 */
async function resolveTemplatesForLearner(sm: StudentModel): Promise<CourseMapTemplate|null>
```

---

## Matching algorithm

```javascript
async resolveTemplatesForLearner(sm) {
  const learnerProgram = sm.target?.program || null;  // e.g., "ielts" or null
  const learnerBand = sm.target?.cefr || null;        // e.g., "B1" or null

  // Step 1: Load all publishable templates (published or status undefined for backward compat)
  const templates = await CourseMapTemplate.find(
    { status: { $nin: ['draft', 'archived'] } }
  ).lean();

  if (!templates.length) return null;  // Fallback to demo-foundation

  // Step 2: Score each template
  const candidates = templates.map(t => {
    let score = 0;

    // Program matching: exact > universal
    if (t.program === learnerProgram) {
      score += 2;  // Exact match
    } else if (t.program === null) {
      score += 1;  // Universal (any program)
    } else {
      // Program-specific template doesn't match learner program → skip
      return null;
    }

    // Band coverage: null = universal, [floor,ceiling] = must overlap
    if (t.cefrRange?.floor === null && t.cefrRange?.ceiling === null) {
      // null cefrRange = universal (covers all bands)
      score += 10;  // Strong signal
    } else if (learnerBand && isInCefrRange(learnerBand, t.cefrRange)) {
      // Learner band falls in template's range
      score += 10;
    } else if (!learnerBand) {
      // Learner has no band (null) → accept template with null cefrRange
      if (t.cefrRange?.floor === null && t.cefrRange?.ceiling === null) {
        score += 10;
      } else {
        return null;  // Skip band-specific template for unplaced learner
      }
    } else {
      // Learner band ≠ template range → skip (NEVER force mismatched band)
      return null;
    }

    // Entry node check: ≥1 unlocked node
    const hasEntry = t.nodes?.some(n => 
      !n.prerequisites || n.prerequisites.length === 0
    );
    if (!hasEntry) {
      // Template has no entry node (all nodes gated) → skip
      return null;
    }

    return { template: t, score };
  });

  // Step 3: Filter nulls and rank by score desc, then version desc
  const eligible = candidates
    .filter(c => c !== null)
    .sort((a, b) => {
      if (b.score !== a.score) return b.score - a.score;  // Higher score first
      return b.template.version - a.template.version;      // Newer version first
    });

  if (eligible.length === 0) return null;  // Fallback to demo-foundation

  return eligible[0].template;  // Return best match
}

function isInCefrRange(band, range) {
  if (!range || range.floor === null || range.ceiling === null) {
    return true;  // Null range = covers all bands
  }
  
  const cefrOrder = ['A1', 'A2', 'B1', 'B2', 'C1', 'C2'];
  const bandIndex = cefrOrder.indexOf(band);
  const floorIndex = cefrOrder.indexOf(range.floor);
  const ceilingIndex = cefrOrder.indexOf(range.ceiling);

  return bandIndex >= floorIndex && bandIndex <= ceilingIndex;
}
```

---

## Input: StudentModel snapshot

```json
{
  "_id": "64xyz...",
  "userId": "63abc...",
  "target": {
    "program": "ielts",    // Enum PROGRAM_KEYS or null
    "cefr": "B1"           // Enum CEFR_LEVELS or null (unplaced)
  },
  // ... other fields
}
```

| Field | Kiểu | Ghi chú |
|---|---|---|
| `target.program` | String (enum) or null | Learner's enrolled program (ielts|toeic|general|...). Null = no program target. |
| `target.cefr` | String (enum) or null | Learner's band (A1–C2) or null (unplaced, onboarding). |

---

## Output: Selected template

**Success (match found):**
```json
{
  "_id": "64template...",
  "templateKey": "course-ielts-foundation",
  "version": 1,
  "status": "published",
  "cefrRange": { "floor": "A1", "ceiling": "B2" },
  "program": "ielts",
  "nodes": [ ... ],
  "edges": [ ... ]
}
```

**No match (fallback to demo-foundation):**
```json
null  // Caller falls back to demo-foundation lookup
```

---

## Usage in ensureActivePath

```javascript
// In adaptive.service.ensureActivePath(studentModel)
async ensureActivePath(sm) {
  // Load/create LearnerPath for this learner
  
  // Step 1: Try template resolver
  const template = await resolveTemplatesForLearner(sm);
  
  // Step 2: Fallback to demo-foundation if no match
  const selectedTemplate = template || 
    await CourseMapTemplate.findOne({
      templateKey: 'demo-foundation',
      status: { $nin: ['draft', 'archived'] }
    }).lean();
  
  // Step 3: Error if no fallback exists
  if (!selectedTemplate) {
    throw new ApiError(404, 'No enrollment template available');
  }
  
  // Step 4: Build path from selected template (snapshot pin)
  return buildPathFromTemplate(selectedTemplate, sm);
}
```

---

## Scoring rules (detailed)

### Rule 1: Program matching

| Program | Template.program | Score Δ | Reason |
|---|---|---|---|
| "ielts" | "ielts" | +2 | Exact match — learner in ielts program, template for ielts. |
| "ielts" | null | +1 | Universal fallback — template serves any program. |
| "ielts" | "toeic" | skip | Mismatch — program-specific template for different program. Skip. |
| null | "ielts" | skip | Learner unaffiliated — don't force program-specific map. |
| null | null | +1 | Both universal — safe baseline. |

### Rule 2: Band coverage (CEFR range)

| Learner.cefr | Template.cefrRange | Coverage | Reason |
|---|---|---|---|
| "B1" | `{floor:"A1",ceiling:"B2"}` | ✓ cover | B1 ∈ [A1,B2] — learner band fits. |
| "B1" | `{floor:null,ceiling:null}` | ✓ cover | Null range = universal (all bands). |
| "C1" | `{floor:"A1",ceiling:"B2"}` | ✗ skip | C1 ∉ [A1,B2] — learner band too high, skip (NEVER force down). |
| null | `{floor:null,ceiling:null}` | ✓ cover | Unplaced learner + universal template OK. |
| null | `{floor:"A1",ceiling:"B2"}` | ✗ skip | Unplaced learner, band-specific template. Skip. |

### Rule 3: Entry node eligibility

| Nodes | Prerequisites | Entry node? | Reason |
|---|---|---|---|
| `[{nodeKey:"n1",prerequisites:[]}]` | (empty) | ✓ yes | n1 unlocked → learner can start. |
| `[{nodeKey:"n1",prerequisites:["n0"]}]` | (only n1 listed) | ✗ no | n1 gated by n0 (not present) → no entry. |
| `[]` | — | ✗ no | Empty template → no nodes. |

---

## Ranking (tiebreaker)

When multiple templates eligible:

1. **Program score first:** exact (+2) > universal (+1).
2. **Band coverage score:** covered (+10) > universal range (+10) — tie.
3. **Version highest:** newer version preferred (learner gets latest published).

**Example:**
- Template A: program=ielts (+2) + band cover (+10) + v2 → score 12, v2
- Template B: program=null (+1) + universal cefrRange (+10) + v3 → score 11, v3
- **Winner:** Template A (higher score).

---

## Edge cases

### Case 1: Program-specific mismatch band

Learner: `{program:"ielts", cefr:"C1"}`
Template: `{program:"ielts", cefrRange:{floor:"A1",ceiling:"B2"}}`

**Result:** SKIP. Never force learner into ielts map that doesn't cover their band. Fallback to demo-foundation instead.

**Rationale:** Content-Team intent clear — map this cefrRange for this program. If learner band outside, they're not the audience. Fallback is safer.

### Case 2: Learner no program, template universal

Learner: `{program:null, cefr:"B1"}`
Template: `{program:null, cefrRange:{floor:null,ceiling:null}}`

**Result:** MATCH. Both universal → safe.

### Case 3: Two templates same program, different versions

Learner: `{program:"ielts", cefr:"B1"}`
Template-v1: `{program:"ielts", cefrRange:{floor:"A1",ceiling:"B2"}, version:1}`
Template-v2: `{program:"ielts", cefrRange:{floor:"A1",ceiling:"B2"}, version:2}`

**Result:** Template-v2 (newer). Learner gets latest improvements.

### Case 4: No templates, fallback

Learner: any
Templates: none or all archived

**Result:** Fallback to demo-foundation (or 404 if demo missing). Existing learers (pinned to old template) unaffected.

### Case 5: Demo-foundation missing

Learner: `{program:"unknown", cefr:null}`
Resolved: nothing
Fallback: demo-foundation (not found)

**Result:** 404 NOT_FOUND. Onboarding blocked (expected — dev env issue).

---

## Testing scenarios

| Scenario | Input | Expected | AC |
|---|---|---|---|
| Exact match | {program:"ielts", cefr:"B1"} + ielts v1 B1 template | ielts v1 | AC-30 |
| Program mismatch | {program:"ielts", cefr:"B1"} + toeic template | fallback demo | AC-30 |
| Band mismatch | {program:"ielts", cefr:"C1"} + ielts A1-B2 template | fallback demo | AC-30 |
| Universal template | any learner + universal template | universal template | AC-30 |
| No templates | any | fallback demo | AC-31 |
| No fallback | any + no demo | 404 | AC-31 |
| Existing learner pinned | learner has LearnerPath.segments (templateKey, version) | segments unchanged (no retroactive) | AC-32 |

---

## Backward compatibility

**Legacy docs** (status field undefined before commit 690df00):
- Treated as published (`$nin:['draft','archived']` includes undefined).
- Still eligible for matching.
- No migration required.

**Learner migration** (existing learners on demo-foundation):
- Pin templateKey=`demo-foundation`, version=(their snapshot).
- Resolver changes affect NEW paths only.
- Existing LearnerPath.segments NEVER updated (per-learner immutability).

