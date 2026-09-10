# Design: Trò chuyện trực tiếp (realtime) với AI

- **Spec:** `specs/talk-live-realtime/spec.md`
- **Ngày:** 2026-09-10
- **Tác giả:** Tech Lead Agent + dev
- **Trạng thái:** Nháp — chờ Technical Review
- **ADR liên quan:** `docs/adr/0002-realtime-voice-livekit-gemini.md`

## 1. Tóm tắt kiến trúc

Thêm mode realtime **bên cạnh** mode talk theo lượt (không thay thế). Dùng hạ tầng truyền âm ngoài (LiveKit, bản free) làm lớp vận chuyển WebRTC. AI là một **worker thường trú** trong service `llm`: worker vào phòng như một người tham gia, nối mô hình giọng nói realtime (Gemini) **qua adapter**, nghe/nói 2 chiều. Service `api` giữ vai trò mỏng: kiểm đăng nhập → cấp vé vào phòng (token ngắn hạn). FE dùng SDK client của hạ tầng để vào phòng bằng vé.

Không mâu thuẫn `ARCHITECTURE.md` (Node = I/O/business, Python = ML/audio). Điểm mới cần ghi ADR: (a) dependency ngoài mới, (b) worker thường trú trong `llm` vốn chỉ có request/response — xem ADR-0002.

## 2. Quyết định kiến trúc (đã chốt)

| Quyết định | Lựa chọn | Lý do | Đánh đổi |
|---|---|---|---|
| Lớp vận chuyển | LiveKit Cloud bản free | Không tự xây WebRTC/SFU; có sẵn plugin nối mô hình giọng nói | Thêm 1 vendor + giới hạn free (1.000 phút, 5 phiên) |
| Cấp vé | REST `POST /api/talk/live/token` | Xin vé là request→response; nhất quán `/talk/turn` (talk là REST) | Không dùng socket (socket để push, không hợp) |
| Nơi đặt worker AI | Trong `llm`, entrypoint riêng | Gộp service, tái dùng env/pattern Python | Trộn 2 kiểu tải; chạy tiến trình tách biệt FastAPI |
| Gọi mô hình giọng nói | Qua adapter mới trong `llm/src/adapters` | Luật `llm`: provider ngoài luôn qua adapter | Thêm 1 lớp bọc |
| Kiểm đăng nhập cấp vé | Tái dùng middleware JWT sẵn có | Đồng nhất auth toàn hệ | — |

## 3. Data model

Nền móng **không thêm collection mới**. Không ghi transcript/lịch sử ở đợt này (xem spec §3 ngoài phạm vi). Định danh phòng suy ra từ `userId` (mỗi user một phòng riêng cho buổi), không lưu.

## 4. Luồng dữ liệu

```
FE  --(POST /api/talk/live/token, JWT)-->  api
      verify JWT (middleware sẵn có)
      -> talk.service: mintLiveToken(userId)  (ký bằng livekit-server-sdk, hạn ngắn)
      -> { url, token, roomName }
FE  --(join room bằng vé, WebRTC)-->  LiveKit room
llm worker  --(tự vào cùng room)-->  LiveKit room
      -> phiên giọng nói realtime qua adapter mô hình (nghe user, sinh giọng trả lời)
      -> nạp system prompt = hiến chương gia sư (rút gọn, phỏng theo talk.rules.js)
FE  <--(audio 2 chiều)-->  worker
```

## 5. Contracts

- `contracts/talk-live-token.md` — request/response endpoint cấp vé (JWT vào, `{url, token, roomName}` ra).

## 6. File structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `services/api/src/modules/talk/talk.routes.js` | Thêm route `POST /talk/live/token` | Sửa |
| `services/api/src/modules/talk/talk.service.js` | Hàm `mintLiveToken(userId)` | Sửa |
| `services/api/src/modules/talk/talk.schema.js` | (Nếu cần) validate input | Sửa |
| `services/api/.env.example` | Thêm `LIVEKIT_URL/KEY/SECRET` | Sửa |
| `services/llm/src/adapters/<livekit-gemini-voice>.py` | Adapter bọc mô hình giọng nói realtime | Tạo |
| `services/llm/src/<talk-live-agent-worker>.py` | Entrypoint worker vào phòng | Tạo |
| `services/llm/requirements` | Thêm thư viện worker + plugin | Sửa |
| `services/llm/.env.example` | Thêm `LIVEKIT_*` + `GOOGLE_API_KEY` | Sửa |
| `exe-web/src/services/talk-live.service.ts` | Gọi lấy vé | Tạo |
| `exe-web/src/hooks/use-live-session.ts` | Logic phiên trực tiếp | Tạo |
| `exe-web/src/components/features/talk/<live-mode>` | UI mode trực tiếp | Tạo |

## 7. Xử lý lỗi

| Tình huống | Mã HTTP | error code |
|---|---|---|
| Chưa đăng nhập gọi cấp vé | 401 | AUTH_REQUIRED |
| Thiếu cấu hình LiveKit ở server | 500 | LIVE_CONFIG_MISSING |
| Quyền micro bị từ chối (FE) | — | hiển thị thông báo FE |

## 8. Bảo mật & quyền

- Khóa LiveKit/mô hình **chỉ ở server** (`api` ký vé, `llm` chạy worker). FE không giữ khóa.
- Vé hạn ngắn (vd 5 phút), gắn đúng phòng của user; không tái sử dụng cho phòng khác.
- Chưa thêm permission string mới (dùng auth học viên hiện có). Nếu sau này gắn quota → theo `<resource>:<action>`.

## 9. Rủi ro / đánh đổi / câu hỏi kỹ thuật mở

- **Độ trễ** là rủi ro chính → **Phase 0 spike** đo trước khi code bản chính (throwaway).
- Giới hạn free (5 phiên đồng thời) → test lần lượt; đặt thời lượng cứng mỗi buổi.
- Quyền dùng mô hình giọng nói realtime của khóa Google — xác nhận ở spike.

## 10. Testing strategy

- `api`: unit test `mintLiveToken` (có/không JWT → 401; token có hạn, đúng room).
- E2E thủ công: FE kết nối → nói → nghe → kết thúc; kiểm mode cũ không hỏng.
- Không test hạ tầng LiveKit (bên thứ 3).

## 11. Rollout / cross-repo sequencing

Thứ tự: (1) `api` cấp vé → (2) `llm` worker → (3) `web` UI. Deploy `api`+`llm` trước, FE sau (FE cần endpoint sẵn). Có công tắc bật/tắt mode để ẩn khi demo lỗi.

## 12. Đối chiếu Acceptance criteria

| Acceptance criterion | Được đáp ứng bởi |
|---|---|
| AC1 (vé + chặn đăng nhập) | §4 luồng + §6 talk.routes/service + §7 lỗi |
| AC2 (kết nối + micro) | §6 FE hook/service |
| AC3 (nghe AI trả lời) | §4 worker + §6 adapter/worker |
| AC4 (ngắt lời) | §4 phiên realtime (nhận diện lượt của mô hình) |
| AC5 (trạng thái UI) | §6 `use-live-session.ts` |
| AC6 (kết thúc, nhả mic) | §6 FE hook cleanup |
| AC7 (tự dừng thời lượng) | §11 công tắc + giới hạn buổi |
| AC8 (mode cũ nguyên vẹn) | §1 song song, không sửa luồng `/talk/turn` |
| AC9 (độ trễ < 2s) | §9 spike Phase 0 |
