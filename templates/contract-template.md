<!--
  contract-template.md — dùng khi design.md cần 1 contract API rõ ràng: nội bộ module, cross-service
  (api↔llm, api↔cat), hoặc cross-repo (api↔web, api↔admin, api↔mobile). Một file = một contract/endpoint
  nhóm liên quan. Đặt trong specs/<feature-slug>/contracts/.
-->
# Contract: <Tên>

- **Loại:** <Nội bộ module / Cross-service (api↔llm|cat) / Cross-repo (api↔web|admin|mobile)>
- **Bên cung cấp (provider):** <repo/service>
- **Bên tiêu thụ (consumer):** <repo/service — liệt kê tất cả nếu nhiều bên dùng>
- **Trạng thái:** <Nháp / Đã thống nhất 2 bên / Đang triển khai / Live>

## Endpoint

```
<METHOD> <path>
```

- **Auth:** <JWT user / X-Service-Key / public — theo <API_REPO>/docs/ARCHITECTURE.md §Trust boundary>
- **Permission required:** <`<resource>:<action>`, hoặc "không">

## Request

```json
{
  "<field>": "<kiểu — mô tả>"
}
```

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|

## Response — thành công

```json
{
  "<field>": "<kiểu>"
}
```

| Field | Kiểu | Ghi chú |
|---|---|---|

## Response — lỗi

| Mã HTTP | error code | Khi nào |
|---|---|---|

## Versioning / breaking change

<Nếu đây là thay đổi lên contract đã live: mô tả cụ thể cái gì breaking, kế hoạch migration, có cần version
field/route mới không. Nếu là contract mới hoàn toàn, ghi "N/A — contract mới".>

## Sequencing triển khai (nếu cross-repo)

<Thứ tự deploy provider trước consumer — tham chiếu .ai/workflows/release-workflow.md §3.>

## Ví dụ gọi thực tế

```bash
curl -X <METHOD> <url> \
  -H "..." \
  -d '...'
```
