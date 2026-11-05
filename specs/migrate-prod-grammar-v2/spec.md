# Spec: Migrate nội dung ngữ pháp prod lên v2

- **Ngày:** 2026-08-24
- **Tác giả:** AI (spec-workflow LITE) — từ điều tra dữ liệu dev/prod trực tiếp
- **Repos/surfaces ảnh hưởng:** `exe-api` (services/api — scripts + 2 collection trên prod DB `enova`). `exe-web` là bên tiêu thụ, **không đổi code** (đã v2-aware). Không đụng `exe-admin`/mobile.
- **Module liên quan:** `services/api/src/modules/grammar-content`, `services/api/src/modules/grammar-practice`
- **Trạng thái:** Nháp — chờ review

## 1. Mục tiêu

Đưa dữ liệu ngữ pháp trên **prod** (`enova`) từ định dạng v1 (phẳng, slug snake_case, bank bài tập nhỏ) lên **v2** (nested `children` + `summary`/`objectives`, slug kebab-case, bank 448 bài) cho khớp dev, để prod web hiển thị đầy đủ trải nghiệm v2 mà không mất tiến độ học của user.

## 2. Bối cảnh

Đo trực tiếp 2 DB (24/08/2026):

| Collection | Dev (`enova_dev`) | Prod (`enova`) |
|---|---|---|
| `grammar_lesson_contents` | 55 v2 (kebab, có summary/children), 55 active | 55 v1 (snake, phẳng, không summary), 36 active + 19 draft |
| `grammar_items` | 448 published (kebab, phủ 55 điểm) | 184 published (snake, phủ 22 điểm) |
| `grammar_attempts` | 56 (test) | 6 (user thật, 2 điểm) |

- 55↔55 là **cùng bộ bài**, chỉ khác slug + cấu trúc (khớp 1:1).
- FE (`exe-web` `GrammarLessonContainer.tsx`) v2-aware nhưng tự thoái xuống render phẳng khi thiếu `children`/`summary` → prod hiện **chạy được nhưng mất summary/mục tiêu + không có accordion phân mục**. Đây là nợ kỹ thuật, không phải bug đang lỗi.
- Coupling: `grammar_items.grammar.grammarPoint` + `grammar_attempts.grammarPointAtTime` neo theo slug (verify `grammar-practice.service.js:61,96`). Đổi lesson slug mà không xử lý item/attempt ⇒ mồ côi bài tập + reset mastery.

## 3. Phạm vi

### Trong phạm vi
- Thay `grammar_lesson_contents` prod (55 v1) bằng bản dev (55 v2).
- Thay `grammar_items` prod (184) bằng bản dev (448).
- Reslug `grammar_attempts.grammarPointAtTime` của prod (giữ 6 attempt user thật).
- Backup trước + đường rollback.

### Ngoài phạm vi
- **Sửa code `exe-web`** — **Lý do:** FE đã backward-compatible sẵn, render v2 ngay khi data có `children`/`summary`; không cần đổi.
- **Copy `grammar_attempts` từ dev** — **Lý do:** dev là data test, `userId` không tồn tại trên prod, sẽ bơm mastery giả.
- **Đổi schema / model** — **Lý do:** model đã hỗ trợ v2 (`summary`/`objectives`/`children` là field sẵn có); đây thuần data migration.
- **Migrate các môi trường khác** — chỉ prod.

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Learner (prod) | Mở bài ngữ pháp | Thấy tóm tắt + mục tiêu + các mục con (accordion) + bài luyện tập, như dev |
| Learner đã học | Giữ tiến độ mastery | Không bị reset về "chưa học" sau migrate |

## 5. Quyết định nghiệp vụ cần chốt

Đã chốt với owner (24/08/2026) — không còn treo:

| Câu hỏi | Đã chọn | Người quyết |
|---|---|---|
| Bank bài tập | Import bộ 448 của dev (thay 184) | owner |
| Phạm vi publish | Cả 55 active | owner |
| 6 attempt user | Reslug giữ mastery | owner |
| Ai chạy | Owner tự chạy trực tiếp tới prod | owner |

## 6. Acceptance criteria (tóm tắt)

1. Prod `grammar_lesson_contents` = 55, slug **toàn kebab**, status **active**, mỗi doc có `summary` + `children`.
2. Prod `grammar_items` = **448**, `grammar.grammarPoint` toàn kebab, **0 item mồ côi** (mọi điểm có lesson tương ứng).
3. Prod `grammar_attempts` = **6** (không mất), `grammarPointAtTime` toàn kebab.
4. So sánh dev↔prod: **0 lesson khác biệt** về nội dung.
5. Prod web `/grammar/present-simple` hiển thị v2: có summary + accordion phân mục + tab Luyện tập ≥ 1 câu.
6. Backup 3 collection tồn tại (size > 0) **trước khi** ghi; script rollback chạy được.

## 7. Câu hỏi mở

- [ ] Prod MongoDB `27017` có cho **ghi** qua `directConnection=true` không, hay là replica-set secondary (insert sẽ báo `not primary`)? Chưa test được vì không thể thử ghi mà không thực sự ghi lên prod. → Xử lý: Task 2 dừng ngay nếu gặp `not primary`, chuyển sang URI `replicaSet=` + seed list (cần topology prod từ owner).

## 8. Ghi chú cho Tech Lead Design

- Prod là DB thật đang phục vụ user → **backup bắt buộc**, thao tác phải ngắn, chọn giờ thấp tải.
- Dùng `deleteMany` (không `drop`) để giữ index/unique constraint của prod.
- Không hardcode secret trong script; đọc URI qua env var.
- `output-v2/*.v2.json` (nguồn của import-script cũ) git-ignored → **không** dùng đường import-script cho item; phải copy trực tiếp dev→prod.
