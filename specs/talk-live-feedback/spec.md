# Spec: Feedback cuối buổi cho Trò chuyện trực tiếp (Phase A — text)

- **Ngày:** 2026-09-10
- **Tác giả:** AI Brainstorm + dev
- **Repos/surfaces ảnh hưởng:** `api` (rubric + endpoint + lưu), `web` (gom transcript + hiển thị), `llm` (tái dùng `/chat`, không sửa)
- **Module liên quan (exe-api):** `services/api/src/modules/talk/`
- **Trạng thái:** Nháp — chờ review
- **Liên quan:** nối tiếp `specs/talk-live-realtime/` (TN1)

## 1. Mục tiêu

Sau mỗi buổi "Trò chuyện trực tiếp", sinh một **báo cáo góp ý có cấu trúc** theo bộ tiêu chí (rubric) cố định, chấm theo trình độ CEFR của người học, hiển thị ở `AssessmentCard` (đang mock).

## 2. Bối cảnh

- TN1 (realtime) đã xong nền móng. Gia sư AI **đã sửa nhẹ tức thì** trong lúc nói (charter). Feature này thêm lớp **báo cáo tổng kết cuối buổi**.
- Đã kiểm chứng: `AgentSession` của LiveKit **mặc định publish transcript** (user + AI) qua text-stream `lk.transcription`; FE nhận qua `registerTextStreamHandler` — lấy transcript **không cần sửa worker**.
- `AssessmentCard.tsx` hiện 100% mock (thanh grammar/vocab/…) — sẵn khung UI.
- `llm /chat` + adapter (OpenAI/Claude/…) + moderation sẵn có để sinh feedback.
- Khung phát âm IPA (`analyzeAudio`) tồn tại nhưng **để Phase B** (cần audio user).

## 3. Phạm vi

### Trong phạm vi (Phase A)
- Bộ rubric **5 mục text**, seed cứng trong code, CEFR-aware: Ngữ pháp · Từ vựng · Bám chủ đề/task · Tương tác · Mạch lạc.
- FE gom transcript trong buổi; kết thúc buổi → gọi api sinh feedback.
- Endpoint `api` dựng prompt từ rubric + CEFR → gọi `llm /chat` → nhận **JSON có cấu trúc** → lưu → trả FE.
- Hiển thị báo cáo ở `AssessmentCard`.

### Ngoài phạm vi
- **Phát âm (mục 6, khung IPA)** — **Lý do:** cần thu + xử lý audio user (Phase B).
- **Feedback giữa buổi** — **Lý do:** gia sư đã sửa nhẹ tức thì; báo cáo cuối buổi rẻ + tổng thể hơn.
- **Rubric admin chỉnh được** — **Lý do:** seed cứng trước (YAGNI); admin hoá sau.
- **Chấm điểm gắn quota/gating** — **Lý do:** feedback chỉ để học, không chặn gì.

## 4. User story / Actor

| Actor | Muốn làm gì | Để làm gì |
|---|---|---|
| Học viên | Sau buổi nói, xem góp ý theo từng mục | Biết điểm yếu cụ thể để cải thiện |
| Học viên | Góp ý đúng trình độ mình | Không bị chấm quá khó/dễ |

## 5. Quyết định nghiệp vụ cần chốt

| Câu hỏi | Đề xuất | Người quyết |
|---|---|---|
| Bộ 5 mục rubric có đúng trọng tâm? | Làm rồi đánh giá sau (user đồng ý) | Product owner |
| Có lưu transcript buổi nói không? | Có (dữ liệu của chính người học; cần cho Phase B) | Product owner |

## 6. Acceptance criteria

1. Kết thúc buổi có ≥1 lượt người học nói → tự sinh feedback, hiển thị báo cáo trong luồng Live (`LiveFeedbackReport`, dữ liệu thật — không đụng `AssessmentCard` mock của mode cũ).
2. Báo cáo gồm: tóm tắt tổng + **5 mục** (mỗi mục: level + ví dụ/gợi ý) + "top điều cần cải thiện".
3. Chấm theo **CEFR** của người học khi có (`user.level`); không có thì chấm chung.
4. Endpoint chỉ cho người **đã đăng nhập** (chưa đăng nhập → 401).
5. Transcript rỗng / buổi quá ngắn → **không gọi AI**, báo nhẹ "chưa đủ dữ liệu để góp ý".
6. AI trả JSON hỏng → **fallback an toàn** (không crash; hiện thông báo lỗi thân thiện).
7. Mode nói theo lượt cũ (`/talk/turn`) vẫn nguyên vẹn.

## 7. Câu hỏi mở

- [ ] (Không có gì chặn — Phase B phát âm tách riêng.)

## 8. Ghi chú cho Tech Lead Design

- Transcript do **FE gửi lên** (client-controlled). Feedback không gating gì nên **chấp nhận**; nếu sau cần chống giả mạo → chuyển sang lấy transcript phía worker. Ghi rõ trade-off trong design.
- Reuse `llm /chat` — không tạo route AI mới. Business logic (rubric, lưu) ở `api`.
- Output phải là JSON parse được → prompt ép JSON + parse phòng thủ (fallback ở AC6).
