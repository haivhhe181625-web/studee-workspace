<!-- Tiếng Việt — FRAMING (đặt vấn đề) Phase 4: engine bài tập tương tác cho các kỹ năng còn lại + quiz.
     Chưa phải spec — chốt §8 trước khi viết spec. Nối tiếp roadmap §Phase 4 + plan-phase3. -->
# Đặt vấn đề: Phase 4 — Engine bài tập tương tác (Nghe/Đọc/Viết/Ngữ pháp/Từ vựng + quiz)

- **Ngày:** 2026-07-17
- **Tác giả:** Dev + Claude (framing)
- **Trạng thái:** Nháp đặt vấn đề — chờ chốt §8 trước khi viết spec
- **Tiền đề:** Phase 1 (Import), Giai đoạn A (quản lý/xuất bản), Phase 2 (Runtime), Phase 3 (Tiến độ), Phase 1.5
  (Program) đã giao. Bài học hiện chỉ có bài tập tương tác **`ipa`** (phát âm) + **`talk`** (hội thoại AI); 5 kỹ năng
  còn lại chỉ có **nội dung tĩnh**, chưa luyện tương tác. `quiz` bị hoãn từ Phase 1 (Q-Quiz).
- **Nguồn:** `docs/roadmap-task-based-learning.md` §Phase 4, khảo sát tái dùng trong `exe-api` (assessment/adaptive/
  llm/course-content).

---

## 1. Điểm xuất phát & Đích đến

### Đã có
| Lớp | Hiện trạng |
|---|---|
| Bài tập tương tác | **ipa** + **talk** (có progress + bắc cầu tiến độ + feed adaptive). Ma trận phủ kỹ năng: chỉ **Phát âm + Nói** có luyện tương tác. |
| Ngân hàng câu hỏi | `assessment_questions` — **đủ itemType** (mcq, mcq_multi, true_false_ng, matching_headings, sentence_completion, labeling, cloze, fill_blank, essay, speaking_prompt, read_aloud…), có `skill/subskill/cefrLevel/targetGoal`, `answerKey` ẩn; stimulus passage/audio. **Chỉ phục vụ trong luồng CAT/MST**, chưa có đường phục vụ cho Bài học. |
| Chấm điểm | `grade.js` `gradeObjective()` **thuần, tách rời CAT** (chấm mọi loại khách quan). AI chấm **viết/nói**: `scoring-engine.adapter` `scoreWriting`/`scoreSpeaking` (rubric, có mock). `llmProxy.chat` (json). |
| Mastery | `learning_events` (source đã reserve **`lesson`/`quiz`**) + `computeMastery` tự nhặt theo subskill. `ingestIpaAttempt` = template feed. |
| Bắc cầu tiến độ | Phase 3 `evaluateStatus` đọc `IpaProgress` để tính Bài completed — pattern chuẩn để nhân bản. |

### Đích đến
Bài học **luyện được đủ 4 kỹ năng**: Nghe/Đọc/Ngữ pháp/Từ vựng qua **quiz khách quan** (tự chấm), Viết/Nói qua
**chấm AI**. Hoàn thành bài tập → tính vào **tiến độ khóa (Phase 3)** và **mastery (adaptive)**. Sau Phase 4,
`ExerciseRef` phủ đủ kỹ năng ⇒ đúng tầm nhìn "học đủ".

---

## 2. Reframe cốt lõi — 5 "kỹ năng" ⇒ 3 engine theo *modality*

Roadmap liệt kê 5 kỹ năng, nhưng về **cơ chế chấm**, chúng gom thành **3 loại engine** (kỹ năng chỉ là **tag** trên
câu hỏi, không phải engine riêng):

| Engine mới | Phủ kỹ năng | Cách chấm | Tái dùng |
|---|---|---|---|
| **`quiz`** (khách quan) | Nghe, Đọc, Ngữ pháp, Từ vựng | `gradeObjective()` tự động | ngân hàng câu hỏi + stimulus (audio/passage) |
| **`writing`** | Viết | AI `scoreWriting(rubric)` | scoring-engine adapter |
| **`speaking`** | Nói (đơn thoại/read-aloud, **khác `talk` hội thoại**) | AI `scoreSpeaking(rubric)` | scoring-engine adapter |

⇒ **Không xây 5 engine kỹ năng**; xây **3 engine modality** tái dùng hạ tầng sẵn có. Nghe/Đọc chỉ là `quiz` gắn
stimulus audio/passage; Ngữ pháp/Từ vựng là `quiz` không stimulus. Đây là điểm giảm phạm vi lớn nhất.

---

## 3. Tái dùng (không xây lại)

| Việc | Tái dùng trực tiếp | Net-new |
|---|---|---|
| Chấm khách quan | **`grade.js gradeObjective`** (thuần) | — |
| Câu hỏi + itemType + stimulus | **`assessment_questions` + `assessment_stimuli`** (đủ loại) | — |
| Chấm viết/nói | **`scoreWriting`/`scoreSpeaking`** + rubric + `llmProxy` | — |
| Mastery | `learning_events` (`quiz`/`lesson` đã có) + `computeMastery` + `ingestIpaAttempt` template | hàm `ingest*` mới cho quiz/writing/speaking |
| Bắc cầu tiến độ | pattern `flatten`+`evaluateStatus`+progress collection (như `IpaProgress`) | collection progress cho engine mới |
| Ref bài tập | pattern `resolveReferences` (ipa→IpaLesson, talk→SCENARIO_IDS) | nhánh mới cho quiz/writing/speaking |
| Program | `programs.js` skills + `targetGoal` trên câu hỏi | — |
| UI runtime | UI exam (assessment) + ipa recorder (mic) làm mẫu | player quiz/writing/speaking trong Bài |

**Phải dựng mới (tối thiểu):** (1) đường **phục vụ item cho Bài** (query/route — chưa có ngoài CAT); (2) **collection
progress** cho engine mới; (3) `EXERCISE_TYPES += quiz/writing/speaking` + nhánh `resolveReferences`; (4) **UI** trong
lesson viewer; (5) **authoring** (curate bộ item cho Bài).

---

## 4. Khoảng trống → khối công việc

```
[Phase 3]  ─►  K1 Engine quiz (khách quan)  ─►  K2 Engine writing (AI)  ─►  K3 Engine speaking (AI)
                       │                              │                          │
                       └──────────────►  K4 Ref + bắc cầu tiến độ + feed mastery (chung 3 engine)
                                                      │
                                                      └─►  K5 UI runtime (exe-web) + K6 Authoring (admin)
```

- **K1 — Engine quiz.** Bộ item (`QuizSet` published, code) trỏ tới `assessment_questions`; route phục vụ (ẩn
  answerKey) + nộp → `gradeObjective` → điểm + progress. Phủ Nghe/Đọc (kèm stimulus)/Ngữ pháp/Từ vựng.
- **K2 — Engine writing.** `WritingTask` (prompt + rubricId) → nộp text → `scoreWriting` → band + progress.
- **K3 — Engine speaking.** `SpeakingTask` (prompt/reference + rubric) → nộp audio → `asr`+`scoreSpeaking` → band +
  progress. **Khác `talk`** (talk = hội thoại tự do, không chấm).
- **K4 — Ref + bắc cầu.** `EXERCISE_TYPES += quiz/writing/speaking`; `resolveReferences` nhánh mới (validate refId theo
  bộ published); `flatten`+`evaluateStatus` AND completion của engine mới với ipa; `ingest*` feed `learning_events`.
- **K5 — UI exe-web.** Player quiz (MCQ/điền/nghe), editor viết, recorder nói trong Bài học; nối vào nút bài tập +
  đồng bộ tiến độ (Phase 3).
- **K6 — Authoring (admin).** Tạo/curate `QuizSet`/`WritingTask`/`SpeakingTask` (CRUD tối thiểu, chọn từ ngân hàng
  có sẵn theo skill/cefr/targetGoal). *(Sinh tự động bằng AI → Phase 5.)*

---

## 5. Bắc cầu 2 hướng (điểm mấu chốt)
Mỗi engine mới phải nối vào **2 hệ đã có**, không tạo hệ đo riêng:
1. **Tiến độ khóa (Phase 3):** progress engine (status ≥ completed) → `evaluateStatus` tính Bài completed (AND với ipa)
   → unlock/% như hiện tại. Ngưỡng completed = điểm ≥ ngưỡng (quiz %/band viết-nói) cấu hình theo bộ.
2. **Mastery (adaptive):** mỗi câu → 1 `learning_event` (`source:'quiz'|'lesson'`, `skill/subskill/itemCefr`,
   `correct` từ gradeObjective hoặc `band` từ AI) → `computeMastery` tự cập nhật. Miễn phí "gắn điểm CEFR A2→B1"
   mà Phase 3 đã hoãn.

---

## 6. Program-aware (TOEIC vs IELTS) — MVP generic, hoãn taxonomy Part
Ngân hàng hiện **CEFR-generic**; tín hiệu kỳ thi chỉ là tag **`targetGoal`** (`general|ielts|toeic|…`); **chưa có**
taxonomy TOEIC Part 1–7 / IELTS task-format.
- **MVP:** bộ item của khóa lọc theo `program`→`targetGoal` + `skill` + `cefr` (khóa TOEIC lấy câu `targetGoal∈
  {toeic,general}`). Đủ để "luyện đúng chất kỳ thi" ở mức cơ bản.
- **Hoãn:** mô hình **Part/Format kỳ thi** (TOEIC Part 1–7, IELTS Writing Task 1/2, Speaking Part 1/2/3) → Phase 4.x
  hoặc 5, khi cần mô phỏng đề thật.

---

## 7. Ngoài phạm vi Phase 4 (chống scope creep)
| Hạng mục | Vì sao hoãn |
|---|---|
| Taxonomy Part/Format kỳ thi (TOEIC Part, IELTS task) | Cần net-new; MVP dùng `targetGoal`+`itemType` |
| Sinh item bằng AI, CMS kéo-thả | → Phase 5 (authoring nâng cao) |
| Mô phỏng **đề thi đầy đủ** trong khóa (full CAT/MST) | Đã có ở assessment; Phase 4 là *luyện theo Bài*, không thi |
| Đề xuất bài tập thích ứng (adaptive khuyến nghị exercise) | → mở rộng adaptive sau |

---

## 8. Câu hỏi mở phải chốt trước khi viết spec (kèm đề xuất)

**Kiến trúc engine**
- **Q-P4.1 (số engine):** 3 engine modality (quiz/writing/speaking) hay 5 theo kỹ năng? *Đề xuất:* **3** — kỹ năng là
  tag `skill` trên câu hỏi, không tách engine.
- **Q-P4.2 (ref quiz trỏ gì):** `refId` quiz → **bộ `QuizSet` published (code)** trỏ danh sách câu hỏi, hay **query
  động** `{skill,cefr,targetGoal}`? *Đề xuất:* **QuizSet published** (ref ổn định, tái lập, validate như ipa). Query
  động để dành (randomize) sau.
- **Q-P4.3 (nguồn item):** tái dùng `assessment_questions` hay dựng ngân hàng item riêng cho Bài? *Đề xuất:* **tái
  dùng** ngân hàng (đã lớn, có type/skill/cefr/targetGoal); `QuizSet` = tập con được curate.
- **Q-P4.4 (progress storage):** 1 collection **`ExerciseProgress` chung** (userId,type,refId,status,score) cho 3
  engine mới, hay mỗi engine 1 collection như IpaProgress? *Đề xuất:* **1 `ExerciseProgress` chung** (gọn; ipa/talk
  giữ nguyên).

**Chấm & hoàn thành**
- **Q-P4.5 (ngưỡng completed):** quiz đạt % nào, viết/nói band nào để tính Bài xong? *Đề xuất:* **cấu hình theo bộ**
  (`passScore`), default quiz ≥ 70%, viết/nói ≥ `targetScore` khóa (hoặc band mặc định); `mastered` khi cao hơn.
- **Q-P4.6 (chi phí AI viết/nói):** gate quota thế nào? *Đề xuất:* qua **`withQuota` + `aiCallLimiter`** như ipa/talk
  (đã có hạ tầng budget); dev dùng **mock** của scoring-engine.
- **Q-P4.7 (speaking vs talk):** giữ cả hai? *Đề xuất:* **có** — `talk` = hội thoại tự do (không chấm, Phase 3 bỏ
  gate), `speaking` = đơn thoại/read-aloud **có chấm** (gate). Rõ ràng khác vai.

**Kỳ thi & nội dung**
- **Q-P4.8 (Part/Format kỳ thi):** làm taxonomy TOEIC Part/IELTS task ngay không? *Đề xuất:* **hoãn** — MVP lọc theo
  `targetGoal`; taxonomy Part để Phase 4.x/5.
- **Q-P4.9 (authoring):** admin CRUD bộ item, hay thêm cột Excel, hay sinh AI? *Đề xuất:* **admin CRUD tối thiểu**
  (chọn câu từ ngân hàng theo skill/cefr/targetGoal → lưu QuizSet/WritingTask/SpeakingTask published). Excel-ref +
  AI-gen → sau.

---

## 9. Đường găng & thứ tự
```
K1 quiz  ──►  K4 ref+bắc cầu (áp cho cả 3)  ──►  K5 UI  ──►  K6 authoring
   └─►  K2 writing  ─┘
   └─►  K3 speaking ─┘   (K2/K3 song song sau khi K1 chốt khung engine+progress)
```
1. **K1 trước** — chốt khung engine + `ExerciseProgress` + bắc cầu (rủi ro cao nhất; K2/K3 bám khung).
2. **K2/K3** — thêm 2 modality AI (tái dùng adapter).
3. **K4** xuyên suốt (ref + progress + mastery).
4. **K5** exe-web, **K6** admin authoring.
5. Deploy: exe-api (K1–K4) → exe-admin (K6) → exe-web (K5).

## 10. Bước tiếp theo
1. Chốt §8 (đặc biệt Q-P4.1/Q-P4.2/Q-P4.4 khung, Q-P4.5 ngưỡng, Q-P4.8 taxonomy).
2. Viết spec `specs/exercise-engines/` (design · data-model · contracts · tasks) khi đã chốt.
3. Triển khai theo §9.
