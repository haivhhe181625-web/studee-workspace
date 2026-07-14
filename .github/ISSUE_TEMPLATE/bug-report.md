---
name: Bug report
about: Báo lỗi hành vi hiện tại không đúng như thiết kế
title: "[Bug] "
labels: bug
assignees: ""
---

## Hiện tượng

<Mô tả cụ thể cái gì sai. Tránh mô tả chung chung ("không hoạt động") — nêu input/hành động cụ thể.>

## Các bước tái hiện

1. <Bước 1>
2. <Bước 2>

## Kết quả mong đợi

<Hành vi đúng phải là gì.>

## Kết quả thực tế

<Hành vi sai thực tế — kèm mã lỗi HTTP/error code nếu có, log nếu có.>

## Môi trường

- [ ] Local
- [ ] Dev (`dev.studeeteam.cloud`)
- [ ] Production

## Service liên quan

- [ ] `<API_REPO>/services/api`
- [ ] `<API_REPO>/services/cat`
- [ ] `<API_REPO>/services/llm`
- [ ] `exe-web`
- [ ] `exe-admin`
- [ ] Chưa rõ

---

*Bug rõ nguyên nhân, sửa nhỏ → không cần qua pipeline spec đầy đủ (xem
`.ai/checklists/techlead-checklist.md` §When to skip the full pipeline). Bug cần điều tra sâu hoặc sửa lớn → tạo
`specs/<feature-slug>/` như một feature.*
