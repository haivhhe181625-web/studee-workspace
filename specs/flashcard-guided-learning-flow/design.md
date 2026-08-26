---
feature: flashcard-guided-learning-flow
status: draft
role: TechLead
created: 2026-08-25
revised: 2026-08-26
supersedes: guided-funnel (bản 25/08) — đổi sang "menu chọn + thang màu năng lực"
---

# Design — Học flashcard: menu chọn cách học + thang màu theo năng lực

Bản 25/08 (funnel ép introduce→match→quiz→type→review) bị bỏ: ép 1 khuôn, mất hồn lật thẻ,
1 phiên 20 từ ≈ 80 lượt. Bản này: **người học chọn cách học/kiểm tra; màu của thẻ = hoạt động
khó nhất từng làm đúng; thời gian làm màu tụt; ôn làm mới**. Giữ engine FSRS + 4 runner nguyên trạng.

## 0. Nguyên tắc
- **Không ép chuỗi.** Trang bộ thẻ là **menu 3 hành động**: Học từ mới · Ôn đến hạn · Kiểm tra / Luyện tập.
- **Ba trục tách biệt** cho mỗi thẻ:
  - **Nấc màu** = năng lực đã chứng minh (lật<ghép<trắc nghiệm<tự luận). Chỉ LÊN khi trả lời đúng mode khó hơn.
  - **Tụt** = đến hạn (`due≤now`) → thẻ GIỮ nấc màu, tô **sọc chéo chồng lên** cùng màu (không rời đoạn, không mờ dần).
  - **⭐ Đã nhớ** = mastered ≥21 ngày (isMastered), trục riêng chồng lên màu.
- **Phân vai rõ:** *Ôn đến hạn* = nhắc lại → **lấp tụt** (làm mới `due`). *Kiểm tra* = làm bài khó hơn → **leo nấc màu**. Đúng lời PO: "ôn lại chỉ là nhắc lại; làm bài để lên trình".
- **new-limit = cỡ lô mỗi LƯỢT** (không phải hạn mức/ngày): "20 luôn là 20 từ mới sẵn sàng", KHÔNG trừ số đã học trong ngày. Hàng đợi due theo ngày VN.

## 1. Mô hình mastery — thang màu năng lực

### 1.1 Field mới: `bestRecallLevel` (0–4) trên SRS row
`flashcard-srs.model.js` thêm `bestRecallLevel: { type: Number, default: 0 }` (Mongo schemaless, thẻ cũ mặc định 0, KHÔNG migration).

| Nấc | Chứng minh bằng | Màu | Nghĩa |
|---|---|---|---|
| 0 | chưa học (`state='new'`) | trống (track) | mới |
| 1 | **Lật** → "Đã nhớ" | 🟡 vàng | nhận mặt chữ |
| 2 | **Ghép** đúng | 🟢 xanh mờ (giữa vàng-xanh) | nhớ có gợi ý |
| 3 | **Trắc nghiệm** đúng · HOẶC Lật "Quá dễ" | 🟩 xanh lá đậm | phân biệt được |
| 4 | **Tự luận** đúng | 🟣 tím/xanh dương | tự sản sinh |

### 1.2 Cập nhật nấc (trong `applyReviewEvent`, KHÔNG đụng FSRS scheduler)
Payload review sẵn có: `{ mode, rating, practice }`. Map **khi trả lời ĐÚNG**:
- `mode='flip' & rating='good'|'hard'` → 1 (nhớ được, kể cả khó) · `mode='flip' & rating='easy'` ("Quá dễ") → 3 · `rating='again'` → 0 (chưa nhớ)
- `mode='match'` đúng → 2 · `mode='quiz'` đúng → 3 · `mode='type'` đúng → 4

Quy tắc ghi: `bestRecallLevel = max(bestRecallLevel, nấc-vừa-đạt)` — chỉ LÊN, "nấc cao nhất từng đạt".
Trả **sai** → không nâng.
- **Cập nhật nấc BẤT KỂ cờ `practice`** — nấc là trục hiển thị riêng, không phải stability → practice không bơm khống lịch FSRS. Nên "Kiểm tra" (practice=true) vẫn leo màu mà không phá spacing.
- **Lapse** (chỉ khi `practice=false` và rating='again'/quên → FSRS chuyển `relearning`): **reset `bestRecallLevel=1`** (về vàng — chốt của PO). Sai khi `practice=true` (đang Kiểm tra) KHÔNG reset.

### 1.3 Tụt (do đến hạn) — tính lúc đọc, KHÔNG lưu
Nhị phân theo FSRS `due` so với `now`. Đến hạn **KHÔNG rời đoạn màu** — thẻ giữ nguyên nấc màu, chỉ được **tô sọc chéo chồng lên** cùng màu. Nhờ vậy leo nấc ở Kiểm tra thấy màu đổi ngay dù thẻ vẫn đang tụt (tránh cảnh "ghép xong không thấy gì" khi cả bộ đang tới hạn):
- `due > now` (chưa tới hạn) → đoạn màu nấc **đặc**.
- `due ≤ now` (tới/quá hạn) → đoạn màu nấc **có sọc chồng lên** (`slipping` của chính nấc đó).
Thẻ `new` (chưa introduce) nằm ngoài thanh (track trống).
- **Ôn lấp:** chỉ lượt đẩy FSRS (`practice=false`: introduce, Ôn đến hạn) mới đẩy `due` ra tương lai → **hết sọc**. Kiểm tra/Luyện tập (`practice=true`) leo màu (đổi nấc) nhưng KHÔNG bỏ sọc (vẫn tụt tới khi ôn thật) — nhất quán "ôn mới là làm mới".

### 1.4 Vòng đời (từ SRS có sẵn, không đổi state machine FSRS)
`new (0)` --Lật "Đã nhớ"--> `learning, nấc 1` --Kiểm tra/ôn mode khó--> nấc 2/3/4
--đến hạn--> tụt (sọc chồng lên màu) --Ôn đến hạn đúng--> hết sọc --Ôn quên (lapse)--> `relearning, nấc 1`.

### 1.5 Huy hiệu ⭐ "Đã nhớ" (≥21 ngày) — trục riêng, chồng lên màu
- **⭐ = `isMastered` = `state='review' && stability≥21`** (dùng nguyên `flashcard-scheduler.service.js`, KHÔNG field/ngưỡng mới). Đây là **trục thứ 3**, độc lập với nấc màu: đo "nhớ bền qua thời gian", không phải "làm được bài gì".
- Một thẻ có thể tím-chưa-⭐ (gõ được nhưng mới học) hoặc vàng-có-⭐ (nhớ lâu mà chỉ từng lật). Hiển thị: màu = nền, ⭐ = badge overlay.
- **Chúc mừng**: khi 1 thẻ **lần đầu** vượt mốc (review này làm `isMastered` false→true), review response trả cờ `crossedMastery=true` → FE hiện toast "🎉 Đã nhớ từ '…' — qua 21 ngày!". (Bổ sung nhỏ ở `applyReviewEvent`, cùng chỗ cập nhật `bestRecallLevel`.)
- **Sai sau khi ⭐ (lapse)** — chốt "xây lại nhanh theo FSRS", **KHÔNG code thêm**: lapse hạ stability (vẫn >0) → `isMastered` tự về false → ⭐ mất; ôn đúng lại vài lần cách ngày là stability leo về ≥21 → ⭐ lại (nhanh hơn lần đầu). Màu về vàng nhưng leo lại tức thì qua hoạt động. KHÔNG bắt đợi trọn 21 ngày mới, cũng KHÔNG công nhận ⭐ chỉ bằng 1 câu đúng tức thời.

## 2. API contract

### GET /flashcards/decks/:deckId/today
```ts
{
  data: {
    newLimit: number,           // studyPlan.flashcardNewPerDay = cỡ lô mỗi LƯỢT (5|10|20, default 10)
    newRemaining: number,       // min(newLimit, số thẻ state='new' còn lại) — "20 luôn là 20", KHÔNG trừ theo ngày
    newCards: Card[],           // ≤ newRemaining thẻ state='new' (cho "Học từ mới")
    dueCards: Card[],           // getDue(userId,{deckId}) — thẻ đến hạn ôn (mỗi thẻ kèm bestRecallLevel)
    progress: {                 // thanh thang màu — §1
      total: number,
      introduced: number,       // reps≥1, clamp ≤ total (thẻ đã học); track trống = total - introduced
      levels: {                 // mỗi nấc: total (GỒM cả thẻ đang tụt) + slipping (số đang tụt của nấc → vẽ sọc)
        flip:  { total: number, slipping: number },
        match: { total: number, slipping: number },
        quiz:  { total: number, slipping: number },
        type:  { total: number, slipping: number },
      },
      slipping: number,         // tổng thẻ đã học tới/quá hạn (due≤now) — cho chip "Đang tụt"
      mastered: number          // thẻ isMastered (stability≥21) — đếm ⭐, TRỤC RIÊNG (chồng lên màu)
      // Bất biến: Σ levels[k].total == introduced; Σ levels[k].slipping == slipping
    }
  }
}
```
- `Card` = shape thẻ như `getDue`/deck card (front/back/example/audio, cardId). Flashcard không có answerKey → không lộ gì nhạy cảm.
- `dueCards[]` kèm `bestRecallLevel` (số) để UI ôn hiển thị nấc hiện tại. **Bỏ** yêu cầu trả `stability` (bản cũ cần cho mode-by-maturity của funnel — nay user tự chọn mode nên không cần).
- `newCards` rỗng khi bộ hết thẻ state='new' → menu disable "Học từ mới".

### POST /flashcards/cards/:cardId/review  (tái dùng, MỞ RỘNG logic — KHÔNG đổi contract)
Client vẫn gửi `{ mode, rating, practice, roomId? }`. Bổ sung phía server trong `applyReviewEvent`:
- cập nhật `bestRecallLevel` (§1.2), lapse reset (§1.2).
- response thêm `crossedMastery: boolean` — true khi review này làm `isMastered` false→true (FE hiện chúc mừng, §1.5).
- KHÔNG đổi cách FSRS chấm/xếp lịch, KHÔNG đổi cờ `practice` hiện có.
- `rating='easy'` từ Lật "Quá dễ" đi thẳng vào FSRS (stability nhảy cao — thẻ đã biết không phải ôn dài).

### PATCH /users/me (mở rộng `updateMe`)
Ghi `profile.studyPlan.flashcardNewPerDay` (5|10|20). Dùng endpoint profile sẵn có.
- `studyPlan` là subschema strict `default:null`: (1) khai báo `flashcardNewPerDay` trong `StudyPlanSchema`; (2) đọc `studyPlan?.flashcardNewPerDay ?? 10`; (3) ghi null-safe (khởi tạo object nếu null).

### Có sửa (nhỏ):
`awardForReview` — bỏ thưởng XP/quest/streak khi `practice=true`; dayKey → `vnDayKey` (§4). Guest phòng học chung giữ thưởng (guard `!practice || roomId`).

## 3. FE — menu chọn + 4 activity

Trang `/flashcards/[deckId]/study` = **StudyMenu** (không còn orchestrator ép chuỗi):

| Mục menu | Thẻ | Runner | rating/mode | practice | Ghi chú |
|---|---|---|---|---|---|
| **Học từ mới** ({newRemaining}) | `newCards` (≤newLimit thẻ 'new') | **Lật thẻ** (FlipCardRunner) | 3 nút: Chưa thuộc(again)/Đã nhớ(good)/Quá dễ(easy) | false | review đầu → new→learning; Quá dễ→nấc 3 |
| **Ôn đến hạn** ({due}) | `dueCards` | **Lật thẻ** (nhắc lại) | 3 nút như trên | false | lấp tụt (đẩy `due`); quên→reset nấc 1 |
| **Kiểm tra / Luyện tập** | toàn bộ đã học chưa ⭐ (`getDue extra`, trộn) | ModePicker → **Lật(học lại)/Ghép/Trắc nghiệm/Tự luận** | mode chọn | true | leo nấc màu (Lật=ôn nhẹ, không bắt lên nấc), KHÔNG đụng lịch/XP |

- **Menu 3 hành động**, người học tự chọn. "Học từ mới" bấm vào là vào phiên (nút chính = Bắt đầu); new-limit (5/10/20) chỉ đặt cỡ lô.
- **new-limit = cỡ lô/lượt**: `newRemaining = min(newLimit, số thẻ 'new')`. KHÔNG trừ số đã học trong ngày ("20 luôn là 20"). Nhãn control: **"Từ mới/lượt"**.
- **Giữ hồn lật thẻ:** Học mới & Ôn đến hạn = lật + tự chấm 3 mức. "Kiểm tra / Luyện tập" cho chọn cả **Lật (học lại)** một thẻ đã học bất kỳ (chưa due) — chỗ duy nhất ôn lại thẻ chưa tới hạn.
- **Kiểm tra lấy thẻ:** `getDue(deckId, extra:true)` = toàn bộ thẻ đã học **chưa ⭐**, trộn ngẫu nhiên. Mở **thẳng ModePicker** (KHÔNG đi vòng qua màn "thẻ đến hạn").
- **Xóa `StudyRunnerContainer`**: nhánh Kiểm tra/Luyện tập gộp vào `StudyMenu` (ModePicker + StudySessionRunner practice), bỏ màn due-first thừa (đã trùng "Ôn đến hạn").
- Dùng lại 4 runner + ModePicker + StudySessionRunner + SessionSummaryView; menu chỉ quyết THẺ + MODE + cờ practice, mỗi lượt vẫn `POST /cards/:id/review`.
- **Gỡ** `GuidedStudyRunner`/`buildGuidedPlan`/`GuidedTurnRunner`/`GuidedSessionSummary`/`IntroduceCard` (funnel cũ, đã gỡ).

## 4. UI thanh tiến độ (per-bộ)
- **Thanh** = mỗi nấc 1 đoạn màu §1.1; phần `slipping` của nấc **tô sọc chéo chồng lên cùng màu** (KHÔNG tách đoạn xám riêng); phần còn lại (total−introduced) = track trống. Chip 4 nấc dùng `total`.
- Màu: vàng · xanh mờ (giữa vàng-xanh) · xanh lá đậm · tím/xanh dương (nấc 4 có nhũ/hiệu ứng nhẹ — tùy, không bắt buộc). Xanh lá CHỈ hiện khi có thẻ thật ở nấc 3 → hết cảnh "xanh mà chưa thành thạo".
- **Chip** 4 nấc (Lật n · Ghép n · Trắc nghiệm n · Tự luận n) + "Đang tụt n" + **⭐ Đã nhớ n** (`progress.mastered`).
- **Chúc mừng**: khi review trả `crossedMastery=true` → toast "🎉 Đã nhớ từ '…'!" (§1.5). Nấc 4 chỉ khác màu, KHÔNG hiệu ứng (KISS).
- Dòng mục tiêu: "còn X từ mới · Y thẻ đến hạn ôn hôm nay".
- Token màu: `app-amber` (vàng), `app-green` (xanh lá) đã có; **nấc 2** dùng biến pha giữa amber-green, **nấc 4** cần token mới (tím/xanh dương) — thêm 1 token `app-violet` (hoặc dùng `app-accent` nếu là xanh dương). Chốt token ở stream FE, không hex trực tiếp.
- **Phạm vi**: CHỈ per-bộ. Dashboard tổng (`getMeSummary`) giữ nguyên — không đụng.

## 5. Ranh giới stream
| # | Repo | Nội dung | Trạng thái so với bản cũ |
|---|---|---|---|
| 1. Today-session endpoint | exe-api | `GET /decks/:id/today`: newCards/newRemaining/dueCards(+level) + `progress{levels,fading}` | **Sửa nhiều**: bỏ counts/masteryScore, thêm levels/fading/fade |
| 1b. Recall-level tracking | exe-api | field `bestRecallLevel` + cập nhật trong `applyReviewEvent` (climb + lapse reset) | **Mới** |
| 2. new-limit setting | exe-api | `studyPlan.flashcardNewPerDay` qua PATCH /me | **Giữ** (đã xong) |
| 2b. Bỏ thưởng practice | exe-api | `awardForReview` bỏ qua practice; vnDayKey | **Giữ** (đã xong) |
| 3. Study menu + routing | exe-web | `StudyMenu` (Học mới/Ôn/Kiểm tra/Luyện tự do), gỡ funnel | **Thay** stream 3 cũ |
| 4. Lật thẻ 3 nút | exe-web | FlipCardRunner 3 rating (again/good/easy), map Quá dễ | **Sửa** (từ IntroduceCard) |
| 5. Thanh 4 màu + fade | exe-web | thanh nhiều đoạn theo `levels`+`fading`, chip 4 nấc | **Thay** stream 5 cũ (thanh 2 lớp) |
| 6. new-limit chọn-rồi-Bắt-đầu | exe-web | control 5/10/20 chỉ set pref + nút "Học từ mới (N)" | **Sửa** (thêm bước Bắt đầu) |

## 6. Rủi ro
- **Không bơm khống FSRS**: `bestRecallLevel` là trục riêng; practice=true chỉ leo màu, KHÔNG advance stability/đổi `due`. Chỉ introduce + Ôn đến hạn = practice=false.
- **Lapse reset về nấc 1**: chỉ áp khi practice=false & FSRS thật sự lapse (relearning). Kiểm tra sai không phạt.
- **Tụt phụ thuộc `due`**: nhị phân `due≤now` → sọc chồng lên màu nấc (không rời đoạn, không mờ opacity); chưa introduce thì bỏ qua.
- **Token màu nấc 2/4**: không hex trực tiếp — thêm biến vào theme (stream FE).
- **Gỡ funnel**: xóa 4 file guided + test tương ứng; đảm bảo 4 runner .test cũ vẫn xanh (chỉ bọc ngoài).
- **studyPlan null/strict · counts→levels · vnDayKey**: như §1-§2.

## 7. Chốt (PO 26/08)
- Menu chọn cách học (Lật/Ghép/Kiểm tra/Luyện tự do) — KHÔNG funnel ép.
- Lật 3 mức: Chưa thuộc / Đã nhớ / Quá dễ(→easy, nhảy nấc 3).
- Màu = nấc năng lực: Lật vàng · Ghép xanh mờ · Trắc nghiệm xanh lá · Tự luận tím.
- Tụt theo lịch `due`: chưa tới hạn = màu đặc · đến hạn = **sọc chồng lên màu nấc** (giữ nấc, không mờ dần). Lapse → về nấc 1.
- Ôn = làm mới (hết mờ); Kiểm tra = leo nấc. Đi theo ngày VN.
- new-limit: chọn rồi bấm Bắt đầu.

### Câu hỏi mở — đã chốt hết (26/08)
- ~~Kiểm tra chọn thẻ nào?~~ **Chốt**: toàn bộ thẻ đã học, trộn ngẫu nhiên — §3.
- ~~Nấc 4 hiệu ứng?~~ **Chốt**: chỉ khác màu (KISS) — §4.
- ~~Sai sau ⭐ lấy lại thế nào?~~ **Chốt**: xây lại nhanh theo FSRS, không code thêm — §1.5.
- ~~⭐ mốc 21 ngày?~~ **Chốt**: badge `isMastered` trục riêng + chúc mừng khi vượt mốc — §1.5.
