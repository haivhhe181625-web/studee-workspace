---
feature: lms-parent-student-portal-api
role: tech-lead → developer
status: draft
date: 2026-08-22
test_runner: cd exe-api/services/api && npx jest <path>
---

# Tasks — parent/student portal API + audit (M10-13→17)

> Chốt Q1 (prefix /api/parent vs /api/lms) TRƯỚC — chặn nhóm C/D. Nhiều bề mặt đã có
> khung (M10-2 nền), phần lớn là thêm endpoint + rule ownership.

## Nhóm A — Tài khoản phụ huynh (M10-13, PRD-222)
- [ ] **A1. `parent.service` — createParent + linkChildren.** Ràng buộc: con cùng org +
  `student`; O4 (1 con ≤1 phụ huynh) → 409; ≤20 con. Index đỡ O4 (unique trên child ref).
  Test: con đã có phụ huynh → 409; con 21 → chặn; con org khác → chặn.
  · verify: `npx jest src/modules/user/__tests__/parent-link.test.js`
- [ ] **A2. Routes `POST /api/admin/parents` + `/:id/children` + mount.** owner/manager.
  Integration khớp FE `parent.service.ts`.
  · verify: `npx jest src/modules/user/__tests__/parent.routes.test.js`

## Nhóm B — OTP phụ huynh (M10-14, PRD-223)
- [ ] **B1. Mở rộng `email-otp` — kênh trung lập, purpose=parent-login.** TTL/max-attempt/
  resend-cooldown (chốt số Q3). Model OTP tách kênh (email|phone) để cắm phone sau.
  Test: hết hạn/sai quá số lần → từ chối; thêm phone không đổi schema.
  · verify: `npx jest src/modules/auth/__tests__/parent-otp.test.js`
- [ ] **B2. Routes `POST /api/parent/otp/{request,verify}` + mount** (theo Q1 prefix).
  Không tồn tại đường đặt mật khẩu cho parent. Integration.
  · verify: `npx jest src/modules/auth/__tests__/parent-otp.routes.test.js`

## Nhóm C — Portal phụ huynh đọc (M10-15, PRD-224)
- [ ] **C1. `GET /parent/children/:id/overview`** (bổ sung; `/children` + `/classes` đã có).
  `assertGuardianOf` trước khi trả. Gộp hồ sơ + classes (Q2 shape). Test: id con nhà
  khác → 403/404, không rò.
  · verify: `npx jest src/lms-api/__tests__/parent-overview.test.js`
- [ ] **C2. Xác nhận `/api/parent/*` không có route ghi nào** (denyGuardianWrites + không
  mount POST/PATCH). Test đối kháng: cố gọi ghi → chặn.
  · verify: `npx jest src/__tests__/surface-guard.test.js`

## Nhóm D — Khu học viên (M10-16, PRD-225)
- [ ] **D1. Chi tiết lớp cho học viên** (bổ sung; `/classes/mine` đã có). Ép studentId
  từ token; trả giáo viên + giáo trình. Test: truyền studentId khác → bỏ qua; không đọc
  bạn cùng lớp.
  · verify: `npx jest src/lms-api/__tests__/student-class-detail.test.js`

## Nhóm E — Audit (M10-17, PRD-226, làm sau cùng, ưu tiên thấp)
- [ ] **E1. Mở rộng `auditLog` phủ 4 nhóm thao tác** (tạo/vô hiệu tài khoản, đổi vai trò,
  link/unlink phụ huynh, reset mật khẩu) + log bypassTenant. Test: mỗi thao tác sinh log
  truy được theo org + actor.
  · verify: `npx jest src/__tests__/audit-sensitive-ops.test.js`

## Prefix decision (Q1) — chặn C/D
```
FE gọi /api/parent/* , /api/classes/mine
BE ở   /api/lms/parent/* , /api/lms/classes/mine
→ chọn: (a) sửa FE sang /lms (khuyến nghị) | (b) BE thêm alias mount
```

## Kiểm tra tổng
- [ ] `npm test` xanh · [ ] parent read-only (test cấp nhầm quyền ghi vẫn chặn) · [ ] student ép token.
