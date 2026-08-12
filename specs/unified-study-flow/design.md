# Design: Luồng học hợp nhất v1

- **Ngày:** 2026-08-05 · **Trạng thái:** ✅ ĐÃ DUYỆT (owner clear mặc định 2026-08-05). Sẵn sàng code.
- **Spec:** `./spec.md`. Dựa trên scout BE+FE 2026-08-05 (file:line trong scout report).

## 1. Kiến trúc tổng (giữ nguyên phần học)
```
Study-space (lộ trình) — GIỮ NGUYÊN: ADAPTIVE_PATH_STUDY, checkpoint UI, exercise inline
        ▲ deep-link (mở)           ▲ đọc tiến độ (tick)
        │                          │
   Lịch = "Nhiệm vụ hôm nay" (lớp mỏng)
        ▲ item mang đủ context (courseSlug/phaseKey/moduleKey/kind/boundaryKey)
        │
   generateSchedule ← resolveLessonList (đã có context, chỉ cần NGỪNG bỏ đi)
```

## 2. Data model — enrich item (KEYSTONE, IS-1/IS-2)
`lesson-schedule.model.js` `days[].items[]` mở rộng (giữ field cũ, thêm):
```
{ kind: 'lesson' | 'review' | 'checkpoint',   // mặc định 'lesson' (back-compat)
  courseSlug: String, phaseKey: String, moduleKey: String,  // context deep-link + tick
  // lesson: lessonKey/title/type/durationMin (như cũ)
  // review: refId/type/durationMin (như cũ) + lessonKey? (bài gốc, để tick best-effort)
  // checkpoint: boundaryKey (=phaseKey|'course'), title, durationMin? }
```
- `courseSlug` cần cho `useCourseProgress(slug)` + route. Lấy từ `path.courses[].slug` / CourseStructure slug trong resolver.
- Thêm field không phá idempotency: chỉ đổi `inputHash` (regenerate 1 lần) — chấp nhận.

## 3. Resolver (IS-1/IS-2) — `lesson-list.resolver.js`
- `buildLessonItem`: emit thêm `courseSlug, phaseKey, moduleKey` (resolver **đã** duyệt qua `course/phase/module` — chỉ truyền xuống, hiện đang drop ở `:87-97`).
- **Checkpoint boundary:** khi duyệt hết modules của 1 phase, nếu `phase.checkpoint` tồn tại → emit thêm 1 **marker checkpoint** `{ kind:'checkpoint', boundaryKey: phase.key, courseSlug, phaseKey: phase.key, title: checkpoint.label ?? 'Vượt chặng' }` **ngay sau** lesson cuối của phase. Cuối course: nếu `course.checkpoint` → marker `boundaryKey:'course'`.
- Resolver trả danh sách hỗn hợp lesson + checkpoint marker (giữ thứ tự path).

## 4. Generator (IS-2) — `lesson-schedule.generator.js`
- Band-filter chỉ áp cho `kind==='lesson'` (checkpoint marker không lọc band).
- Chunking: checkpoint marker **độc chiếm 1 "ngày checkpoint"** (như oversized nhưng `kind:'checkpoint'`), đặt đúng vị trí path (sau cụm bài của phase, trước bài phase kế). `durationMin` checkpoint = hằng ước lượng (vd 30′) — không nhồi chung ngày học.
- Review day (buildReviewDay) giữ nguyên + gắn `courseSlug/phaseKey/moduleKey/lessonKey` cho từng exercise item (từ lesson gốc) để deep-link/tick best-effort.
- Summary/warnings không đổi (checkpoint không tính oversized/heavy).

## 5. Deep-link contract (IS-3, FE) — bảng map item → URL
| kind | URL |
|---|---|
| lesson | `ROUTES.ADAPTIVE_PATH_STUDY(courseSlug) + "?phase=" + phaseKey + "&module=" + moduleKey` |
| review (mỗi exercise) | như lesson (mở module chứa bài tập) — best-effort; talk → fallback `/talk` hoặc disable |
| checkpoint | `ADAPTIVE_PATH_STUDY(courseSlug) + "?phase=" + phaseKey + "&module=" + <module cuối phase> + "&checkpoint=1"` |

- **Giới hạn (D5):** mở ở mức **module** (study-space không nhận lessonKey). Chấp nhận v1.
- Checkpoint cần 1 `moduleKey` để land — dùng module cuối của phase (resolver gắn sẵn vào marker).

## 6. Tick tiến độ (IS-4, FE) — đọc `useCourseProgress(courseSlug)`
- Bài học: `done = progress.lessons["{phaseKey}/{moduleKey}/{lessonKey}"]?.status ∈ {completed, mastered}`.
- Checkpoint: `done = boundaryKey ∈ progress.passedCheckpoints`.
- **Khóa (visual):** nếu study-space đang lock (phase sau checkpoint chưa qua) → item hiện "khóa". Suy từ progress (lesson status `locked`) — `useCourseProgress` đã trả `status:'locked'`.
- Buổi ôn: **không tick chính xác** (D3) — chỉ hiển thị, không đọc done per-exercise. (Tùy: 1 nút "đã ôn" cục bộ, v1 có thể bỏ.)
- Nhiều course trong 1 lịch → gọi `useCourseProgress` theo `courseSlug` của item (gom theo slug; thường 1 lộ trình 1 course chính).

## 7. "Nhiệm vụ hôm nay" (IS-5, FE)
- `/schedule` đã có khối "Kế hoạch hôm nay" — nâng cấp: item bấm được (mục 5) + trạng thái xong/khóa (mục 6). Không tạo route mới.
- (Tùy chọn nhỏ) thêm entry ở dashboard trỏ tới `/schedule` "hôm nay" — để sau nếu cần.

## 8. Kiểm thử (map AC → test)
| AC | Loại | Test |
|---|---|---|
| AC-1 | unit(BE) | resolver/generator emit context đúng cho item học/ôn |
| AC-2 | unit(BE) | phase có/không checkpoint → có/không marker đúng vị trí + boundaryKey |
| AC-3 | unit(FE) | map item→URL đúng 3 kind (lesson/review/checkpoint) |
| AC-4 | unit(FE) | tick done theo progress giả lập (lesson status + passedCheckpoints); khóa khi locked |
| AC-5 | component(FE) | "hôm nay" render item bấm được + trạng thái |

## 9. Rủi ro / lưu ý
- **Không đụng `buildProgress`/study-space** (đang phục vụ học thật) — chỉ ĐỌC.
- Talk không có completion → item ôn talk: disable tick, deep-link best-effort.
- Deep-link mức module: nếu owner cần mức lesson → cần thêm param `?lesson=` ở study-space (ngoài scope v1).
- Checkpoint "ngày riêng" chèn vào chuỗi ngày — kiểm tra không phá determinism (vẫn thuần: vị trí suy từ path + asOfDate).

## 10. Không tạo (YAGNI)
- Không model tiến độ mới · không gating mới · không remediation (M4-10) · không route mới.
