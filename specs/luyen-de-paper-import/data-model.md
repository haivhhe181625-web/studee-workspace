---
feature: luyen-de-paper-import
status: draft
created: 2026-08-17
role: tech-lead
---

# Data Model — Canonical schema + provenance

## 1. Canonical JSON schema (v1.0)

```jsonc
{
  "schemaVersion": "1.0",
  "paper": {
    "title": "string",
    "exam": "ielts | toeic",            // → Question.targetGoal
    "skill": "reading | listening",
    "variant": "academic | null",
    "blueprintKey": "string | null",    // optional; P1 KHÔNG validate cấu trúc
    "targetBandRange": "string | null",
    "timeLimitSec": "number | null",
    "provenance": {                      // bắt buộc
      "sourceRef": "string",
      "importBatch": "string | null",
      "license": "self-authored | licensed | ai-generated-reviewed | CC-BY-4.0",
      "attribution": "string | null"
    }
  },
  "sections": [
    {
      "order": 1,
      "cefr": "A1..C2",                  // → Question.cefrLevel
      "difficultyTier": "easy | mid | hard | null",
      "stimulus": null,                  // null = câu độc lập (TOEIC P5). Hoặc 1 trong 2 dạng dưới:
      // dạng ref:    { "ref": "<existing stimulusId>" }
      // dạng inline: { "kind": "passage|audio", "body": "... (passage)",
      //               "media": { "audioUrl": "..." } | { "audioFile": "path.mp3", "maxPlays": 1 },
      //               "transcript": "... (listening, select:false)" }
      // LƯU Ý: Stimulus model KHÔNG có field `title` → import bỏ qua. difficultyTier lấy từ section (default 'mid' nếu thiếu).
      "questions": [
        {
          "itemType": "enum ITEM_TYPES",
          "stem": "string",
          "options": ["..."],           // tuỳ itemType (§2)
          "answerKey": "shape theo itemType (§2)",
          "acceptedVariants": ["..."],  // optional; free-text tolerance
          "explanation": "string | null",
          "subskill": "string | null",
          "grammarPoint": "string | null"
        }
      ]
    }
  ]
}
```

## 2. `answerKey` shape theo `itemType` (khớp `grade.js` — validator ép đúng)

| itemType | options | answerKey | acceptedVariants |
|---|---|---|---|
| `mcq` | mảng string | **number** (index) hoặc **string** (value) | — |
| `mcq_multi` | mảng string | **array** (set, all-or-nothing) | — |
| `true_false_ng` | `["true","false","ng"]` | **string** | optional |
| `yes_no_ng` | `["yes","no","ng"]` | **string** | optional |
| `matching_information` / `matching_features` / `matching_sentence_endings` | mảng nhãn | **string** (1 giá trị) | — |
| `matching_headings` | mảng nhãn | **number**(index) hoặc **string** — giống mcq (F2 verified: bank lưu index) | — |
| `matching` / `labeling` | mảng nhãn | **array** thứ tự | — |
| `cloze` / `fill_blank` / `sentence_completion` / `short_answer` / `dictation` | — | **string** | ✅ nên có |

Validator: itemType ∉ enum → reject. answerKey không đúng kiểu (số/string/array) theo bảng → reject.

## 3. Provenance — field mới trên model (surgical, additive)

Thêm vào **`AssessmentQuestion`** và **`AssessmentStimulus`** (cùng shape):

```js
const ProvenanceSchema = new Schema(
  {
    sourceRef: { type: String, default: null },     // danh tính nguồn gốc (đề gốc/batch)
    importBatch: { type: String, default: null },
    attribution: { type: String, default: null },   // ghi công (vd CC BY)
  },
  { _id: false }
);
// ... trong schema chính:
provenance: { type: ProvenanceSchema, default: () => ({}) },
```

- `license` KHÔNG lưu riêng — map sang `contentSource` (enum sẵn có).
- Default `{}` (các field null) → item cũ không ảnh hưởng.

## 4. Mapping canonical → model

| Canonical | Model.field | Ghi chú |
|---|---|---|
| `paper.title/exam/skill/variant/blueprintKey/targetBandRange/timeLimitSec` | `TestPaper.*` | `status='draft'`, `centerId=null` |
| (sinh) | `TestPaper.frozenItemIds` | ID câu theo thứ tự section→question |
| `paper.provenance.license` | `→ contentSource` (Q + Stimulus) | map §5; unknown→reject |
| `paper.provenance.{sourceRef,importBatch,attribution}` | `provenance` (Q + Stimulus) | |
| `sections[].stimulus.kind/title/body/media/transcript` | `AssessmentStimulus.*` | `status='draft'`; `audioFile`→copy→`media.audioUrl` |
| `sections[].cefr` | `Question.cefrLevel` | |
| `sections[].difficultyTier` | `Stimulus.difficultyTier` | |
| `questions[].*` | `AssessmentQuestion.*` | `stimulusId`=stimulus section; `order`=vị trí; `skill`,`targetGoal=exam`,`status='draft'`,`centerId=null` |

## 5. `provenance.license` → `contentSource`

| license (input) | contentSource (model enum) |
|---|---|
| `self-authored` | `self-authored` |
| `licensed`, `CC-BY-4.0` (và biến thể CC-* có ghi công) | `licensed` |
| `ai-generated-reviewed` | `ai-generated-reviewed` |
| thiếu / khác | **reject** (không map được → chặn) |
