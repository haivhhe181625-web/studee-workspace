# ADR-0002 — Thêm hạ tầng thoại realtime (LiveKit + mô hình giọng nói) cho mode Trò chuyện trực tiếp

**Ngày:** 2026-09-10 · **Trạng thái:** Đề xuất · **Thay thế:** không có

## Bối cảnh

Mode talk hiện tại theo lượt (REST: thu âm → STT → chat → TTS) có độ trễ cả câu, không ngắt lời, không liên tục. Cần một mode **nói–nghe realtime** để luyện phản xạ hội thoại. Tự xây lớp truyền âm WebRTC 2 chiều độ trễ thấp là quá tầm cho giai đoạn này. Đồng thời AI cần một phiên giọng nói realtime thường trú trong phòng — khác hẳn kiểu request/response của `llm` hiện tại. Đây là 2 thay đổi kiến trúc thật (dependency ngoài mới + kiểu tải mới) nên cần ADR.

## Quyết định

1. **Dùng LiveKit Cloud (bản free)** làm lớp vận chuyển WebRTC/SFU cho thoại realtime, thay vì tự xây. Bắt đầu bản free, quản chi phí sau.
2. **Dùng mô hình giọng nói realtime của Google (Gemini)** làm bộ não hội thoại, gọi **qua adapter** trong `llm/src/adapters` — tuân luật "provider ngoài luôn qua adapter".
3. **Đặt worker AI trong service `llm`** dưới dạng **entrypoint riêng, tiến trình thường trú**, tách biệt tiến trình FastAPI. `llm` từ nay có 2 kiểu tải: ACL request/response (cũ) + worker thường trú (mới).
4. **Service `api` chỉ cấp vé** vào phòng qua REST (`POST /api/talk/live/token`), sau khi kiểm đăng nhập; không đụng luồng streaming.
5. **Bí mật chỉ ở server** — khóa LiveKit/Google không bao giờ xuống client; FE chỉ nhận vé ngắn hạn.
6. **Mode mới chạy song song**, không thay thế `/talk/turn`.

## Lộ trình

| Phase | Nội dung | Trạng thái |
|---|---|---|
| 0 | Spike đo độ trễ (throwaway) | ✅ đạt — ~1s nói→nghe, worker region Japan, Gemini realtime qua LiveKit chạy tốt (2026-09-10) |
| 1 | `api` cấp vé | ✅ `POST /api/talk/live/token` (talk-live.service.js) — verifyToken + rate limit, mint token 15m/phòng riêng; 2 unit test pass (2026-09-10) |
| 2 | `llm` worker + adapter giọng nói | ✅ adapter đổi 2 hãng (gemini mặc định/openai), worker entrypoint riêng + log metrics đánh giá; 5 unit test pass, symbol livekit verified (2026-09-10) |
| 3 | FE mode trực tiếp | ✅ service getLiveToken + hook use-live-session (livekit-client) + TalkLivePanel; entry song song trong TalkContainer, không đụng mode cũ; typecheck + lint sạch (2026-09-10) |
| 4 | Ghép nối + kiểm thử + giới hạn thời lượng | ✅ cap thời lượng (worker session.shutdown + client timer + kết thúc khi AI rời phòng), feature flag FE, runbook end-to-end; typecheck/compile sạch (2026-09-10) |

> **Nền móng code xong (Phase 0–4).** E2E headless đã verify (2026-09-10): worker thật register LiveKit project thật → dispatch job → adapter Gemini → **Gemini realtime sinh lời chào** (ttft 0.62s, metrics log đúng). Fix 1 bug tích hợp: plugin phải register ở main thread nên chuyển import provider từ lazy-trong-hàm sang top-level của adapter (adapter chỉ worker import, không lọt FastAPI).
>
> Còn chờ: (a) human Technical Review duyệt ADR + design; (b) quyết định nghiệp vụ moderation realtime (spec §5); (c) kiểm thử tay với mic thật + trình duyệt (runbook trong plans/260908-0021-talk-live-foundation/); (d) minor: `metrics_collected` API deprecated ở livekit-agents 1.8 (vẫn chạy, đổi sang `session_usage_updated` sau).

## Hệ quả

- **Dependency ngoài mới:** LiveKit + Google (mô hình giọng nói). Thêm biến env: `LIVEKIT_URL/KEY/SECRET` (ở `api` và `llm`), `GOOGLE_API_KEY` (ở `llm`).
- **`llm` không còn thuần request/response** — cần tài liệu hóa cách chạy worker (khác cách chạy FastAPI).
- Giới hạn bản free (1.000 phút/tháng, 5 phiên đồng thời) → phải đặt giới hạn thời lượng mỗi buổi; nâng gói khi scale.
- **Chưa** kiểm duyệt nội dung ở luồng realtime (pipeline moderate cũ không áp được) → dựa an toàn mô hình + hiến chương; cần quyết định nghiệp vụ (spec §5). Bổ sung moderate ở đợt sau.
- Cập nhật `docs/adr/0000-index.md` (thêm dòng ADR-0002).
