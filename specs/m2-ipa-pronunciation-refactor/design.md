# M2 IPA — Design note

Chỉ ghi phần thiết kế **non-obvious**: nơi lưu Mastery + máy trạng thái SRS 2 nhịp. Phần còn lại theo spec Plane v1.3.

---

## Quyết định T0.1 — Nơi lưu Mastery: **mở rộng `IpaProgress` + tái dùng `IpaSrsQueue`**

**Chốt:** KHÔNG tạo model `IpaMastery` mới (tên spec đặt). Tái dùng 2 model đã tồn tại.

### Vì sao
- `IpaProgress` (`ipa_progress`) **đã là** bản ghi mastery per (user×lesson): `status` enum **đã có `mastered`**, unique index `(userId, lessonId)`, đã có `overallScore`. Tạo `IpaMastery` cùng khóa = trùng lặp, buộc dual-write + migration kép.
- `IpaSrsQueue` (`ipa_srs_queue`) **đã là** lịch ôn per (user×lesson): có `nextReviewDate`, `mastered`, unique `(userId, lessonId)`. Field spec `srsDueAt` ≡ `nextReviewDate` sẵn có — không cần đẻ field mới ở model mới.

→ DRY: mọi field spec "IpaMastery" ánh xạ vào 2 model đang chạy.

### Ánh xạ field spec (v1.3 §10.1) → nơi lưu thật

| Field spec ("IpaMastery") | Nơi lưu thật | Ghi chú |
|---|---|---|
| `state (learning/mastered)` | **`IpaProgress.status`** | tái dùng: `learning` ≡ `status='completed'` (đã đạt đk, chờ SRS); `mastered` ≡ `status='mastered'`. **Không thêm field `state` riêng.** |
| `needsReview` (bool) | **`IpaProgress.needsReview`** (thêm mới) | FR-008: trượt SRS → gắn cờ, không thu hồi |
| `srsDueAt` | **`IpaSrsQueue.nextReviewDate`** (sẵn có) | hạn ôn nhịp kế |

---

## Thay đổi schema

### `IpaProgress` (`ipa-progress.model.js`)
- **Thêm** `needsReview: { type: Boolean, default: false }`.
- **Nghỉ hưu** `masteredStreak` (luật 3×90 cũ bị thay): ngừng ghi; **giữ cột trong cửa sổ migration** để đọc dữ liệu cũ, gỡ sau.
- `status`: mang nghĩa mới — `completed` = đã qua 80% + sàn câu + cổng cảm âm (learning, bài mở), `mastered` = qua 2 nhịp SRS. `warmUpConfidence` bỏ dùng (warmup đã cắt), giữ cột cho dữ liệu cũ.
- `screenScores`: chỉ còn `m5` (câu) và điểm tổng còn ý nghĩa cho ngưỡng; m2/m3/m4/m6 dần vô nghĩa theo flow 3 màn — **không xóa** (surgical), chỉ ngừng phụ thuộc.

### `IpaSrsQueue` (`ipa-srs-queue.model.js`) — chuyển từ SM-2 sang **2 nhịp cố định**
- **Thêm** `reviewStage: { type: Number, default: 0 }` — `0`=chưa vào ôn, `1`=đã qua nhịp 1 (1 ngày), `2`=graduated (đã qua 7 ngày → mastered).
- `nextReviewDate` = hạn nhịp kế: đạt đk → `+1 ngày`; qua nhịp 1 → `+7 ngày`; qua nhịp 2 → `null` + `mastered=true`.
- `intervalDays`: set `1` rồi `7` (khớp reviewStage).
- `easeFactor`: **không dùng** cho lịch cố định (để mặc định 2.5, bỏ qua). Không xóa (surgical).
- `mastered` (sẵn có): set `true` khi `reviewStage=2`; đồng bộ với `IpaProgress.status='mastered'`.

---

## Máy trạng thái (FR-007 + FR-008)

```
[đạt: tổng≥80 & câu≥70 & cổng cảm âm≥3/4 (trừ template D)]
      → IpaProgress.status = completed (learning); mở bài kế
      → IpaSrsQueue: reviewStage=0→ hẹn nextReviewDate=+1d, intervalDays=1
Ôn nhịp 1 (sau 1 ngày, vòng rút gọn, bỏ màn Học):
      đạt  → reviewStage=1, nextReviewDate=+7d, intervalDays=7
      trượt→ IpaProgress.needsReview=true; giữ status=completed; hẹn ôn gần; KHÔNG thu hồi
Ôn nhịp 2 (sau 7 ngày):
      đạt  → reviewStage=2, mastered=true, IpaProgress.status=mastered, nextReviewDate=null
      trượt→ needsReview=true; giữ nguyên; hẹn lại
```

---

## Chạm code (định hướng, không phải diff)
- `ipa.service.js` `computeLessonResult`: bỏ `masteredStreak`; áp 80% + sàn câu (`app-setting`) + cổng cảm âm; set `IpaProgress.status`.
- `ipa-srs.service.js`: thay map SM-2 bằng 2 nhịp cố định + `reviewStage`.
- Migration (FR-012): cũ `masteredStreak` đạt luật 3×90 → `status='mastered'` + `IpaSrsQueue.reviewStage=2, mastered=true`.

## Sub-quyết định đã khóa
- Không thêm field `state` riêng — dùng `IpaProgress.status`.
- SRS dùng nhịp **cố định** (1d/7d), bỏ ease-factor SM-2 (KISS, YAGNI).
- Giữ (không xóa) các cột cũ vô nghĩa dần (`masteredStreak`, `warmUpConfidence`, `easeFactor`, screenScores m2/m3/m4/m6) trong cửa sổ migration — surgical, gỡ ở dọn dẹp sau.

## Câu hỏi mở
Không còn cho T0.1. (Ngưỡng cảm âm 3/4 và ngưỡng 80/70 đã chốt ở v1.3.)
