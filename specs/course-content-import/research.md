# Research: Đảm bảo ghi "toàn-hoặc-không" (IS-5) cho cấu trúc khóa học nhiều tầng trên MongoDB

- **Spec:** `specs/course-content-import/spec.md`
- **Ngày:** 2026-07-15
- **Trạng thái:** Đã có kết luận

## Câu hỏi

Khi Import lưu một cấu trúc khóa học nhiều tầng (Lộ trình → Chặng → Chuyên đề → Bài học), làm sao đảm bảo
**all-or-nothing** (IS-5 / AC-4): hoặc toàn bộ cây được ghi, hoặc không ghi bản ghi nào — trên đúng hạ tầng
MongoDB mà `exe-api` đang chạy?

Cụ thể, checkable:

1. Deployment MongoDB hiện tại có phải replica set (điều kiện bắt buộc để dùng multi-document transaction của
   Mongoose `session.withTransaction`) không?
2. Nếu KHÔNG, codebase đang xử lý các thao tác ghi nhiều bản ghi cần nguyên tử như thế nào?
3. Cách lưu nào cho cấu trúc phân tầng đảm bảo nguyên tử **không phụ thuộc** vào cấu hình replica set?

## Vì sao cần trả lời trước khi design

Quyết định **data model** (§3 của `design.md`) phụ thuộc trực tiếp vào câu trả lời:

- Nếu ghi nhiều collection (mỗi tầng 1 collection, có `ObjectId` ref cha–con) thì buộc phải có transaction để
  không tạo dữ liệu rác một phần → phụ thuộc replica set.
- Nếu MongoDB là standalone (không replica set) thì transaction không dùng được, và mọi cách ghi tuần tự nhiều
  bản ghi đều có nguy cơ ghi dở → **vi phạm IS-5**.

Đây là ràng buộc §8 mà BA đã nêu ("MongoDB không có transaction mặc định giữa nhiều collection nếu không cấu
hình") — cần chốt bằng bằng chứng thực tế trong code, không đoán.

## Phương pháp điều tra

- [x] Đọc `docker-compose.yml` (dev) + `docker-compose.prod.yml` để xem image/cấu hình MongoDB.
- [x] Kiểm tra `MONGODB_URI` trong `services/api/.env` có tham số `replicaSet` / `directConnection` không.
- [x] Grep toàn bộ `src/` tìm chỗ đã dùng `startSession` / `withTransaction`.
- [x] Đọc cách module dùng transaction (`payment.service.js`) xử lý khi transaction không được hỗ trợ.
- [x] Đọc tiền lệ lưu cấu trúc phân tầng đã có: `roadmap.model.js` (template phases nhúng) để đối chiếu.

## Phát hiện

**1. MongoDB dev là standalone, không replica set.**
`exe-api/docker-compose.yml` khai báo:

```yaml
mongodb:
  image: mongo:7
  container_name: exe_db
  # KHÔNG có command: --replSet ...  → mongod chạy standalone
```

`MONGODB_URI` trong `services/api/.env` không có tham số `replicaSet` (đã kiểm tra: "no replicaSet/directConnection
param in URI"). `config/db.js` connect thẳng, không set replica options.

**2. Codebase ĐÃ biết MongoDB có thể standalone và có sẵn fallback non-atomic.**
`src/modules/payment/payment.service.js` là chỗ duy nhất dùng transaction:

```js
const session = await mongoose.startSession();
try {
  await session.withTransaction(async () => {
    await order.save({ session });
    await Transaction.create([txnDoc], { session });
  });
} catch (err) {
  if (_isNoTransactionSupport(err)) {
    // Standalone MongoDB — redo writes without a session (non-atomic).
    logger.warn({ orderCode }, 'MongoDB has no transaction support — writing without a session');
    await order.save();
    await Transaction.create(txnDoc);
  } else { throw err; }
} finally { await session.endSession(); }
```

với helper:

```js
function _isNoTransactionSupport(err) {
  const msg = err && err.message ? String(err.message) : '';
  return (
    err?.code === 20 ||
    /Transaction numbers are only allowed on a replica set/i.test(msg) ||
    /Transactions are not supported/i.test(msg)
  );
}
```

→ Với payment (chỉ 2 bản ghi, đã có idempotency guard riêng), fallback ghi tuần tự **không nguyên tử** là chấp
nhận được. Nhưng với Import (một cây nhiều tầng, hàng chục–hàng trăm bản ghi), fallback này để lại **dữ liệu rác
một phần** khi lỗi giữa chừng → **vi phạm trực tiếp IS-5 / AC-4**. Vậy: **không được dựa vào transaction** cho
feature này (không được coi replica set là điều kiện đã có).

**3. Ghi 1 document là nguyên tử trên MỌI cấu hình MongoDB.**
MongoDB đảm bảo thao tác ghi trên **một single document** (kể cả document có mảng/sub-document lồng nhau) là
nguyên tử — không cần transaction, không cần replica set (tài liệu MongoDB: "a write operation is atomic on the
level of a single document"). `insertOne` một document lồng cả cây hoặc thất bại toàn bộ, không có trạng thái ghi
dở nửa document.

**4. Đã có tiền lệ lưu cấu trúc phân tầng dạng nhúng trong repo.**
`roadmap.model.js`: `RoadmapTemplateSchema.phases = [TemplatePhaseSchema]` — các tầng con được **nhúng
(embedded sub-document)** trong 1 document, không tách collection riêng, không dùng ref `ObjectId` cha–con. Đây
đúng là mẫu để noi theo cho cấu trúc Lộ trình → Chặng → Chuyên đề → Bài học.

## Kết luận

**Lưu toàn bộ cây cấu trúc một khóa học thành MỘT document nhúng (embedded tree) trong một collection mới
(`course_structures`), ghi bằng một lần `create()` / `insertOne` duy nhất.**

- Thao tác ghi 1 document là nguyên tử trên standalone lẫn replica set → **đáp ứng IS-5 / AC-4 mà không phụ thuộc
  transaction**, không cần đổi hạ tầng, không cần fallback non-atomic như payment.
- Validate toàn bộ file **trước** khi build document; chỉ khi 0 lỗi mới `create()`. Không ghi thăm dò từng tầng.
- Ảnh hưởng tới `design.md`:
  - **§3 Data model / `data-model.md`:** chọn embedded-tree một collection thay vì 5 collection có ref cha–con.
  - **§2 Quyết định kiến trúc:** ghi rõ lựa chọn "single embedded document" + lý do nguyên tử.
  - **§9 Rủi ro:** kèm giới hạn 16MB BSON (xem "Rủi ro còn lại").

## Rủi ro còn lại

- **Giới hạn 16MB/BSON document.** Một khóa học cực lớn (hàng nghìn Bài học) về lý thuyết có thể chạm trần. Với
  Phase 1 (khóa học tĩnh, quy mô một chương trình học thật), 16MB là rất rộng — ước lượng thô mỗi Bài học vài trăm
  byte metadata → cỡ chục nghìn Bài học mới tới hạn. **Giảm thiểu:** validate thêm một ngưỡng số lượng
  tầng/bài học ở lớp parse và trả lỗi rõ ràng thay vì để Mongo ném lỗi ghi (ngưỡng cụ thể **[CHỜ Q6]** —
  giới hạn kích thước/số dòng file). Ghi rõ trần này trong `design.md` §9 để spec Phase 2 biết mô hình đọc.
- **Truy vấn con sâu trong tương lai.** Nếu Phase 2/3 cần query/aggregate xuyên tầng ở quy mô lớn (vd "mọi Bài học
  dùng exercise X"), embedded tree kém linh hoạt hơn collection tách. Đây là đánh đổi có ý thức cho Phase 1 (ghi
  nguyên tử + đọc nguyên khối là ưu tiên); nếu Phase 2 phát sinh nhu cầu query xuyên tầng nặng, cân nhắc thêm
  index phụ hoặc projection, chưa cần tách collection ở Phase 1.
