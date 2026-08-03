# Contract: Band Credit Distribution (Internal)

- **Loại:** Nội bộ module (checkpoint.service ↔ adaptive.service)
- **Bên cung cấp (provider):** `adaptive.service` (applyCheckpointResult, StudentPath snapshot, band-point logic)
- **Bên tiêu thụ (consumer):** `checkpoint.service` (submitCheckpoint → per-skill credit decision)
- **Trạng thái:** Đã triển khai

---

## Overview

After checkpoint grading completes (overall score + per-skill subscores computed), `submitCheckpoint` calls `applyCheckpointResult` for each skill where sub-score ≥ threshold. This contract defines:

1. **Budget resolution** — fetch frozen skillBudget/skillTarget from StudentPath snapshot.
2. **Credit logic** — increment bandPoint, clamp at target, track in creditedBoundaries.
3. **History logging** — append HistoryEntry with sourceGroup='checkpoint', skillCefr snapshot, changedSkills.

---

## Internal Service Call

```javascript
// checkpoint.service.submitCheckpoint line ~159
for (const skill of passingSkills) {  // skills where sub-score ≥ threshold
  if (!creditedBoundaries.includes(boundaryKey)) {
    await adaptive.applyCheckpointResult(
      userId,
      skill,
      true,     // passed=true (always true in this path)
      attemptId,
      { budgetBySkill, targetBySkill }  // optional: pre-resolved budget
    );
  }
}
```

### Function Signature

```javascript
/**
 * @param {ObjectId} userId
 * @param {String} skill (e.g., 'listening', 'reading')
 * @param {Boolean} passed (true = credit band)
 * @param {ObjectId} attemptId (CheckpointAttempt ID)
 * @param {Object} [opts]
 *   @param {Map|Object} budgetBySkill { skill → Number (bandPoint increment) }
 *   @param {Map|Object} targetBySkill { skill → Number (bandPoint max) }
 * @returns {Promise<{ skillCefr, changedSkills }>}
 */
async function applyCheckpointResult(userId, skill, passed, attemptId, opts = {})
```

---

## Behavior

### Input Resolution

```javascript
const { budgetBySkill = {}, targetBySkill = {} } = opts;

// If budget not pre-resolved, fetch from StudentPath
if (!Object.keys(budgetBySkill).length) {
  const { budgetBySkill: b, targetBySkill: t } = await resolveBandBudget(
    userId,
    courseSlug,
    boundaryKey
  );
  budgetBySkill = b;
  targetBySkill = t;
}
```

**resolveBandBudget implementation** (checkpoint.service.js:42):
```javascript
async function resolveBandBudget(userId, slug, boundaryKey) {
  if (boundaryKey === 'course') {
    return { budgetBySkill: {}, targetBySkill: {} };  // course-level: NO credit
  }
  const sp = await getActiveStudentPath(userId);
  const course = sp?.courses?.find(c => c.slug === slug);
  const phase = course?.phases?.find(p => p.key === boundaryKey);
  return {
    budgetBySkill: phase?.skillBudget || {},
    targetBySkill: course?.skillTarget || {}
  };
}
```

**Semantics:**
- Course-level checkpoint (`boundaryKey='course'`) → empty budget (NO per-skill credit).
- Missing snapshot / pre-budget snapshot → empty maps → applyCheckpointResult no-op (safe).

### Credit Logic

```javascript
const model = await StudentModel.findOne({ userId });
if (!model || !model.skill) return null;  // No model: 409 NEEDS_ASSESSMENT (caller handle)

const ability = model.skill.get ? model.skill.get(skill) : model.skill[skill];
if (!ability) return null;  // Skill not in model

const prior = {
  cefr: ability.cefr,
  bandPoint: ability.bandPoint || cefrToBandPoint(ability.cefr)
};

if (passed) {
  // Idempotent: skip if already credited
  const creditedBoundaries = ability.creditedBoundaries || [];
  if (creditedBoundaries.includes(boundaryKey)) {
    // Already credited → no-op (return prior state)
    return { skillCefr: prior.cefr, changedSkills: [] };
  }

  // Credit band
  const budget = mapGet(budgetBySkill, skill) || 0;
  const target = mapGet(targetBySkill, skill) || 6.0;  // default C2 if no target

  const newBandPoint = Math.min(prior.bandPoint + budget, target);
  const newCefr = bandPointToCefr(newBandPoint);  // floor

  // Update model
  ability.bandPoint = newBandPoint;
  ability.cefr = newCefr;
  ability.creditedBoundaries.push(boundaryKey);

  await model.save();

  // Log history
  const skillCefr = {};
  const entries = model.skill instanceof Map ? model.skill.entries() : Object.entries(model.skill);
  for (const [s, a] of entries) {
    if (a?.cefr) skillCefr[s] = a.cefr;
  }

  const priorCefr = prior.cefr;
  const changedSkills = newCefr !== priorCefr ? [skill] : [];

  await LearnerProfile.findOneAndUpdate(
    { userId },
    {
      $push: {
        history: {
          takenAt: new Date(),
          sourceGroup: 'checkpoint',
          bandSource: 'checkpoint',
          resultId: attemptId,
          overallCefr: null,  // Per-skill only (not overall)
          skillCefr,
          changedSkills,
          actor: { kind: 'system' },
          notifiedLearner: false
        }
      }
    },
    { new: true }
  );

  return { skillCefr: newCefr, changedSkills };
} else {
  // passed=false → no-op (no penalty, no change)
  return { skillCefr: prior.cefr, changedSkills: [] };
}
```

### Output

```javascript
{
  skillCefr: String,        // New CEFR display (A1–C2)
  changedSkills: [String]   // [] or [skill] if CEFR changed
}
```

---

## Data State

### Before Credit

```javascript
StudentModel.skill['listening'] = {
  theta: 2.1,
  se: 0.3,
  cefr: 'B1',
  bandPoint: 3.2,
  creditedBoundaries: ['phase-0']
}
```

### After Credit (pass, budget=0.5)

```javascript
StudentModel.skill['listening'] = {
  theta: 2.1,
  se: 0.3,
  cefr: 'B1',                      // floor(3.7) = floor(3.x) = B1
  bandPoint: 3.7,                  // 3.2 + 0.5 = 3.7
  creditedBoundaries: ['phase-0', 'phase-1']  // Added
}

// LearnerProfile.history[0] (newest)
{
  takenAt: 2026-08-03T10:30:00Z,
  sourceGroup: 'checkpoint',
  bandSource: 'checkpoint',
  resultId: ObjectId('checkpoint-attempt-123'),
  skillCefr: { listening: 'B1', reading: 'A2', … },
  changedSkills: [],                   // B1→B1 (no change visible)
  actor: { kind: 'system' },
  notifiedLearner: false
}
```

**Note:** bandPoint changed (3.2→3.7) but CEFR floor unchanged (both = B1). History entry captures this via skillCefr snapshot.

---

## Idempotency Guarantee

**Retake same phase:**
1. First attempt: sub-score ≥ threshold → credit band (boundaryKey='phase-1' pushed to creditedBoundaries).
2. Second attempt: sub-score ≥ threshold again → check creditedBoundaries.includes('phase-1') → **SKIP** → return (no update).

**Result:** band raised exactly once per phase/skill, dù retry 100 lần.

---

## Edge Cases

### Case 1: Missing StudentModel

```javascript
// applyCheckpointResult called before assessment
const model = await StudentModel.findOne({ userId });
if (!model) {
  throw new ApiError(HTTP.CONFLICT, 'Học viên chưa làm đánh giá', CODES.NEEDS_ASSESSMENT);
}
```

Caller (`submitCheckpoint`) catches → 409 response.

### Case 2: Skill not in StudentModel

```javascript
// StudentModel exists but skill was never assessed
const ability = model.skill.get('writing');  // undefined
if (!ability) return null;
```

**Silent no-op** (skill not in model → can't credit). Caller doesn't flag error (acceptable: skill outside course scope).

### Case 3: Course-level checkpoint (no per-skill credit)

```javascript
if (boundaryKey === 'course') {
  return { budgetBySkill: {}, targetBySkill: {} };
}
```

**resolveBandBudget** returns empty maps → applyCheckpointResult sees budget=0 → **no credit** (only gates completion).

### Case 4: Budget exhausted (reached target)

```javascript
const newBandPoint = Math.min(3.2 + 0.5, 4.0);  // clamped at target 4.0
// = 3.7 (not over)
```

If learner already at target:
```javascript
const newBandPoint = Math.min(3.9 + 0.5, 4.0);  // clamped
// = 4.0 (max out)

// HistoryEntry shows changedSkills=[] (no new change if already at target)
```

---

## Event Emission

```javascript
// After successful credit
emit('checkpoint:passed', {
  userId,
  skill,
  attemptId,
  priorCefr,
  newCefr,
  raisedBands: newCefr !== priorCefr ? 1 : 0
});

emit('band:changed', {
  userId,
  source: 'checkpoint',
  skill,
  delta: newBandPoint - priorBandPoint
});
```

**(Consumers: future — notifications, XP/badges, etc.)**

---

## Comparison to Assessment Band-up

| Aspect | Assessment | Checkpoint |
|---|---|---|
| **Trigger** | Result.seedFromResult (after CAT) | applyCheckpointResult (after submit) |
| **Per-skill credit?** | NO (whole student recalibrated) | YES (per-skill independent) |
| **Budget?** | NO (theta-based → CEFR direct) | YES (frozen per-chặng budget) |
| **Idempotent?** | Implicit (Result per assessment) | Explicit (creditedBoundaries list) |
| **History entry** | sourceGroup='assessment' | sourceGroup='checkpoint' |

---

## Contract Validation

- [ ] `budgetBySkill[skill]` ≥ 0 (can be 0 if no budget allocated).
- [ ] `targetBySkill[skill]` ≥ current bandPoint (no rollback).
- [ ] `newBandPoint` ≤ 6.0 (hardcoded ceiling).
- [ ] `creditedBoundaries` NEVER duplicated (push unique).
- [ ] `changedSkills` array contains only if CEFR floor changed (not bandPoint alone).

---

## Rollback / Error Recovery

**If `model.save()` fails** (e.g., concurrent edit):
```javascript
try {
  await model.save();
} catch (e) {
  logger.error('applyCheckpointResult save failed', { userId, skill, error: e });
  throw new ApiError(HTTP.CONFLICT, 'Tính toán band thất bại (retry)', CODES.CONCURRENT_EDIT);
}
```

Caller (`submitCheckpoint`) returns 409 → learner retries entire checkpoint submit (latest model.save will succeed).

---

## Future Extensions

1. **Bonus credit** — if overall checkpoint score > 0.9, bonus 0.1 bandPoint per skill (additive before clamp).
2. **Skill-specific threshold** — different threshold per skill (vs single threshold now).
3. **Weighted allocation** — budget ∝ skill difficulty, not equal per-phase.
4. **Negative credit (penalt­y)** — future design; currently `passed=false` → no-op.
