---
feature: course-flashcard-vocab-mcq
role: tech-lead
status: draft
date: 2026-08-08
depends_on: spec.md
---

# Design: Kiểm tra vocab MCQ trong khóa học

> Chỉ viết phần **non-obvious**. Grading/SRS/MCQ đã có — design tập trung ranh giới tái dùng + thay đổi data model + luồng ghi tín hiệu.

## 1. Nguyên tắc

- **DRY:** tái dùng `flashcard.service.buildQuizOptions` + `applyReviewEvent` + `FlashcardSrs` + `scheduler`. KHÔNG viết grading/SRS mới.
- **Ranh giới:** course thao tác trên **system deck** (không `userId`); standalone giữ nguyên (owned-deck). Điểm khác duy nhất là **cách nạp thẻ** (system vs owned) + **ghi thêm `LearningEvent`**.
- **KISS:** không ngưỡng %, không test-out, không cross-surface (v-next).

## 2. Thay đổi data model

### 2.1 `learning-event.model.js`
- `SOURCES`: thêm `'flashcard'`.
- `SUBSKILLS`: thêm `'vocabulary'`.
- Giữ nguyên shape; `correct` (bool), `subskill:'vocabulary'`, `itemCefr` (xem §5), `countsTowardMastery:true`.

### 2.2 `knowledge-graph.js` — KHÔNG đụng (quyết định isolation)
- **Không** thêm `vocabulary` vào `GRAPH`/`PRIMARY`/`subskillToSkillMap`.
- Lý do (verify `adaptive-path-builder.js:68-74`): `skillsMeetingMastery` **AND mọi subskill** của 1 skill qua `subskillToSkillMap`. Nếu map `vocabulary→use_of_english`, thì `use_of_english` sẽ đòi thêm vocabulary ≥ 0.8 mới "đủ mastery" → chặn checkpoint-invite use_of_english khi vocab yếu/mới → **đổi hành vi adaptive-path + phá test (nghịch AC-8)**.
- Isolation: `vocabulary` chỉ là subskill trong `SUBSKILLS` enum. `computeMastery` **vẫn lưu** `mastery.vocabulary` (group theo `LearningEvent.subskill`, không cần graph). `skillsMeetingMastery` **bỏ qua** vocabulary (`if (!sk) continue`, `:72`) → **0 ảnh hưởng gating skill nào**. Vocab theo dõi riêng, không nhiễu.

### 2.3 `course-content.model.js`
- **Không đổi schema** `EXERCISE_TYPES` (giữ `'flashcard'`); chỉ **đổi ngữ nghĩa runtime** của exercise `flashcard`. `refId` vẫn = system deck id (giữ nguyên `course-content.references`).

### 2.4 `FlashcardSrs`
- Không đổi. Course tạo row trên `(userId, system cardId)` — schema đã cho phép (không ràng buộc deck phải có userId).

## 3. Luồng (course MCQ)

```
GET  /courses/:slug/.../exercise/flashcard/quiz      → dựng MCQ từ system deck (buildQuizOptions)
POST /courses/:slug/.../exercise/flashcard/answer    → chấm 1 thẻ:
      applyReviewEvent(userId, systemCard, {mode:'quiz', correct, ...})   // SRS + review-log (tái dùng)
      + LearningEvent.create({source:'flashcard', subskill:'vocabulary', correct, itemCefr})
POST /courses/:slug/.../exercise/flashcard/complete   → xong lượt → recordLesson(completed) + score
```

- Endpoint đặt trong `quiz.service`/`quiz.routes` course-side (nơi `getFlashcardCards`/`completeFlashcard` đang sống) — thay 2 hàm cũ, **không** tạo module mới (surgical).
- `applyReviewEvent` nhận `card` object → truyền thẳng system card (lean) — nó không check ownership (ownership ở tầng gọi). ✅ tái dùng an toàn.
- Mastery recompute: gọi `adaptive.ingest*`/`recomputeMastery(userId)` sau khi ghi LearningEvent (giống `quiz.service` sau submit, `:139`).

## 4. Đổi nghĩa exercise `flashcard` — tương thích

- `getFlashcardCards` (đọc thẻ để flip) → **giữ** cho lesson `vocab_flashcard` (bài học). Exercise `flashcard` (bài tập) chuyển sang path MCQ mới.
- `completeFlashcard` (reviewedCardIds ⊇ deck) → **thay** bằng complete theo lượt MCQ. Cần rà seed/FE gọi cũ (câu hỏi mở §6 spec).

## 5. Quyết định câu hỏi mở (đề xuất — cần sign-off)

| # | Câu hỏi | Đề xuất | Lý do |
|---|---|---|---|
| OQ-1 | `itemCefr` | Lấy **CEFR của phase** chứa lesson (`Phase.cefrFrom`); fallback `deck.level`; fallback `null` | Phase là nơi CEFR sống chính thức (spec M4); deck.level tùy chọn |
| OQ-2 | XP/gamification | **Có cộng XP** qua `applyReviewEvent` mặc định | DRY, học viên có động lực; nếu owner muốn tách course XP → cờ sau |
| OQ-3 | Migrate flip-complete | Giữ `getFlashcardCards` cho lesson; chỉ FE course đổi runner exercise sang MCQ; rà seed exercise `flashcard` | Tránh phá dữ liệu khóa hiện có |

## 6. Không cần

- Không contract cross-service (thuần trong `api`).
- Không data-model.md riêng (đủ gọn ở đây).
- Không ADR (không phải quyết định kiến trúc bền — chỉ mở rộng enum + tái dùng).

## 7. Test surface

- Unit: course MCQ build (system deck, distractor co giãn, min 2 thẻ); ghi LearningEvent đúng source/subskill; mastery `vocabulary` cập nhật, band khác bất biến; enum không phá `computeMastery`.
- Integration: answer → FlashcardSrs upsert (không nhân đôi khi học lại); complete → lesson completed + score; ownership boundary (system deck không cần owned).
