<!-- Tiếng Việt — hợp đồng api↔web cho Hồ sơ năng lực + Lộ trình thích nghi. Mở rộng router modules/adaptive. -->
# Hợp đồng API — Hồ sơ năng lực & Lộ trình thích nghi

Base: `/api/adaptive` (module `adaptive`, router đã có). Auth mọi route: **`verifyToken → withTenant`** (ownership
ngầm qua `userId` từ JWT — không `verifyPermission`, khớp các route self-service adaptive/assessment). Chưa có
`StudentModel` (chưa làm đánh giá) ⇒ **409** `NEEDS_ASSESSMENT` (mẫu đã có ở `/recommendations`).

Envelope: trả object trực tiếp `{ profile }` / `{ target }` / `{ path }` (không bọc `{ data }`).

---

## M1 — Hồ sơ năng lực

### §A1 — `GET /api/adaptive/profile` — Xem hồ sơ (FR-A2)
Diễn giải `StudentModel` + `resolveTarget` + `Result` (nguồn seed) thành hồ sơ tiếng Việt. Khối A (trình độ + độ tin
cậy + CI + mốc thời gian) và chẩn đoán chủ điểm/khối E lấy **verbatim** từ `Result` đã chấm (không tự tính). **200**:
```jsonc
{
  "profile": {
    "level": "A2",                        // CEFR tổng (Block A)
    "levelLabel": "Sơ trung cấp",         // tên VN (§7.4)
    "levelDescription": "Giao tiếp trong các tình huống quen thuộc.",  // can-do 1 dòng
    "confidence": "medium",               // Result.confidenceLabel (high|medium|low|null) — BR-03
    "confidenceInterval": { "lo":"A2", "hi":"B1" }, // Result.overallCiLo/Hi; null khi ciPlaceholder (BR-04)
    "assessedAt": "…",                    // Result.createdAt (ngày đánh giá gốc — BR-34)
    "updatedAt": "…",                     // StudentModel.recomputedAt (cập nhật gần nhất)
    "bandSource": "assessment",           // nguồn band hiện tại (BR-33)
    "target": { "program":"ielts", "targetScore":6.5, "cefr":"B2", "source":"onboarding" },
    "skills": [                            // ĐỦ 5 kỹ năng; đã đo trước, xếp yếu → mạnh; chưa đo xuống cuối
      { "skill":"listening", "cefr":"A2", "gap":2, "band":"yếu",  "hasData":true,  "ciLo":"A1","ciHi":"B1", "label":"Nghe — cần ưu tiên" },
      { "skill":"reading",   "cefr":"B1", "gap":1, "band":"trung bình", "hasData":true, "ciLo":null,"ciHi":null, "label":"Đọc — quanh trình độ chung" },
      { "skill":"writing",   "cefr":null, "gap":null, "band":"trung bình", "hasData":false, "ciLo":null,"ciHi":null, "label":"Viết — chưa có dữ liệu" }
    ],
    "weakSubskills": [                     // mastery thấp; GIỮ weakSignal (gắn cờ, confirmed trước) — BR-35/38
      { "subskill":"detail", "skill":"listening", "value":0.32, "weakSignal":false, "label":"Nghe chi tiết" },
      { "subskill":"gist",   "skill":"listening", "value":0.20, "weakSignal":true,  "label":"Nghe ý chính" }
    ],
    "weakTopics": [                        // chẩn đoán chủ điểm (Block E) từ Result.topicBreakdown, yếu nhất trước
      { "tag":"past_perfect", "skill":"use_of_english", "correct":1, "total":5, "weakSignal":false, "label":"past perfect" }
    ],
    "weakPhonemes": [                      // deep-link IpaLesson (findLessonsForPhonemes)
      { "phoneme":"æ", "score":48, "lessonId":"…", "route":"/pronunciation/…" }
    ],
    "examConversion": {                    // Khối F — quy đổi CEFR→điểm thi; null khi program='general' (BR-14)
      "program":"ielts", "scale":"IELTS", "unit":"band", "band":"B1",
      "estimate":"4.0–5.0",               // dải, KHÔNG 1 số (BR-43); null khi vượt ngưỡng (BR-46, vd C2+TOEIC)
      "low":4.0, "high":5.0, "widened":false, // widened=true khi mở dải theo CI (BR-45)
      "targetScore":6.5, "gapBands":1,    // khoảng cách tới mục tiêu (bậc CEFR)
      "skillSkew":false,                  // true khi lệch kỹ năng ≥2 bậc (BR-51)
      "note":"Còn khoảng 1 bậc so với mục tiêu.",
      "disclaimer":"Điểm ước tính nội bộ … không phải điểm thi chính thức."  // BR-13
    },
    "hasTarget": true
  }
}
```
- `gap = cefrNum(target.cefr) − cefrNum(skill.cefr)`; `band`: gap≥2 `yếu` · ==1 `trung bình` · ≤0 `mạnh`. Kỹ năng
  `hasData:false` (chưa đo) hiển thị "chưa có dữ liệu", KHÔNG quy về band thấp nhất (BR-06).
- `weakSignal:true` = cỡ mẫu nhỏ → gắn cờ "có dấu hiệu", vẫn hiển thị (KHÔNG ẩn — BR-35/BR-38). `weakTopics.weakSignal`
  khi `total < 4`.
- `examConversion` (Khối F, §3.6): tái dùng `programs.scoreBandForCefr` (bảng §3.6.2 ở `cefr-mapping.js` — nguồn đơn
  BR-42). Một chiều CEFR→dải điểm (BR-44), luôn kèm `disclaimer` (BR-13). `null` khi không có mục tiêu thi
  (`program='general'`). **409** `NEEDS_ASSESSMENT` nếu chưa có model. **401** thiếu token.

### §A2 — `PUT /api/adaptive/target` — Đặt/sửa mục tiêu (FR-A4)
- Body: `{ "program":"ielts", "targetScore":6.5, "cefr":"B2" }` (mọi field optional; thiếu → giữ/suy mặc định).
- Validate `targetScore` qua `programs.validateTargetScore(program, targetScore)` (status **400 BAD_REQUEST** —
  đồng bộ Phase 1.5 import, codebase không dùng 422):
  - `general` + targetScore≠null ⇒ **400** `INVALID_TARGET_SCORE` (NO_SCALE).
  - ngoài [min,max] ⇒ **400** `INVALID_TARGET_SCORE` (OUT_OF_RANGE); sai step ⇒ **400** (BAD_STEP).
  - `program` không hợp lệ ⇒ **400** `INVALID_PROGRAM` (Joi enum chặn trước; service double-check).
- Đổi `program` mà không kèm `targetScore` ⇒ score cũ bị **reset null** (điểm khác thang không so được).
- Lưu `source:'manual'`, `updatedAt`. **200** `{ target }`. **409** `NEEDS_ASSESSMENT`. **401** thiếu token.

---

## M2 — Lộ trình thích nghi (sau/song song Phase 4)

### §B1 — `GET /api/adaptive/path` — Lộ trình đề xuất (M2 **v3**)
Thuần deterministic (KHÔNG AI). Output: **courses → phases → {modules, allModules}** (nested, khớp builder v3).

**Nguyên tắc chọn khóa:** ceiling (max cefrTo) ∈ (band yếu nhất, target] → skip khóa đã qua + vượt đích.
**Chọn chuyên đề:** per khóa, tính `neededSkills` (gap≥1, chưa mastery ≥0.8), áp cho tất cả phase → lọc module.
**IELTS/TOEIC tách riêng** — `needsProgram:true` khi general ⇒ `courses:[]`. **200**:
```jsonc
{
  "path": {
    "target": { "program":"ielts", "cefr":"B2", "targetScore":6.5 },
    "level": "B1",
    "needsProgram": false,
    "skippedMasteredSkills": ["listening"],       // gap≥1 nhưng mastery≥0.8 → BỎ module (không học lại)
    "readyForCheckpoint": ["listening"],          // mời nâng band: gap≥1 + mastery≥0.85 + chưa từ chối (nguồn = /profile)
    "courses": [
      {
        "slug":"ielts-a2-b1", "title":"IELTS A2 → B1 (4.0–5.0)",
        "program":"ielts", "targetScore":5.0,
        "cefrFrom":"A2", "cefrTo":"B1",
        "neededSkills":["listening","speaking"],   // kỹ năng cần ở khóa này (chưa mastery)
        "fitsLevel":true,                          // khóa ĐẦU = điểm vào
        "customLessons":8, "estimatedDays":16,     // custom: chỉ chuyên đề kỹ năng cần
        "totalLessons":20, "fullEstimatedDays":40, // full: học cả khóa
        "phases": [
          {
            "key":"phase-1-nen-tang", "title":"Chặng 1: Nền tảng", "order":1,
            "cefrFrom":"A2", "cefrTo":"A2", "goalNote":"…",
            "modules": [                            // custom: chỉ listening, speaking
              { "skill":"listening", "category":"listening", "moduleKey":"m-listening",
                "moduleTitle":"Luyện Nghe", "lessonCount":3,
                "route":"/learn/ielts-a2-b1?phase=phase-1-nen-tang&module=m-listening" },
              { "skill":"speaking", "category":"speaking", "moduleKey":"m-speaking",
                "moduleTitle":"Luyện Nói", "lessonCount":2, "route":"…" }
            ],
            "allModules": [                         // full: tất cả modules của phase
              { "category":"listening", "moduleKey":"m-listening", "moduleTitle":"Luyện Nghe", "lessonCount":3, "route":"…" },
              { "category":"reading", "moduleKey":"m-reading", "moduleTitle":"Luyện Đọc", "lessonCount":4, "route":"…" },
              { "category":"grammar", "moduleKey":"m-grammar", "moduleTitle":"Ngữ pháp", "lessonCount":2, "route":"…" },
              { "category":"speaking", "moduleKey":"m-speaking", "moduleTitle":"Luyện Nói", "lessonCount":2, "route":"…" }
            ],
            "lessonCount":5, "totalLessons":11, "estimatedDays":10
          }
        ]
      }
    ]
  }
}
```
- **2 chế độ (FE 2 tab):** `modules` = custom (kỹ năng cần, estimate shorter); `allModules` = full (học cả khóa).
- Kỹ năng **yếu** = `gap≥1`. `neededSkills` áp cho mọi phase của khóa.
- Mastery-skip (Q-AP9): kỹ năng gap≥1 nhưng mastery{value≥0.8, !weakSignal} → **bỏ module**, liệt kê ở `skippedMasteredSkills`.
- `readyForCheckpoint`: mời nâng band — ngưỡng chặt hơn (0.85) + trừ kỹ năng đã từ chối. Lấy **cùng nguồn với `/profile`** nên 2 màn hình không mâu thuẫn.
- **409** `NEEDS_ASSESSMENT`. **401**.

## M2 Bổ sung — Band History & Checkpoint

### §B2 — `GET /api/adaptive/band-history` — Lịch sử thay đổi band
Trả append-only log từ `LearnerProfile.history` (mở rộng M3). **409** `NEEDS_ASSESSMENT`. **200**:
```jsonc
{
  "history": [
    {
      "takenAt": "2026-07-23T10:30:00Z",
      "sourceGroup": "assessment",
      "bandSource": "assessment",
      "overallCefr": "B1",
      "skillCefr": { "listening":"A2", "reading":"B1", "writing":null, "speaking":"B1", "use_of_english":"B1" },
      "changedSkills": ["listening"],    // so vs entry trước
      "actor": { "kind":"learner", "id":null }
    },
    {
      "takenAt": "2026-07-20T14:15:00Z",
      "sourceGroup": "checkpoint",
      "bandSource": "checkpoint",
      "overallCefr": "B1",
      "skillCefr": { "listening":"B1", "reading":"B1", … },
      "changedSkills": ["listening"],
      "actor": { "kind":"system", "id":null },
      "reason": null,
      "notifiedLearner": false
    }
  ]
}
```
Phục vụ Block A "Nguồn cập nhật" + biểu đồ tiến bộ + audit.

### §B3 — `POST /api/adaptive/path/replan` — Cập nhật lộ trình (re-plan explicit)

<!-- 2026-08-03: bổ sung replan endpoint -->

Re-plan **explicit** (tách bạch khỏi auto regen): archive `StudentPath` active + tạo snapshot mới từ live path. Được gọi khi:
- `PUT /target` thay đổi mục tiêu → auto replan
- Nút "Cập nhật lộ trình" (FE) → POST này

Body: empty `{}`. **200**:
```jsonc
{
  "path": {
    // structure + mastered overlay giống GET /path (được tính từ snapshot mới)
    "target": { "program":"ielts", "cefr":"B2", "targetScore":6.5 },
    "level": "B1",
    "courses": [ ... ],
    ...
  }
}
```

**Semantic:**
- Archive bản active hiện (isActive=false, archivedAt=now) → chặng/module cũ không bị mất (lịch sử).
- Tạo snapshot mới từ `computeLivePath` (tình trạng hiện tại + StudentModel mới).
- Trả path mới (từ snapshot vừa tạo) — thay thế full lộ trình FE.
- **Không auto** khi band-up → snapshot giữ structure, chỉ `mastered` flag thay (overlay lúc read).

**Lỗi:**
- **409** `NEEDS_ASSESSMENT` — chưa có StudentModel.
- **401** — thiếu token.

### §B5 — `POST /api/adaptive/checkpoint` — Ghi kết quả kiểm tra nâng band
⚠️ **Scope Phase 1:** service implement + unit test. Endpoint tạm **KHÔNG mở** (chờ assessment engine cung cấp kết quả).
Khung `POST /checkpoint` body `{ skill, passed, attemptId }` → `applyCheckpointResult()` giành cho Phase 2B.
**200** `{ skill, cefr(mới), updatedAt }` nếu pass. **400/409** nếu không eligible. **Ghi chú:** fail KHÔNG hạ band.

### Chưa làm (batch sau)
- **Overlay tiến độ** trên courses/phases (`status` + `percent`) đọc `CourseEnrollment`.
- **Content-tagging chủ điểm** (mức mịn skip hơn = bài/chủ điểm, cần track riêng).
- **Bài tập/media** trong khóa + IPA xen chặng.
- **Admin view StudentPath** — lịch sử re-plan (chưa scope).

---

## Quan hệ lộ trình cố định (FR-C)
Không đổi `GET /api/courses` (Phase 2/3) — lộ trình cố định chạy song song, chỉ đọc chung `CourseEnrollment`. Một
Chuyên đề xuất hiện ở **cả** lộ trình cố định lẫn thích nghi; hoàn thành 1 lần tính cho cả hai (Q-AP7, FR-C2).
