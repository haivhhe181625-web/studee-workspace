# Spec: IPA quota tính theo bài + cap 2 lần chấm mỗi mục

- **Ngày:** 2026-08-19
- **Tác giả:** AI Brainstorm (chốt cùng owner qua hội thoại "limit")
- **Repos/surfaces ảnh hưởng:** `exe-api` (backend, chính) · `exe-web` (learner UI)
- **Module liên quan (exe-api):** `services/api/src/modules/quota/`, `services/api/src/modules/ipa/`, `services/api/src/modules/user/user.model.js`
- **Trạng thái:** Nháp — chờ BA review

## 1. Mục tiêu

Cho học viên (khách và đã đăng nhập) **hoàn thành trọn vẹn bài luyện phát âm IPA** trong hạn mức miễn phí, đồng thời **chặn spam thu âm** bằng cách tính quota theo **số bài** thay vì theo từng lần chấm, và giới hạn **mỗi mục chỉ chấm tối đa 2 lần**.

## 2. Bối cảnh

Hiện trạng (đã kiểm code, không đoán):
- Quota IPA tính theo feature `pronunciation` — **trừ 1 đơn vị mỗi lần chấm audio** (`ipa.service.js` `submitItem` gọi `withQuota(user,'pronunciation',…)`; `submitMiniTest` gọi `consumeQuota`). Cấu hình ở `quota.config.js` (`TIERS`).
- PR #69 vừa merge vào `develop` nâng `pronunciation` lên guest 5 / free 5 (+bonus → max 7).
- **Vấn đề:** 1 bài IPA có ~11 mục "Đọc theo" (8 từ + 3 câu; template F: 8 từ + 4 hội thoại) — ma trận nội dung `ipa-content.service.js` `TEMPLATE_PLAN`. Mỗi mục thu âm = 1 lần chấm. → Học trọn 1 bài cần **~12 lần chấm**, nên limit 5–7 **không đủ hoàn thành nổi 1 bài** rồi bị 402 giữa chừng. UX gãy.
- Đã có sẵn cơ chế đếm số lần thử mỗi mục: `ipa.service.js:249-250` (`IpaAttempt.find({userId,lessonId,itemId})` → `attemptNo`). `recordIpaAttempt` được gọi mỗi lần chấm.
- Cảm âm ("Nghe & phân biệt", minimal pairs) chấm client-side, **không tốn quota** (`recordPractice`, route `/practice` không limiter).
- Bonus pool (`quotaUsed.bonus`, free 2) là quỹ fallback dùng chung; IPA hiện có mượn được.

## 3. Phạm vi

### Trong phạm vi

- Đổi granularity quota IPA: từ **per-lần-chấm** → **per-bài** (đếm số bài đã từng chấm audio).
- Hạn mức số bài (trọn đời): **cả guest lẫn free = 1 bài** (giống nhau).
- Cap **2 lần chấm/mục** (trọn đời theo mục), áp cho **mọi item có mic**: từng mục "Đọc theo" (word/sentence/dialogue) **và** từng câu mic trong Mini Test.
- Cap 2 lần **dùng chung cả khi ôn tập SRS** (không cấp thêm lượt).
- IPA **không dùng** bonus pool nữa (đếm theo bài).
- FE: khoá nút thu âm khi mục đã đủ 2 lần; modal phân biệt "hết bài" vs "hết lượt mục".

### Ngoài phạm vi

- **conversationTurns / assessment / adaptive / flashcardGeneration** — **Lý do:** phiên bàn này chỉ chốt IPA; các feature khác giữ nguyên hạn mức + bonus.
- **Chuyển quota sang reset ngày/tháng** — **Lý do:** owner chốt tạm giữ mô hình trọn đời (RT-13 billing để sau).
- **Đưa hạn mức lên admin config** — **Lý do:** đã thống nhất là follow-up sau launch, không chặn feature này.
- **Phân tầng engine (guest→wav2vec)** — **Lý do:** owner chọn (3a) không phân tầng; 5/7… nay thay bằng mô hình bài, engine selection giữ nguyên theo ngân sách Azure.
- **Migration dữ liệu `quotaUsed.pronunciation` cũ** — **Lý do:** counter cũ vô hại khi bỏ dùng; không cần backfill (xem design §9).

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Khách (guest) | Học thử trọn 1 bài IPA, mỗi mục nói tối đa 2 lần | Trải nghiệm đủ giá trị trước khi đăng ký |
| Học viên free (đã login) | Học trọn 1 bài IPA | Trải nghiệm đủ giá trị để cân nhắc nâng cấp |
| Chủ sản phẩm | Chặn spam thu âm + giới hạn chi phí AI | Bảo vệ ngân sách provider (Azure/LLM) |

## 5. Quyết định nghiệp vụ cần chốt

Đã chốt hết với owner trong hội thoại — không còn mục treo:

| Câu hỏi | Đã chốt | Người quyết |
|---|---|---|
| Số bài guest / free | 1 / 1 (cả hai loại như nhau) | owner |
| Cap lần chấm mỗi mục | 2 (trọn đời, dùng chung SRS) | owner |
| Bonus pool cho IPA | Bỏ | owner |
| Mini Test | Áp cùng cap 2 lần/câu mic (không luật riêng) | owner |

## 6. Acceptance criteria (tóm tắt)

1. Guest chấm audio ở **1 bài** → sang bài thứ 2 chấm audio đầu tiên bị **402** (hết bài trải nghiệm).
2. Free chấm audio ở **1 bài** → sang bài thứ 2 cũng bị **402** (giống guest).
3. Vào lại **cùng một bài** đã tính không bị trừ thêm lượt bài (chấm nhiều mục trong bài đó vẫn thuộc 1 lượt bài).
4. Một mục (word/sentence/dialogue) chấm audio **lần 3** → bị chặn (**403**, "hết lượt mục này"), lần 1–2 vẫn chấm được.
5. Câu mic trong Mini Test cũng bị cap **2 lần** như trên.
6. Cảm âm / TAP / trắc nghiệm **không** trừ lượt bài và **không** bị cap.
7. User `quotaUnlimited` hoặc `TRIAL_LIMIT_MODE=unlimited` → **không** bị cả 2 tầng giới hạn.
8. Cap 2 lần/mục **tính cả lượt ôn tập SRS** (không reset khi review).
9. IPA **không** rút từ `quotaUsed.bonus`; bonus của các feature khác không đổi.
10. FE: khi mục đủ 2 lần → nút thu âm khoá + thông báo; khi hết bài → modal "hết bài, đăng ký/nâng cấp" phân biệt với "hết lượt mục".

## 7. Câu hỏi mở

Không còn — mọi quyết định nghiệp vụ đã chốt (mục 5). Chi tiết kỹ thuật xử lý ở `design.md`.

## 8. Ghi chú cho Tech Lead Design

- Có **nhiều entrypoint chấm audio** phải phủ hết cùng luật: `submitItem`, `submitMiniTest`, và `submitAttempt` (merged write FR-006, route `/attempts`). Nên **tập trung gate vào 1 helper dùng chung** (DRY) thay vì rải logic.
- Tận dụng `IpaAttempt` count sẵn có cho cap-2-lần (không cần schema mới cho tầng trong).
- Tầng ngoài (số bài) cần biết **bài nào đã tính** để không trừ trùng → cần lưu tập lessonId đã chấm trên User.
- Hot path chấm audio nhạy cảm (consume-before-LLM + refund on error, RT-3/RT-4) — giữ nguyên tính atomic/idempotent, không phá.
- Ràng buộc chi phí: giữ trần ngân sách Azure + fallback wav2vec làm phanh tổng; 2 tầng này chặn per-user.
