<!-- Tiếng Việt — dưới docs/, theo convention repo. Đặt vấn đề xuyên suốt các giai đoạn từ Phase 1 (đã xong)
     đến khi khóa học hiển thị & học được trên exe-web. Đây là tài liệu FRAMING (đặt vấn đề), chưa phải spec chi
     tiết — mỗi giai đoạn sẽ có bộ spec riêng khi tới lượt. -->
# Đặt vấn đề: Đường tới hiển thị khóa học trên exe-web (Phase 1 → Runtime)

- **Ngày:** 2026-07-17
- **Tác giả:** Dev + Claude (framing)
- **Trạng thái:** Nháp đặt vấn đề — chờ chốt câu hỏi mở §9 trước khi viết spec từng giai đoạn
- **Mục tiêu tài liệu:** định nghĩa **toàn bộ công việc còn lại** từ chỗ Phase 1 (Ingestion) đã xong cho tới khi
  **một khóa học import được hiển thị & học được trên `exe-web`**, để lập kế hoạch **triển khai một thể** thay vì
  làm rời rạc.
- **Nguồn:** `docs/roadmap-task-based-learning.md`, `specs/course-content-import/` (Phase 1 đã giao).

---

## 1. Điểm xuất phát & Đích đến

### Đã có (Phase 1 — Ingestion, giao xong 2026-07-16)

| Lớp | Hiện trạng |
|---|---|
| **API (`exe-api`)** | Module `course-content`: model cây nhúng, parser `.xlsx`, resolver tham chiếu, service `importCourse`. Admin-api: `POST /_/import`, `GET /template`, `GET /` (list), `GET /:id` (**trả cả cây**), `DELETE /:id` (chặn khóa `published`). |
| **Admin (`exe-admin`)** | Trang `/course-import`: upload + tải template + danh sách + xoá. **Không có** xem chi tiết, đổi trạng thái, sửa. |
| **Web (`exe-web`)** | **Không có gì** — `features/courses/` chỉ có `.gitkeep`; không route, không service, không endpoint đọc. |
| **Trạng thái khóa** | Import luôn tạo `status: 'ready'`. Enum đủ `['draft','ready','published','archived']` nhưng **không có phép chuyển trạng thái nào** được cài. |

### Đích đến

Một khóa đã import có thể được **admin xuất bản** rồi **learner mở trên exe-web**: thấy danh sách khóa, mở khóa
xem cây (Chặng → Chuyên đề → Bài học), mở Bài học đọc **lý thuyết + phát video/audio**, và mở **bài tập ipa/talk**
(engine đã có sẵn).

### Hai chốt chặn (blocking) rút ra từ khoảng trống

1. **Không xuất bản được:** learner chỉ nên thấy khóa `published`, nhưng chưa có đường đưa khóa từ `ready` →
   `published`. ⇒ Năng lực **đổi trạng thái (admin)** là *điều kiện cần* cho việc hiển thị.
2. **Không có bề mặt đọc cho web:** `course-content` chỉ tồn tại trong admin-api. exe-web cần một **API đọc dành
   cho learner** + hợp đồng api↔web.

---

## 2. Phân tích khoảng trống → 3 giai đoạn

Để đi từ "đã import" tới "hiển thị & học trên web", chia làm **3 giai đoạn nối tiếp** (đặt tên A/B/C để **tránh
đụng cách đánh số Phase của roadmap** — xem §3):

```
[Phase 1: Import]  ──►  A. Admin: Quản lý + Sửa + Xuất bản  ──►  B. API: Bề mặt đọc learner  ──►  C. exe-web: Hiển thị & học
   (ĐÃ XONG)              (api + exe-admin)                        (api)                          (exe-web)
                          ▲ sinh ra khóa `published`              ▲ hợp đồng api↔web              ▲ tiêu thụ API B
```

- **A** là tiền đề của B/C (không có khóa `published` thì web không có gì hợp lệ để hiển thị).
- **B** là hợp đồng mà **C** tiêu thụ — nên **chốt B trước, code C sau** (hoặc song song sau khi ký hợp đồng).
- Cả ba đều **không cần** tiến độ/mastery/điểm số (roadmap Phase 3) hay engine bài tập mới (roadmap Phase 4) — xem
  §8 phần hoãn.

---

## 3. Bản đồ thuật ngữ (đối chiếu roadmap)

Roadmap gốc đánh số Phase theo epic; tài liệu này cắt lát mịn hơn theo repo. Ánh xạ để không lẫn:

| Giai đoạn (tài liệu này) | Tương ứng roadmap | Ghi chú |
|---|---|---|
| **A — Admin quản lý/sửa/xuất bản** | Kéo sớm một phần **Phase 5 (Authoring)** + phần "chuyển trạng thái" nói ở Phase 2 | Cần để có khóa `published` phục vụ hiển thị |
| **B — API đọc learner** | Nửa **api** của **Phase 2 (Runtime)** | `specs/lesson-runtime/contracts/web-course-consume.md` |
| **C — exe-web hiển thị & học** | Nửa **web** của **Phase 2 (Runtime)** | Render + lesson viewer + mở ipa/talk |
| *(hoãn)* Tiến độ/mastery/unlock-điểm | **Phase 3** | §8 |
| *(hoãn)* Engine Nghe/Đọc/Viết + quiz | **Phase 4** | §8 |
| *(hoãn)* CMS kéo-thả + AI + versioning đầy đủ | **Phase 5** | §8 |

---

## 4. Giai đoạn A — Admin: Quản lý, Sửa & Xuất bản khóa

> Ranh giới: **`exe-api` + `exe-admin`**. Không đụng exe-web.

**Vấn đề.** Sau khi import, admin không xem được nội dung cây, không sửa được sai sót (chỉ có "xoá & import lại" —
quá thô cho khóa sắp publish), và không đưa được khóa lên trạng thái để learner thấy.

**Mục tiêu.** Admin xem chi tiết cây khóa; sửa nội dung; và chuyển vòng đời `draft → ready → published → archived`.

**Phạm vi — TRONG:**
- **Xem chi tiết** cây khóa (Chặng→Chuyên đề→Bài học→media/bài tập). *Backend `GET /:id` đã trả cả cây ⇒ chủ yếu là FE.*
- **Đổi trạng thái**: endpoint `PATCH /:id/status` + guard vòng đời + **re-validate khi publish** (chạy lại
  `validateCourse` + `resolveReferences`) + permission + audit + test. Điều khiển tương ứng trên UI.
- **Sửa nội dung**: đề xuất **ghi lại cả cây (`PUT /:id`) + re-validate** (tái dùng pipeline import, atomic) thay
  vì PATCH từng node; form sửa từng node trên UI.

**Phạm vi — NGOÀI (hoãn):** versioning đầy đủ, sửa khóa đang `published`, CMS kéo-thả, sinh nội dung bằng AI (→ Phase 5).

**Phụ thuộc:** Phase 1 (model/service/validate đã có, tái dùng). Không phụ thuộc B/C.

**Tiêu chí Done:** admin lấy một khóa `ready`, sửa vài Bài học, bấm **Xuất bản** → khóa thành `published`; nếu cây
lỗi thì publish bị chặn kèm lỗi chỉ rõ dòng/node.

**Câu hỏi mở:** → §9 (Q-A1 versioning/published-edit, Q-A2 độ mịn khi sửa, Q-A3 vòng đời & unpublish).

---

## 5. Giai đoạn B — API: Bề mặt đọc cho learner (hợp đồng api↔web)

> Ranh giới: **`exe-api`**. Sinh ra hợp đồng cho C.

**Vấn đề.** `course-content` mới chỉ có admin-api. exe-web cần API đọc khóa **`published`** cho learner, với dữ
liệu đã lọc (không lộ field nội bộ như `importedBy`, `sourceMeta`).

**Mục tiêu.** Endpoint learner đọc catalog + chi tiết khóa `published`, kèm hợp đồng ổn định để C code theo.

**Phạm vi — TRONG:**
- `GET /courses` — catalog khóa `published` (projection nhẹ: slug/title/thumbnail/mô tả/đếm tầng).
- `GET /courses/:slug` — cây khóa cho learner (ẩn field nội bộ; giữ theory/media/exercise refs).
- *(tuỳ chọn)* `GET /courses/:slug/lessons/:key` — nếu muốn tải lười từng Bài học.
- Contract `specs/lesson-runtime/contracts/web-course-consume.md`.

**Phạm vi — NGOÀI (hoãn):** ghi tiến độ, chấm điểm, unlock theo điểm (→ Phase 3).

**Phụ thuộc:** Giai đoạn A (cần khóa `published` để có dữ liệu hợp lệ trả về).

**Tiêu chí Done:** gọi `GET /courses` trả đúng các khóa đã publish ở A; `GET /courses/:slug` trả cây đầy đủ để
render; khóa chưa publish **không** lộ ra.

**Câu hỏi mở:** → §9 (Q-B1 assign/enroll, Q-B2 gate truy cập: cần đăng nhập/enroll hay xem tự do, Q-B3 đa-tenant `centerId`).

---

## 6. Giai đoạn C — exe-web: Hiển thị & học khóa (Runtime UI)

> Ranh giới: **`exe-web`**. Tiêu thụ API của B.

**Vấn đề.** exe-web chưa có gì cho course-content (`features/courses` rỗng). Cần dựng toàn bộ UI learner.

**Mục tiêu.** Learner thấy khóa, mở cây, học Bài học (lý thuyết + media), mở bài tập ipa/talk.

**Phạm vi — TRONG:**
- Route `(main)/courses` (catalog), `(main)/courses/[slug]` (chi tiết cây + điều hướng Bài học).
- **Lesson viewer**: render lý thuyết (markdown) + **player video/audio** cho `media[]`.
- **Mở bài tập**: nối tới engine **ipa/talk đã có** (tái dùng player pronunciation/talk hiện hữu).
- Service + hooks (`use-*`, TanStack Query) theo convention exe-web.
- *(tối thiểu, tuỳ chọn)* **unlock tuần tự** đơn giản — mở Bài/Chặng kế tiếp, **chưa gắn điểm số** (điểm số → Phase 3).

**Phạm vi — NGOÀI (hoãn):** thanh tiến độ %, mastery, unlock theo ngưỡng điểm, gamification (→ Phase 3); bài tập
Nghe/Đọc/Viết/quiz (→ Phase 4).

**Phụ thuộc:** Giai đoạn B (hợp đồng đọc) — nên B ký xong trước.

**Tiêu chí Done:** learner đăng nhập → `/courses` → chọn khóa → mở Bài học → đọc lý thuyết, xem video, nghe audio,
bấm vào bài tập ipa/talk và luyện được.

---

## 7. Đường găng & thứ tự "triển khai một thể"

```
A (admin: sửa + xuất bản)  ─────────────►  cần trước, vì sinh khóa `published`
        │
        ├─►  B (API đọc learner)  ──► ký hợp đồng web-course-consume
        │            │
        │            └─►  C (exe-web hiển thị)  ── tiêu thụ B
```

Khuyến nghị triển khai một thể theo lát:
1. **A trước** (api trạng thái/sửa → admin UI). Kết thúc A: có ≥1 khóa `published` thật để test B/C.
2. **B ngay sau A** — chốt & ký `web-course-consume.md`.
3. **C** bắt đầu **ngay khi hợp đồng B ổn định** (có thể song song phần cuối B nếu hợp đồng đã khóa).
4. Deploy thứ tự: `exe-api` (A+B) → `exe-admin` (A) → `exe-web` (C).

> Vì cả 3 dùng chung một model `course_structures` (đã chốt từ Phase 1, không đổi khung), rủi ro tích hợp thấp;
> điểm rủi ro duy nhất là **hợp đồng B** — nên viết & review kỹ trước khi C bám vào.

---

## 8. Ngoài phạm vi lần này (nêu rõ để chống scope creep)

| Hạng mục | Thuộc | Vì sao hoãn |
|---|---|---|
| Tiến độ %, mastery, streak, **unlock theo điểm** | Roadmap Phase 3 | "Hiển thị & học" không cần đo lường; cần runtime chạy trước để phát sinh dữ liệu |
| Engine bài tập **Nghe/Đọc/Viết/Ngữ pháp/Từ vựng + `quiz`** | Roadmap Phase 4 | Chưa có engine; ipa/talk đã đủ để "học được" ở mức hiển thị |
| **CMS kéo-thả**, AI generator, host media tập trung, **versioning đầy đủ** | Roadmap Phase 5 | Giai đoạn A chỉ kéo sớm "sửa + xuất bản" tối thiểu, không làm CMS đầy đủ |
| Đa-tenant (center tạo khóa riêng) | Phase 2+ | Phase 1 dùng `centerId=null` dùng chung; mở khi có nhu cầu thật |

---

## 9. Câu hỏi mở phải chốt trước khi viết spec

**Giai đoạn A**
- **Q-A1 (versioning/published-edit):** khóa `published` có cho sửa trực tiếp không? *Đề xuất:* **không** — chỉ sửa
  tại chỗ khi `draft`/`ready`; muốn sửa published phải `archive`/clone. Versioning đầy đủ → Phase 5.
- **Q-A2 (độ mịn sửa):** PATCH từng node hay **PUT cả cây + re-validate**? *Đề xuất:* PUT cả cây (tái dùng pipeline, atomic).
- **Q-A3 (vòng đời):** cho phép `published → ready` (gỡ publish) không? *Đề xuất:* có, kèm cảnh báo; chặn nếu (sau này) có learner enroll.

**Giai đoạn B**
- **Q-B1 (assign/enroll):** learner có khóa thế nào — tự chọn / theo placement / admin gán? *Đề xuất tối thiểu để hiển thị:* xem tự do khóa `published`; enroll để dành cho Phase 3.
- **Q-B2 (gate truy cập):** đọc khóa cần đăng nhập không? *Đề xuất:* cần đăng nhập, nhưng chưa cần enroll.
- **Q-B3 (đa-tenant):** giữ `centerId=null` dùng chung ở giai đoạn này? *Đề xuất:* có.

**Giai đoạn C**
- **Q-C1 (unlock):** có làm unlock tuần tự tối thiểu ngay không, hay hiển thị mở toàn bộ? *Đề xuất:* mở toàn bộ trước (đơn giản), unlock gắn điểm để Phase 3.

---

## 10. Bước tiếp theo

1. Chốt §9 (đặc biệt Q-A1/Q-A2 và Q-B1) — quyết định độ lớn & data-model.
2. Viết spec từng giai đoạn theo khuôn Phase 1 (`spec · design · data-model · contracts · acceptance · tasks`):
   - `specs/course-management-admin/` (Giai đoạn A)
   - `specs/lesson-runtime/` (Giai đoạn B + C; contract `web-course-consume.md`)
3. Triển khai theo thứ tự §7.
