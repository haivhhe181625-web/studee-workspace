---
feature: luyen-de-difficulty-assembly
status: draft
created: 2026-08-18
repos: [exe-api]
---

# Tasks — Bật độ khó (irt.b) thành filter trong assemble (Bước 3/3)

> Phụ thuộc **Bước 1** (khung DB) + **Bước 2** (bank đủ dư). Làm sau cùng.

## T1 — Dải b theo section

**File:** `src/modules/test-paper/paper-blueprint.model.js` · `src/modules/test-paper/paper-blueprint.js`
(helper map tier→b)

- [ ] Thêm field optional `sections[].irtBand { min, max }` (default null).
- [ ] Helper `sectionBRange(section)`: nếu `irtBand` có → dùng; else map `difficultyTier`→dải mặc định
  (easy: ≤-0.5, mid: -0.5..0.5, hard: ≥0.5); tier vắng → null (không lọc b).
- [ ] Test helper: 3 tier ra 3 dải; irtBand override; tier vắng → null.

## T2 — assemble `$match` b + fallback

**File:** `src/modules/test-paper/test-paper.service.js` · test kèm

- [ ] Trong loop section của assemble: nếu `sectionBRange` không null → thêm `'irt.b': { $gte:min, $lte:max }`
  vào `$match` (giữ nguyên cefr/itemTypes/targetGoal/status/$nin).
- [ ] **Fallback**: nếu số câu lấy được < itemCount → chạy lại section đó **bỏ** ràng buộc b (giữ phần còn
  lại), đánh dấu section "nới b"; assemble return kèm cờ nới (để admin/report biết).
- [ ] Giữ `$sample` random trong tập đã lọc.
- [ ] Test: bank dư có b phân tầng → section hard chỉ lấy b≥0.5 (AC1); bank thiếu trong dải → nới + cờ (AC2);
  section không có dải b → như cũ (AC3); assemble lại ra bộ khác (AC4).

**Verify:** `npx jest test-paper` → xanh (test cũ không cấu hình b giữ nguyên).

## Definition of done

- [ ] test-paper full xanh; assemble lọc theo b khi cấu hình, fallback an toàn khi thiếu, backward compat khi không cấu hình.

## Câu hỏi mở

- Ngưỡng dải b theo tier + ngưỡng "phải nới" (spec §Câu hỏi mở).
