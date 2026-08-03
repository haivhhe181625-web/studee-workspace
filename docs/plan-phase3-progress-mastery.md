<!-- Tiếng Việt — dưới docs/, tài liệu FRAMING (đặt vấn đề) cho Phase 3 (Tiến độ & Mastery). Chưa phải spec chi
     tiết — chốt câu hỏi mở §9 trước khi viết spec. Nối tiếp plan-course-to-web.md (Phase 1→2 đã xong). -->
# Đặt vấn đề: Phase 3 — Tiến độ, Mastery & Unlock theo điểm

- **Ngày:** 2026-07-17
- **Tác giả:** Dev + Claude (framing)
- **Trạng thái:** Nháp đặt vấn đề — chờ chốt §9 trước khi viết spec
- **Tiền đề:** Phase 1 (Import), Phase 2 (Runtime — Giai đoạn B+C) đã giao. Learner đã **xem & học** được khóa
  `published`, nhưng **mở toàn bộ, không đo lường** (Q-C1 cố ý hoãn phần đo tới đây).
- **Nguồn:** `docs/roadmap-task-based-learning.md` §Phase 3, `docs/plan-course-to-web.md`, khảo sát tiền lệ đo lường
  trong `exe-api` (ipa/adaptive/roadmap/user-streak).

---

## 1. Điểm xuất phát & Đích đến

### Đã có (sau Phase 2)

| Lớp | Hiện trạng |
|---|---|
| **exe-api** | `/api/courses` đọc khóa published (catalog + cả cây). `course_structures` là **content global**, KHÔNG có tham chiếu user. Resolver `ipa code→_id` đã có. |
| **exe-web** | `/courses`, `/courses/[slug]`, lesson viewer, mở bài tập ipa/talk. **Mở toàn bộ**, không khóa, không % tiến độ. |
| **Đo lường sẵn có** | **ipa**: `ipa_progress` (1 doc/user×lesson: `status[not_started/in_progress/completed/mastered]`, `overallScore`, `masteredStreak`, ngưỡng `masteredThreshold{consecutive,minScore}`). **adaptive**: `learning_events` (append-log) + `StudentModel.mastery` (per subskill); ipa đã tự feed `pronunciation`. **streak**: subdoc trên `User` (`current/longest/lastActiveDate`) **nhưng chưa có nơi ghi**. |
| **talk** | **KHÔNG lưu gì** (chỉ trừ quota lượt). Không có bản ghi phiên/hoàn thành. |
| **roadmap** | `user_roadmaps` có *hình dạng* unlock (`locked/unlocked/in_progress/completed` + timestamp/node + 1-active/user) **nhưng chỉ mở phase đầu**, chưa có engine tiến trình. |

### Đích đến

Learner **học có đo lường**: ghi danh (enroll) một khóa, hệ thống **theo dõi tiến độ** từng Bài/Chuyên đề/Chặng,
**gate mở bài kế theo hoàn thành/điểm**, tổng hợp **"đã tiến A2→B1 tới đâu"**, và (tối thiểu) **streak** hoạt động.
exe-web hiển thị thanh tiến độ, khoá/mở, "học tiếp".

---

## 2. Vấn đề cốt lõi — Bắc cầu tiến độ Bài học ↔ engine bài tập

Đây là mấu chốt của Phase 3 (khác các phase trước): **một Bài học của khóa không tự chấm điểm** — nó gồm lý
thuyết + media + **tham chiếu bài tập `ipa`/`talk`**. Trạng thái "đã học/đạt" của Bài phải **suy ra từ tiến độ của
chính engine bài tập**, không đo lại:

- **ipa** — đã có `ipa_progress` (keyed `userId + IpaLesson._id`), có `status` + `overallScore`. Course chỉ cần
  **đọc** qua `refId`(code)→`_id` (resolver đã có). ⚠️ 1 IpaLesson dùng chung nhiều khóa ⇒ điểm phát âm là **của
  user với bài đó**, dùng chung mọi nơi (hợp lý, không tách theo khóa).
- **talk** — **không có tín hiệu hoàn thành nào**. Phải quyết: coi talk là *luyện tự do không gate*, hay thêm một
  bản ghi tối thiểu "đã luyện scenario X" để tính hoàn thành (talk **chưa có engine chấm điểm**).

⇒ Phase 3 cần một **lớp tiến độ của khóa** (mới) *tổng hợp* các tín hiệu engine, chứ không phải một hệ chấm điểm
mới.

---

## 3. Phân tích khoảng trống → các khối công việc

```
[Phase 2: Runtime]  ─►  K1 Enroll + data model tiến độ  ─►  K2 Ghi nhận hoàn thành Bài (bắc cầu ipa/talk)
   (ĐÃ XONG)                    (api)                              (api)
                                   │                                  │
                                   ├─►  K3 Unlock tuần tự + gate điểm (api)
                                   ├─►  K4 Tổng hợp tiến độ % + mastery Chặng (api)
                                   ├─►  K5 Streak writer (api, tận dụng scaffold)
                                   └─►  K6 exe-web: enroll, thanh tiến độ, khoá/mở, "học tiếp"
```

- **K1 — Enroll + data model (nền tảng).** Tạo collection tiến độ mới (không có gì tái dùng cho *ownership* khóa).
  Endpoint enroll khóa `published`. **Quyết định định danh Bài** (xem §5).
- **K2 — Ghi nhận hoàn thành Bài.** Bắc cầu: đọc `ipa_progress` theo exercise ipa; xử lý talk (§2). Định nghĩa
  "Bài completed" (§9 Q-P3.3).
- **K3 — Unlock.** Khoá/mở Bài–Chặng kế theo hoàn thành (±ngưỡng điểm). Tái dùng *hình dạng* roadmap
  (`locked/unlocked/in_progress/completed`), dựng engine tiến trình (roadmap chưa có).
- **K4 — Tổng hợp.** % hoàn thành per Chuyên đề/Chặng/khóa; ánh xạ ngữ nghĩa `cefrFrom→cefrTo` của Chặng.
- **K5 — Streak.** Hiện thực `touchStreak(userId)` khi có hoạt động học (scaffold User đã có, chỉ thiếu writer).
- **K6 — exe-web.** Nút "Ghi danh", thanh tiến độ, badge khoá/mở, nút "Học tiếp", hiển thị mastery/điểm; sửa
  catalog/detail/lesson viewer đã dựng ở Phase 2.

*(Tuỳ chọn, hoãn)* **K7 — exe-admin xem tiến độ learner** → §8.

---

## 4. Ranh giới & tái dùng

| Việc | Tái dùng được | Phải dựng mới |
|---|---|---|
| Doc tiến độ per user×lesson | *Khuôn* `ipa_progress` (status enum + score + unique compound index) | Collection tiến độ **khóa** (không có ownership sẵn) |
| Unlock tuần tự | *Hình dạng* `user_roadmaps` (locked/unlocked/… + timestamp/node) | Engine tiến trình (roadmap mới mở phase đầu) |
| Điểm phát âm | `ipa_progress.overallScore/status` (đọc qua refId→id) | — (không chấm lại) |
| Hoàn thành talk | — (talk 0 persistence) | Tín hiệu hoàn thành talk (hoặc bỏ gate) |
| Mastery kỹ năng | `learning_events` + `StudentModel.mastery` (ipa đã feed) | % tiến độ **theo Bài/Chặng** (trục khác adaptive) |
| Streak | Subdoc `User.streak` | Writer `touchStreak` |

---

## 5. Quyết định data-model quan trọng (định danh Bài học)

Từ khảo sát: Bài học `key` **chỉ unique trong Chuyên đề**, và mọi node embedded để `_id:false` (không có ObjectId).
⇒ **không thể** khóa tiến độ bằng `courseId + lessonKey`. Phải dùng **path đầy đủ**:

```
progressKey = `${phaseKey}/${moduleKey}/${lessonKey}`   (trong phạm vi 1 courseId)
```

**Hệ quả — xung đột với "sửa khóa" (Giai đoạn A):** Giai đoạn A cho phép sửa **cả cây** khi `draft`/`ready` (ghi đè
`phases`), gồm cả đổi `key`. Khi đã có learner enroll + tiến độ khóa theo path, **đổi key/cấu trúc sẽ làm lệch tiến
độ**. Đây chính là ràng buộc **Q-A3** ("chặn sửa nếu có learner enroll") mà Phase 2 hoãn — Phase 3 làm cho nó **có
thật**, phải chốt (xem Q-P3.2).

---

## 6. Ngoài phạm vi Phase 3 (chống scope creep)

| Hạng mục | Thuộc | Vì sao hoãn |
|---|---|---|
| Engine chấm **talk**, bài tập Nghe/Đọc/Viết/Ngữ pháp/Từ vựng + `quiz` | **Phase 4** | Phase 3 chỉ *tổng hợp* tín hiệu engine đang có (ipa), không xây engine mới |
| Gamification sâu (XP, huy hiệu ngoài ipa, bảng xếp hạng) | sau Phase 3 | Streak tối thiểu là đủ cho "đo lường"; phần còn lại là UX |
| CMS/AI authoring, versioning đầy đủ, host media | **Phase 5** | Không liên quan đo lường |
| Báo cáo tiến độ cho center/giáo viên (đa-tenant) | Phase 3+ | `centerId=null` dùng chung; mở khi có nhu cầu |

---

## 7. Đường găng & thứ tự "triển khai một thể"

```
K1 (enroll + model) ──► K2 (hoàn thành Bài) ──► K3 (unlock) ──► K4 (tổng hợp) ──► K6 (exe-web)
                                   └── K5 (streak) song song, ghép vào K2/K6
```

1. **K1 trước** — chốt data-model + định danh (rủi ro cao nhất, mọi thứ bám vào).
2. **K2** — bắc cầu ipa/talk (quyết định "completed").
3. **K3–K4** — unlock + tổng hợp (bám K2).
4. **K5** — streak, nhỏ, ghép khi tiện.
5. **K6** — exe-web tiêu thụ, sau khi API K1–K4 ký hợp đồng.
6. Deploy: exe-api (K1–K5) → exe-web (K6).

> Rủi ro tích hợp lớn nhất: **định danh Bài (path) + xung đột sửa khóa** (§5) và **định nghĩa hoàn thành + talk**
> (§2). Chốt hai điểm này trước khi code K1.

---

## 8. Câu hỏi mở phải chốt trước khi viết spec (kèm đề xuất)

**Data model & định danh**
- **Q-P3.1 (cấu trúc tiến độ):** 1 collection `course_enrollments` (doc/user×course) **nhúng** map tiến độ per-Bài,
  hay tách `course_progress` riêng? *Đề xuất:* **1 doc/user×course**, nhúng `lessons: { "pKey/mKey/lKey": {status,
  score, completedAt, unlockedAt} }` + rollup Chặng — giống khuôn `ipa_progress`, ít join, đọc nhanh cho UI.
- **Q-P3.2 (khoá cấu trúc khi có learner):** khi khóa `published` đã có enroll, có **chặn sửa cây/đổi key** không?
  *Đề xuất:* **có** — publish xong + có ≥1 enroll ⇒ khóa cấu trúc (chỉ cho sửa metadata an toàn: title/theory/media,
  KHÔNG đổi key/thêm-bớt node). Hiện thực ràng buộc Q-A3 đã hoãn. Sửa lớn ⇒ archive/clone (Phase 5 versioning).

**Định nghĩa hoàn thành**
- **Q-P3.3 ("Bài completed" là gì):** đã xem, hay phải đạt bài tập? *Đề xuất:* Bài `completed` = **đã mở (viewed)**
  **và** mọi exercise `ipa` đạt `status ≥ completed` (theo `ipa_progress`); Bài không có exercise ⇒ chỉ cần viewed.
  Thêm mốc `mastered` khi mọi ipa `mastered`.
- **Q-P3.4 (nguồn điểm ipa):** đọc `ipa_progress` qua `refId`→`_id`, **không chấm lại**, dùng chung mọi khóa?
  *Đề xuất:* **có** — điểm phát âm thuộc về (user, bài phát âm), độc lập khóa.
- **Q-P3.5 (talk):** có tính hoàn thành talk không? *Đề xuất tối thiểu:* talk là **luyện tự do, KHÔNG gate hoàn
  thành** ở Phase 3 (chưa có engine chấm). Nếu muốn đánh dấu "đã luyện": thêm 1 ghi nhận nhẹ (đếm phiên) — nhưng
  **không** dùng để khoá bài kế. (Chấm talk → Phase 4.)

**Unlock & tổng hợp**
- **Q-P3.6 (kiểu gate):** mở bài kế theo **hoàn thành** hay theo **ngưỡng điểm**? *Đề xuất:* MVP **completion-gate**
  (xong bài trước → mở bài kế), ngưỡng điểm ipa lấy sẵn `masteredThreshold`; ngưỡng cấu hình per-khóa để sau. Hay
  **mở toàn bộ + chỉ hiển thị tiến độ** (không khoá)? — cần chốt "khoá thật" hay "chỉ đo".
- **Q-P3.7 (mastery Chặng/CEFR):** tự tính % theo Bài, hay nhét vào adaptive `StudentModel.mastery`? *Đề xuất:* **tự
  tính % hoàn thành per Chặng/khóa** (trục Bài học), KHÔNG trộn vào adaptive (trục subskill). Giữ liên thông adaptive
  riêng (ipa vẫn feed `pronunciation` như hiện tại).
- **Q-P3.8 (streak):** hiện thực writer luôn? *Đề xuất:* **có** — `touchStreak(userId)` gọi khi mở/nộp bài; nhỏ,
  tận dụng scaffold. Có thể tách optional nếu muốn thu gọn Phase 3.

**Ranh giới**
- **Q-P3.9 (assign/enroll):** learner tự ghi danh, hay admin gán? *Đề xuất:* **tự ghi danh** khóa `published` (nút
  trên exe-web); admin-gán/placement để Phase sau.
- **Q-P3.10 (admin xem tiến độ):** exe-admin có màn theo dõi learner không? *Đề xuất:* **hoãn** — Phase 3 tập trung
  learner; báo cáo admin tách sau.

---

## 9. Bước tiếp theo

1. Chốt §8 — đặc biệt **Q-P3.1/Q-P3.2** (data-model + khoá cấu trúc) và **Q-P3.3/Q-P3.5/Q-P3.6** (hoàn thành + gate).
2. Viết spec `specs/course-progress/` theo khuôn (design · data-model · contracts · tasks) khi đã chốt.
3. Triển khai theo §7 (K1→K6), deploy exe-api trước, exe-web sau.
