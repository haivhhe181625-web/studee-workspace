# Tasks: Import cấu trúc Khóa học tĩnh (Admin Content Ingestion — Phase 1)

> **Cho Developer Agent:** implement theo đúng thứ tự Task 1 → 8. Mỗi Task độc lập test được, commit riêng.
> Dùng `.ai/prompts/implement-task.md` cho từng task.

**Design:** `specs/course-content-import/design.md` (đã duyệt Technical Review 2026-07-15)
**Acceptance:** `specs/course-content-import/acceptance.md`
**Lưu ý chung khi chạy lệnh:** mọi lệnh `npm`/`npx jest` chạy trong thư mục `services/api` của repo `exe-api`
(`cd services/api`). Test đặt trong `src/__tests__/` (repo dùng 1 folder test tập trung, không co-located). Chạy
1 file: `npx jest <pattern> -i` (`-i` = `--runInBand`, khớp `npm test`).

**Quy ước cột file Excel Phase 1** (template chốt với đội học thuật; parser Task 3 bám theo):

| Cột | Tên | Ý nghĩa |
|---|---|---|
| A | `Level` | `ROADMAP` / `PHASE` / `MODULE` / `LESSON` / `EXERCISE` |
| B | `Key` | định danh: `slug` (ROADMAP) hoặc `key` (PHASE/MODULE/LESSON); bỏ trống ở EXERCISE |
| C | `Title` | tên tầng |
| D | `Type` | chỉ EXERCISE: `ipa` / `talk` (`quiz` **chưa hỗ trợ** — Phase 2) |
| E | `RefId` | chỉ EXERCISE: mã bài tập đã tồn tại (IPA = `ipa_lessons.code`; talk = scenario id) |

`order` của mỗi node suy ra từ thứ tự dòng trong cùng cha. Dòng 1 là header (bỏ qua). Số dòng báo lỗi = số dòng
Excel thật (1-based).

> **Ghi chú kiến trúc (đã chốt khi breakdown):** endpoint Import được **hand-mount** trong `_router.js` (không dùng
> `actions` của `_crud.factory.js`) vì cần chèn middleware `multer` để nhận `multipart/form-data` — chain factory
> hiện không cho chèn middleware giữa `verifyPermission` và handler. Đây đúng tiền lệ các route admin hand-mounted
> (`/staff`, `/reviews` trong `_router.js`): vẫn `verifyToken → verifyPermission → delegate service → auditLog thủ
> công`. **List/read khóa đã import bỏ khỏi Phase 1** (không AC nào cần duyệt/browse khóa — đó là Phase 2 tiêu
> thụ); chỉ build endpoint Import.

---

## File Structure

| File | Trách nhiệm | Tạo/Sửa |
|---|---|---|
| `services/api/package.json` | Thêm dependency `exceljs` | Sửa |
| `services/api/src/constants/error-codes.js` | Thêm `PARSE_FAILED`, `EMPTY_COURSE_FILE`, `IMPORT_VALIDATION_FAILED`, `UNSUPPORTED_EXERCISE_TYPE`, `INVALID_FILE_TYPE` | Sửa |
| `services/api/src/constants/permissions.js` | Thêm `COURSECONTENT_READ/WRITE/MANAGE` (KHÔNG vào `CENTER_PERMISSIONS`) | Sửa |
| `services/api/src/modules/course-content/course-content.model.js` | Schema `CourseStructure` + cây nhúng + index | Tạo |
| `services/api/src/modules/course-content/course-content.parser.js` | `parseXlsx(buffer)` → `{ course, errors }` (IR + số dòng) | Tạo |
| `services/api/src/modules/course-content/course-content.references.js` | `resolveReferences(course)` — validate IPA published / talk / quiz | Tạo |
| `services/api/src/modules/course-content/course-content.service.js` | `validateCourse`, `importCourse` — parse→validate→build→ghi | Tạo |
| `services/api/src/middlewares/course-import.upload.js` | multer `.xlsx` ≤5MB, map lỗi → ApiError | Tạo |
| `services/api/src/admin-api/resources/course-content.admin.js` | Router hand-mount `POST /_/import` (delegate service + audit) | Tạo |
| `services/api/src/admin-api/_router.js` | `router.use('/course-imports', ...)` | Sửa |
| `services/api/src/__tests__/course-content.model.test.js` | Unit model (validate + unique slug) | Tạo |
| `services/api/src/__tests__/course-content.parser.test.js` | Unit parser (IR, PARSE/EMPTY/ORPHAN) | Tạo |
| `services/api/src/__tests__/course-content.references.test.js` | Unit resolver (ipa/talk/quiz) | Tạo |
| `services/api/src/__tests__/course-content.service.test.js` | Unit service (validate + importCourse + all-or-nothing) | Tạo |
| `services/api/src/__tests__/course-content.import.api.test.js` | API test (403/200/audit qua supertest) | Tạo |
| `exe-admin/src/services/course-content.service.ts` | Gọi `POST /admin/course-imports/_/import` (multipart) | Tạo |
| `exe-admin/src/app/(admin)/course-import/page.tsx` | Trang Import (upload + summary/lỗi) | Tạo |

---

## Task 1: Cài dependency + error codes + permissions

**Files:**
- Modify: `services/api/package.json`
- Modify: `services/api/src/constants/error-codes.js`
- Modify: `services/api/src/constants/permissions.js`

- [ ] **Step 1: Cài exceljs**

Run:
```bash
cd services/api && npm install exceljs
```
Expected: `package.json` `dependencies` có `exceljs`; lệnh exit 0.

- [ ] **Step 2: Thêm error codes**

Trong `services/api/src/constants/error-codes.js`, thêm một nhóm mới trước dấu đóng `})`:

```js
  // ── Course content import (Phase 1) ──────────────────────────────────────
  PARSE_FAILED:              'PARSE_FAILED',              // file .xlsx hỏng/không đọc được
  INVALID_FILE_TYPE:         'INVALID_FILE_TYPE',         // không phải .xlsx
  EMPTY_COURSE_FILE:         'EMPTY_COURSE_FILE',         // parse được nhưng không có tầng nào
  IMPORT_VALIDATION_FAILED:  'IMPORT_VALIDATION_FAILED',  // lỗi cấu trúc/tham chiếu/trùng — kèm errors[]
  UNSUPPORTED_EXERCISE_TYPE: 'UNSUPPORTED_EXERCISE_TYPE', // type quiz (hoãn Phase 2)
```
(`FILE_TOO_LARGE` đã tồn tại — tái dùng.)

- [ ] **Step 3: Thêm permissions**

Trong `services/api/src/constants/permissions.js`, thêm vào object `PERMISSIONS` (sau nhóm `IPA_MANAGE`):

```js
  // Course content (static course structure import — content ingestion Phase 1).
  // Platform-only (Q5): KHÔNG thêm vào CENTER_PERMISSIONS — nội dung dùng chung.
  COURSECONTENT_READ: 'coursecontent:read',
  COURSECONTENT_WRITE: 'coursecontent:write',
  COURSECONTENT_MANAGE: 'coursecontent:manage',
```

- [ ] **Step 4: Smoke check — constants load**

Run:
```bash
cd services/api && node -e "const p=require('./src/constants/permissions').PERMISSIONS; const c=require('./src/constants/error-codes'); console.log(p.COURSECONTENT_WRITE, c.IMPORT_VALIDATION_FAILED)"
```
Expected: in `coursecontent:write IMPORT_VALIDATION_FAILED`.

- [ ] **Step 5: Commit**

```bash
cd services/api && git add package.json package-lock.json src/constants/error-codes.js src/constants/permissions.js
git commit -m "chore(course-content): add exceljs dep, import error codes and coursecontent permissions"
```

---

## Task 2: Model `CourseStructure` (cây nhúng + unique slug)

**Files:**
- Create: `services/api/src/modules/course-content/course-content.model.js`
- Test: `services/api/src/__tests__/course-content.model.test.js`

- [ ] **Step 1: Viết test thất bại**

Tạo `services/api/src/__tests__/course-content.model.test.js`:

```js
'use strict';

/** Unit test model CourseStructure: validate bắt buộc + unique slug. In-memory Mongo. */
jest.mock('../config/logger', () => ({ info: jest.fn(), warn: jest.fn(), error: jest.fn(), debug: jest.fn() }));

const mongoose = require('mongoose');
const { connectDb, clearDb, disconnectDb } = require('./helpers/db');
const { CourseStructure } = require('../modules/course-content/course-content.model');

const oid = () => new mongoose.Types.ObjectId();

const validDoc = (extra = {}) => ({
  slug: 'khoa-a2',
  title: 'Khóa A2',
  status: 'ready',
  importedBy: oid(),
  phases: [{
    key: 'p1', title: 'Chặng 1', order: 1, cefrFrom: 'A2', cefrTo: 'B1', goalNote: 'IELTS 5.0→6.0',
    modules: [{
      key: 'm1', title: 'Ngữ pháp', order: 1, category: 'grammar',
      lessons: [{
        key: 'l1', title: 'Bài 1', order: 1,
        theory: '# Thì hiện tại', videoUrl: 'https://cdn.x/v.mp4', audioUrl: 'https://cdn.x/a.mp3',
        exercises: [{ type: 'ipa', refId: 'L1', order: 1 }],
      }],
    }],
  }],
  ...extra,
});

beforeAll(async () => { await connectDb(); await CourseStructure.init(); });
afterEach(clearDb);
afterAll(disconnectDb);

test('tạo document hợp lệ với cây nhúng + field Hướng B', async () => {
  const doc = await CourseStructure.create(validDoc());
  expect(doc.slug).toBe('khoa-a2');
  expect(doc.centerId).toBeNull();
  expect(doc.status).toBe('ready');
  expect(doc.phases[0].cefrFrom).toBe('A2');
  expect(doc.phases[0].cefrTo).toBe('B1');
  expect(doc.phases[0].modules[0].category).toBe('grammar');
  const l = doc.phases[0].modules[0].lessons[0];
  expect(l.theory).toContain('Thì hiện tại');
  expect(l.videoUrl).toBe('https://cdn.x/v.mp4');
  expect(l.exercises[0].type).toBe('ipa');
});

test('Bài học chỉ lý thuyết (không exercises) — model chấp nhận', async () => {
  const d = validDoc();
  d.phases[0].modules[0].lessons[0].exercises = [];
  const doc = await CourseStructure.create(d);
  expect(doc.phases[0].modules[0].lessons[0].exercises).toHaveLength(0);
});

test('category ngoài enum → ValidationError', async () => {
  const d = validDoc();
  d.phases[0].modules[0].category = 'unknown-cat';
  await expect(CourseStructure.create(d)).rejects.toThrow(/validation/i);
});

test('thiếu title → ValidationError', async () => {
  await expect(CourseStructure.create(validDoc({ title: undefined }))).rejects.toThrow(/validation/i);
});

test('type ngoài enum [ipa,talk] → ValidationError', async () => {
  const bad = validDoc();
  bad.phases[0].modules[0].lessons[0].exercises[0].type = 'quiz';
  await expect(CourseStructure.create(bad)).rejects.toThrow(/validation/i);
});

test('trùng slug → duplicate key error (E11000)', async () => {
  await CourseStructure.create(validDoc());
  await expect(CourseStructure.create(validDoc({ title: 'Khác' }))).rejects.toMatchObject({ code: 11000 });
});
```

- [ ] **Step 2: Chạy test — xác nhận FAIL**

Run:
```bash
cd services/api && npx jest course-content.model -i
```
Expected: FAIL — `Cannot find module '../modules/course-content/course-content.model'`.

- [ ] **Step 3: Tạo model**

Tạo `services/api/src/modules/course-content/course-content.model.js`:

```js
/**
 * CourseStructure — a static course tree imported by the academic team (Phase 1
 * content ingestion). ONE document = ONE course, whole hierarchy embedded
 * (Roadmap → Phase → Module → Lesson → exercise refs) so the write is a single
 * atomic insert (all-or-nothing / IS-5) on standalone MongoDB — see
 * specs/course-content-import/research.md. Exercises are SOFT references
 * ({ type, refId }) validated at import time, NOT Mongoose refs (talk scenarios
 * live in code, IPA is referenced by `code`).
 */
const { Schema, model } = require('mongoose');

const EXERCISE_TYPES = ['ipa', 'talk']; // quiz hoãn Phase 2 (Q-Quiz)
const CEFR_LEVELS = ['A1', 'A2', 'B1', 'B2', 'C1', 'C2'];
const MODULE_CATEGORIES = ['grammar', 'pronunciation', 'vocabulary', 'listening', 'reading', 'writing', 'speaking', 'other'];

const ExerciseRefSchema = new Schema(
  {
    type: { type: String, enum: EXERCISE_TYPES, required: true },
    refId: { type: String, required: true, trim: true },
    order: { type: Number, required: true },
    label: { type: String, default: null },
  },
  { _id: false }
);

const LessonSchema = new Schema(
  {
    key: { type: String, required: true, trim: true },
    title: { type: String, required: true, trim: true },
    order: { type: Number, required: true },
    // Hướng B — multi-modal content. Media are URLs (team self-hosts), not uploaded here.
    theory: { type: String, default: '' }, // markdown
    videoUrl: { type: String, default: null },
    audioUrl: { type: String, default: null },
    // NOT required: a theory-only lesson (no exercise) is valid. The "at least one of
    // {theory,video,audio,exercise}" rule is enforced in the service (EMPTY_LESSON).
    exercises: { type: [ExerciseRefSchema], default: [] },
  },
  { _id: false }
);

const ModuleSchema = new Schema(
  {
    key: { type: String, required: true, trim: true },
    title: { type: String, required: true, trim: true },
    order: { type: Number, required: true },
    category: { type: String, enum: MODULE_CATEGORIES, default: 'other' }, // Hướng B
    lessons: { type: [LessonSchema], required: true },
  },
  { _id: false }
);

const PhaseSchema = new Schema(
  {
    key: { type: String, required: true, trim: true },
    title: { type: String, required: true, trim: true },
    order: { type: Number, required: true },
    // Hướng B — level progression (A2→B1) or a free-text goal ("IELTS 5.0→6.0").
    cefrFrom: { type: String, enum: [...CEFR_LEVELS, null], default: null },
    cefrTo: { type: String, enum: [...CEFR_LEVELS, null], default: null },
    goalNote: { type: String, default: null },
    modules: { type: [ModuleSchema], required: true },
  },
  { _id: false }
);

const SourceMetaSchema = new Schema(
  {
    filename: { type: String, default: null },
    contentHash: { type: String, default: null },
    nodeCount: { type: Number, default: null },
    format: { type: String, default: 'xlsx' },
  },
  { _id: false }
);

const CourseStructureSchema = new Schema(
  {
    slug: { type: String, required: true, unique: true, trim: true, index: true },
    title: { type: String, required: true, trim: true },
    description: { type: String, default: '' },
    // Phase 1: platform-shared content (Q5) — always null; kept for future center scoping.
    centerId: { type: Schema.Types.ObjectId, ref: 'Center', default: null },
    status: { type: String, enum: ['ready'], default: 'ready', index: true },
    phases: { type: [PhaseSchema], required: true },
    importedBy: { type: Schema.Types.ObjectId, ref: 'User', required: true },
    sourceMeta: { type: SourceMetaSchema, default: null },
  },
  { timestamps: true, collection: 'course_structures' }
);

const CourseStructure = model('CourseStructure', CourseStructureSchema);

module.exports = { CourseStructure, EXERCISE_TYPES, CEFR_LEVELS, MODULE_CATEGORIES };
```

- [ ] **Step 4: Chạy test — xác nhận PASS**

Run:
```bash
cd services/api && npx jest course-content.model -i
```
Expected: 6 test PASS.

- [ ] **Step 5: Commit**

```bash
cd services/api && git add src/modules/course-content/course-content.model.js src/__tests__/course-content.model.test.js
git commit -m "feat(course-content): add CourseStructure embedded-tree model"
```

---

## Task 3: Parser `.xlsx` → IR (`parseXlsx`)

**Files:**
- Create: `services/api/src/modules/course-content/course-content.parser.js`
- Test: `services/api/src/__tests__/course-content.parser.test.js`

- [ ] **Step 1: Viết test thất bại**

Tạo `services/api/src/__tests__/course-content.parser.test.js`:

```js
'use strict';

/** Unit test parser Excel → IR. Dựng .xlsx thật bằng exceljs trong test. */
const ExcelJS = require('exceljs');
const { parseXlsx } = require('../modules/course-content/course-content.parser');
const ApiError = require('../utils/apiError');

// Cột: A Level | B Key | C Title | D Type | E RefId | F CefrFrom | G CefrTo | H Category | I Content | J VideoUrl | K AudioUrl | L Note
const HEADER = ['Level', 'Key', 'Title', 'Type', 'RefId', 'CefrFrom', 'CefrTo', 'Category', 'Content', 'VideoUrl', 'AudioUrl', 'Note'];
async function xlsx(rows) {
  const wb = new ExcelJS.Workbook();
  const ws = wb.addWorksheet('Course');
  ws.addRow(HEADER); // header (dòng 1)
  rows.forEach((r) => ws.addRow(r));
  return Buffer.from(await wb.xlsx.writeBuffer());
}

const HAPPY = [
  // Level      Key       Title      Type   RefId    CefrFrom CefrTo Category   Content            Video               Audio               Note
  ['ROADMAP', 'khoa-a2', 'Khóa A2', '', '', '', '', '', 'Giới thiệu khóa', '', '', ''],                 // dòng 2
  ['PHASE', 'p1', 'Chặng 1', '', '', 'A2', 'B1', '', '', '', '', 'IELTS 5.0→6.0'],                       // dòng 3
  ['MODULE', 'm1', 'Ngữ pháp', '', '', '', '', 'grammar', '', '', '', ''],                               // dòng 4
  ['LESSON', 'l1', 'Bài 1', '', '', '', '', '', '# Lý thuyết', 'https://cdn.x/v.mp4', 'https://cdn.x/a.mp3', ''], // dòng 5
  ['EXERCISE', '', '', 'ipa', 'L1', '', '', '', '', '', '', ''],                                         // dòng 6
  ['EXERCISE', '', '', 'talk', 'travel', '', '', '', '', '', '', ''],                                    // dòng 7
];

test('parse cây hợp lệ → IR đúng + field Hướng B + số dòng', async () => {
  const { course, errors } = await parseXlsx(await xlsx(HAPPY));
  expect(errors).toEqual([]);
  expect(course).toMatchObject({ slug: 'khoa-a2', row: 2, description: 'Giới thiệu khóa' });
  const phase = course.phases[0];
  expect(phase).toMatchObject({ cefrFrom: 'A2', cefrTo: 'B1', goalNote: 'IELTS 5.0→6.0' });
  expect(course.phases[0].modules[0].category).toBe('grammar');
  const lesson = course.phases[0].modules[0].lessons[0];
  expect(lesson).toMatchObject({ key: 'l1', theory: '# Lý thuyết', videoUrl: 'https://cdn.x/v.mp4', audioUrl: 'https://cdn.x/a.mp3' });
  expect(lesson.exercises).toHaveLength(2);
  expect(lesson.exercises[1]).toMatchObject({ type: 'talk', refId: 'travel', row: 7 });
});

test('file không phải xlsx / hỏng → ApiError PARSE_FAILED', async () => {
  await expect(parseXlsx(Buffer.from('not an excel file'))).rejects.toMatchObject({ code: 'PARSE_FAILED' });
});

test('file rỗng (chỉ header) → ApiError EMPTY_COURSE_FILE', async () => {
  await expect(parseXlsx(await xlsx([]))).rejects.toMatchObject({ code: 'EMPTY_COURSE_FILE' });
});

test('PHASE trước ROADMAP → lỗi ORPHAN_NODE trong errors[]', async () => {
  const { errors } = await parseXlsx(await xlsx([['PHASE', 'p1', 'Chặng lạc', '', '', '', '', '', '', '', '', '']]));
  expect(errors.some((e) => e.code === 'ORPHAN_NODE' && e.row === 2)).toBe(true);
});
```

> `ApiError` vẫn được import ở đầu file test (dùng cho `instanceof` nếu cần) — các assertion lỗi trên đối chiếu
> `code` qua `.rejects.toMatchObject`.

- [ ] **Step 2: Chạy test — xác nhận FAIL**

Run:
```bash
cd services/api && npx jest course-content.parser -i
```
Expected: FAIL — `Cannot find module '.../course-content.parser'`.

- [ ] **Step 3: Tạo parser (async) + đồng bộ hoá test**

Tạo `services/api/src/modules/course-content/course-content.parser.js`:

```js
/**
 * Parse the academic team's .xlsx (columns Level|Key|Title|Type|RefId) into a
 * NormalizedCourse IR. Every node carries `row` (real Excel row number) so
 * validation errors can point the team at the exact line (IS-7). Format is
 * isolated here (Q2 = Excel): swapping/adding a format = another parse fn
 * returning the same IR, untouched downstream.
 */
const ExcelJS = require('exceljs');
const ApiError = require('../../utils/apiError');
const HTTP = require('../../constants/http-status');
const CODES = require('../../constants/error-codes');

const LEVELS = new Set(['ROADMAP', 'PHASE', 'MODULE', 'LESSON', 'EXERCISE']);

/** @returns {Promise<{ course: object|null, errors: object[] }>} */
async function parseXlsx(buffer) {
  const wb = new ExcelJS.Workbook();
  try {
    await wb.xlsx.load(buffer);
  } catch {
    throw new ApiError(HTTP.BAD_REQUEST, 'Không đọc được file Excel', CODES.PARSE_FAILED);
  }
  const ws = wb.worksheets[0];
  if (!ws) throw new ApiError(HTTP.BAD_REQUEST, 'File Excel không có sheet nào', CODES.PARSE_FAILED);

  const cell = (row, i) => {
    const v = row.getCell(i).value;
    return v === null || v === undefined ? '' : String(v).trim();
  };

  const errors = [];
  let course = null;
  let curPhase = null;
  let curModule = null;
  let curLesson = null;
  let dataRows = 0;

  ws.eachRow((row, rowNumber) => {
    if (rowNumber === 1) return; // header
    const level = cell(row, 1).toUpperCase();
    if (!level) return; // dòng trống
    dataRows += 1;
    const key = cell(row, 2);
    const title = cell(row, 3);
    const type = cell(row, 4).toLowerCase();
    const refId = cell(row, 5);
    const nz = (i) => cell(row, i) || null; // '' → null (F/G/H/J/K/L)

    if (!LEVELS.has(level)) {
      errors.push({ code: 'MISSING_LEVEL', message: `Dòng ${rowNumber}: Level '${level}' không hợp lệ`, row: rowNumber });
      return;
    }

    if (level === 'ROADMAP') {
      course = { slug: key, title, description: cell(row, 9), row: rowNumber, phases: [] };
      curPhase = curModule = curLesson = null;
    } else if (level === 'PHASE') {
      if (!course) { errors.push(orphan(rowNumber, 'PHASE', 'ROADMAP')); return; }
      curPhase = {
        key, title, order: course.phases.length + 1, row: rowNumber,
        cefrFrom: nz(6), cefrTo: nz(7), goalNote: nz(12), modules: [],
      };
      course.phases.push(curPhase);
      curModule = curLesson = null;
    } else if (level === 'MODULE') {
      if (!curPhase) { errors.push(orphan(rowNumber, 'MODULE', 'PHASE')); return; }
      curModule = {
        key, title, order: curPhase.modules.length + 1, row: rowNumber,
        category: nz(8), lessons: [],
      };
      curPhase.modules.push(curModule);
      curLesson = null;
    } else if (level === 'LESSON') {
      if (!curModule) { errors.push(orphan(rowNumber, 'LESSON', 'MODULE')); return; }
      curLesson = {
        key, title, order: curModule.lessons.length + 1, row: rowNumber,
        theory: cell(row, 9), videoUrl: nz(10), audioUrl: nz(11), exercises: [],
      };
      curModule.lessons.push(curLesson);
    } else if (level === 'EXERCISE') {
      if (!curLesson) { errors.push(orphan(rowNumber, 'EXERCISE', 'LESSON')); return; }
      curLesson.exercises.push({ type, refId, order: curLesson.exercises.length + 1, row: rowNumber });
    }
  });

  if (dataRows === 0 || !course) {
    throw new ApiError(HTTP.BAD_REQUEST, 'Không có nội dung khóa học để import', CODES.EMPTY_COURSE_FILE);
  }
  course.nodeCount = countNodes(course);
  return { course, errors };
}

const orphan = (row, level, parent) => ({
  code: 'ORPHAN_NODE',
  message: `Dòng ${row}: ${level} không thuộc ${parent} nào`,
  row,
});

function countNodes(course) {
  let n = 1;
  for (const p of course.phases) {
    n += 1;
    for (const m of p.modules) {
      n += 1;
      for (const l of m.lessons) n += 1 + l.exercises.length;
    }
  }
  return n;
}

module.exports = { parseXlsx };
```

(Test ở Step 1 đã viết theo API **async** — mọi `parseXlsx(...)` đều `await`, các case lỗi dùng
`.rejects.toMatchObject({ code })`. Không cần sửa thêm.)

- [ ] **Step 4: Chạy test — xác nhận PASS**

Run:
```bash
cd services/api && npx jest course-content.parser -i
```
Expected: 4 test PASS.

- [ ] **Step 5: Commit**

```bash
cd services/api && git add src/modules/course-content/course-content.parser.js src/__tests__/course-content.parser.test.js
git commit -m "feat(course-content): parse .xlsx into normalized course tree with row tracking"
```

---

## Task 4: Reference resolver (IPA published / talk / quiz)

**Files:**
- Create: `services/api/src/modules/course-content/course-content.references.js`
- Test: `services/api/src/__tests__/course-content.references.test.js`

- [ ] **Step 1: Viết test thất bại**

Tạo `services/api/src/__tests__/course-content.references.test.js`:

```js
'use strict';

/** Unit test resolver tham chiếu: IPA (chỉ published), talk (SCENARIO_IDS), quiz (unsupported). */
jest.mock('../config/logger', () => ({ info: jest.fn(), warn: jest.fn(), error: jest.fn(), debug: jest.fn() }));

const mongoose = require('mongoose');
const { connectDb, clearDb, disconnectDb } = require('./helpers/db');
const { IpaLesson } = require('../modules/ipa/ipa-lesson.model');
const { resolveReferences } = require('../modules/course-content/course-content.references');

const oid = () => new mongoose.Types.ObjectId();

const seedIpa = (code, status) =>
  IpaLesson.create({ sectionId: oid(), code, order: 1, targetIpa: 'ɪ', template: 'A', status });

// course chỉ cần cây tối thiểu tới exercises để resolver duyệt.
const courseWith = (exercises) => ({
  phases: [{ modules: [{ lessons: [{ exercises }] }] }],
});

beforeAll(connectDb);
afterEach(clearDb);
afterAll(disconnectDb);

test('IPA published hợp lệ → không lỗi', async () => {
  await seedIpa('L1', 'published');
  const errors = await resolveReferences(courseWith([{ type: 'ipa', refId: 'L1', row: 6 }]));
  expect(errors).toEqual([]);
});

test('IPA tồn tại nhưng chưa publish → REFERENCE_NOT_PUBLISHED', async () => {
  await seedIpa('L2', 'draft');
  const errors = await resolveReferences(courseWith([{ type: 'ipa', refId: 'L2', row: 6 }]));
  expect(errors).toEqual([
    expect.objectContaining({ code: 'REFERENCE_NOT_PUBLISHED', refType: 'ipa', refId: 'L2', row: 6 }),
  ]);
});

test('IPA không tồn tại → REFERENCE_NOT_FOUND', async () => {
  const errors = await resolveReferences(courseWith([{ type: 'ipa', refId: 'L99', row: 6 }]));
  expect(errors[0]).toMatchObject({ code: 'REFERENCE_NOT_FOUND', refType: 'ipa', refId: 'L99', row: 6 });
});

test('talk scenario hợp lệ / sai', async () => {
  const errors = await resolveReferences(courseWith([
    { type: 'talk', refId: 'travel', row: 7 },   // hợp lệ (SCENARIO_IDS)
    { type: 'talk', refId: 'aiport', row: 8 },   // sai
  ]));
  expect(errors).toEqual([
    expect.objectContaining({ code: 'REFERENCE_NOT_FOUND', refType: 'talk', refId: 'aiport', row: 8 }),
  ]);
});

test('quiz → UNSUPPORTED_EXERCISE_TYPE', async () => {
  const errors = await resolveReferences(courseWith([{ type: 'quiz', refId: 'q1', row: 9 }]));
  expect(errors[0]).toMatchObject({ code: 'UNSUPPORTED_EXERCISE_TYPE', refType: 'quiz', row: 9 });
});
```

- [ ] **Step 2: Chạy test — xác nhận FAIL**

Run:
```bash
cd services/api && npx jest course-content.references -i
```
Expected: FAIL — `Cannot find module '.../course-content.references'`.

- [ ] **Step 3: Tạo resolver**

Tạo `services/api/src/modules/course-content/course-content.references.js`:

```js
/**
 * Validate every exercise reference in a NormalizedCourse against its real source
 * (IS-4). Three DIFFERENT sources: IPA → ipa_lessons collection (must be
 * status:'published', Q3); talk → in-code SCENARIO_IDS constant (no DB); quiz →
 * unsupported in Phase 1 (Q-Quiz, deferred). Refs are batched per type ($in) —
 * one query per type, not per ref. READ-ONLY: never mutates ipa/talk.
 */
const { IpaLesson } = require('../ipa/ipa-lesson.model');
const { SCENARIO_IDS } = require('../talk/talk.scenarios');

/** Flatten all exercise refs (with row) out of the tree. */
function collectRefs(course) {
  const refs = [];
  for (const p of course.phases || []) {
    for (const m of p.modules || []) {
      for (const l of m.lessons || []) {
        for (const ex of l.exercises || []) refs.push(ex);
      }
    }
  }
  return refs;
}

async function resolveReferences(course) {
  const refs = collectRefs(course);
  const errors = [];

  const ipaIds = [];
  const talkRefs = [];
  for (const ex of refs) {
    if (ex.type === 'ipa') ipaIds.push(ex);
    else if (ex.type === 'talk') talkRefs.push(ex);
    else {
      errors.push({
        code: 'UNSUPPORTED_EXERCISE_TYPE',
        message: `Dòng ${ex.row}: loại bài tập '${ex.type}' chưa được hỗ trợ (dự kiến Phase 2)`,
        row: ex.row, refType: ex.type, refId: ex.refId,
      });
    }
  }

  // IPA: một query batch, chỉ published.
  if (ipaIds.length) {
    const codes = [...new Set(ipaIds.map((e) => e.refId))];
    const rows = await IpaLesson.find({ code: { $in: codes } }).select('code status').lean();
    const byCode = new Map(rows.map((r) => [r.code, r.status]));
    for (const ex of ipaIds) {
      const status = byCode.get(ex.refId);
      if (status === undefined) {
        errors.push(refErr('REFERENCE_NOT_FOUND', ex, `bài tập IPA '${ex.refId}' không tồn tại`));
      } else if (status !== 'published') {
        errors.push(refErr('REFERENCE_NOT_PUBLISHED', ex, `bài tập IPA '${ex.refId}' chưa được publish`));
      }
    }
  }

  // talk: đối chiếu hằng in-code.
  for (const ex of talkRefs) {
    if (!SCENARIO_IDS.includes(ex.refId)) {
      errors.push(refErr('REFERENCE_NOT_FOUND', ex, `talk scenario '${ex.refId}' không tồn tại`));
    }
  }

  return errors;
}

const refErr = (code, ex, msg) => ({
  code, message: `Dòng ${ex.row}: ${msg}`, row: ex.row, refType: ex.type, refId: ex.refId,
});

module.exports = { resolveReferences, collectRefs };
```

- [ ] **Step 4: Chạy test — xác nhận PASS**

Run:
```bash
cd services/api && npx jest course-content.references -i
```
Expected: 5 test PASS.

- [ ] **Step 5: Commit**

```bash
cd services/api && git add src/modules/course-content/course-content.references.js src/__tests__/course-content.references.test.js
git commit -m "feat(course-content): validate exercise references (ipa published / talk / quiz deferred)"
```

---

## Task 5: Service `importCourse` (validate + all-or-nothing)

**Files:**
- Create: `services/api/src/modules/course-content/course-content.service.js`
- Test: `services/api/src/__tests__/course-content.service.test.js`

- [ ] **Step 1: Viết test thất bại**

Tạo `services/api/src/__tests__/course-content.service.test.js`:

```js
'use strict';

/** Unit test service: validateCourse (thuần) + importCourse (DB, all-or-nothing). */
jest.mock('../config/logger', () => ({ info: jest.fn(), warn: jest.fn(), error: jest.fn(), debug: jest.fn() }));

const mongoose = require('mongoose');
const ExcelJS = require('exceljs');
const { connectDb, clearDb, disconnectDb } = require('./helpers/db');
const { IpaLesson } = require('../modules/ipa/ipa-lesson.model');
const { CourseStructure } = require('../modules/course-content/course-content.model');
const svc = require('../modules/course-content/course-content.service');

const oid = () => new mongoose.Types.ObjectId();
const user = () => ({ id: oid().toString() });

async function xlsx(rows) {
  const wb = new ExcelJS.Workbook();
  const ws = wb.addWorksheet('Course');
  ws.addRow(['Level', 'Key', 'Title', 'Type', 'RefId']);
  rows.forEach((r) => ws.addRow(r));
  return Buffer.from(await wb.xlsx.writeBuffer());
}

const VALID = [
  ['ROADMAP', 'khoa-a2', 'Khóa A2', '', ''],
  ['PHASE', 'p1', 'Chặng 1', '', ''],
  ['MODULE', 'm1', 'Chuyên đề 1', '', ''],
  ['LESSON', 'l1', 'Bài 1', '', ''],
  ['EXERCISE', '', '', 'ipa', 'L1'],
];

beforeAll(async () => { await connectDb(); await CourseStructure.init(); });
afterEach(clearDb);
afterAll(disconnectDb);

// ── validateCourse (thuần, không DB) ──────────────────────────────────────
describe('validateCourse', () => {
  const lesson = (extra = {}) => ({ key: 'l1', title: 'L', order: 1, row: 5, theory: '', videoUrl: null, audioUrl: null, exercises: [{ type: 'ipa', refId: 'L1', order: 1, row: 6 }], ...extra });
  const tree = (lessonExtra = {}) => ({
    slug: 'k', title: 'K', row: 2,
    phases: [{ key: 'p1', title: 'P', order: 1, row: 3, cefrFrom: 'A2', cefrTo: 'B1', modules: [
      { key: 'm1', title: 'M', order: 1, row: 4, category: 'grammar', lessons: [lesson(lessonExtra)] },
    ] }],
  });

  test('cây hợp lệ → không lỗi', () => {
    expect(svc.validateCourse(tree())).toEqual([]);
  });

  test('Bài học chỉ có lý thuyết (không exercises) → hợp lệ', () => {
    expect(svc.validateCourse(tree({ theory: '# Lý thuyết', exercises: [] }))).toEqual([]);
  });

  test('Bài học rỗng hoàn toàn → EMPTY_LESSON', () => {
    const errs = svc.validateCourse(tree({ theory: '', videoUrl: null, audioUrl: null, exercises: [] }));
    expect(errs.some((e) => e.code === 'EMPTY_LESSON' && e.row === 5)).toBe(true);
  });

  test('Chuyên đề không có Bài học → EMPTY_CHILDREN', () => {
    const t = tree(); t.phases[0].modules[0].lessons = [];
    expect(svc.validateCourse(t).some((e) => e.code === 'EMPTY_CHILDREN' && e.row === 4)).toBe(true);
  });

  test('trùng key Bài học trong cùng Chuyên đề → DUPLICATE_KEY', () => {
    const t = tree();
    t.phases[0].modules[0].lessons.push({ key: 'l1', title: 'L2', order: 2, row: 7, theory: 't', exercises: [] });
    expect(svc.validateCourse(t).some((e) => e.code === 'DUPLICATE_KEY' && e.row === 7)).toBe(true);
  });

  test('cefrTo < cefrFrom → INVALID_CEFR', () => {
    const t = tree(); t.phases[0].cefrFrom = 'B1'; t.phases[0].cefrTo = 'A2';
    expect(svc.validateCourse(t).some((e) => e.code === 'INVALID_CEFR' && e.row === 3)).toBe(true);
  });

  test('videoUrl không phải http(s) → INVALID_URL', () => {
    expect(svc.validateCourse(tree({ videoUrl: 'ftp://x/v.mp4' })).some((e) => e.code === 'INVALID_URL' && e.row === 5)).toBe(true);
  });
});

// ── importCourse (DB) ─────────────────────────────────────────────────────
describe('importCourse', () => {
  const seedPublishedL1 = () =>
    IpaLesson.create({ sectionId: oid(), code: 'L1', order: 1, targetIpa: 'ɪ', template: 'A', status: 'published' });

  test('file hợp lệ → tạo đúng 1 document + summary counts', async () => {
    await seedPublishedL1();
    const summary = await svc.importCourse({ buffer: await xlsx(VALID), user: user() });
    expect(summary).toMatchObject({ slug: 'khoa-a2', status: 'ready', counts: { phases: 1, modules: 1, lessons: 1, exercises: 1 } });
    expect(await CourseStructure.countDocuments()).toBe(1);
  });

  test('all-or-nothing: ID không tồn tại → throw, 0 document', async () => {
    // KHÔNG seed L1 → REFERENCE_NOT_FOUND
    await expect(svc.importCourse({ buffer: await xlsx(VALID), user: user() }))
      .rejects.toMatchObject({ status: 400, code: 'IMPORT_VALIDATION_FAILED' });
    expect(await CourseStructure.countDocuments()).toBe(0);
  });

  test('trùng slug đã tồn tại → IMPORT_VALIDATION_FAILED (ALREADY_EXISTS), không đè', async () => {
    await seedPublishedL1();
    await svc.importCourse({ buffer: await xlsx(VALID), user: user() });
    let thrown;
    try { await svc.importCourse({ buffer: await xlsx(VALID), user: user() }); } catch (e) { thrown = e; }
    expect(thrown).toMatchObject({ status: 400, code: 'IMPORT_VALIDATION_FAILED' });
    expect(thrown.errors.some((x) => x.code === 'ALREADY_EXISTS')).toBe(true);
    expect(await CourseStructure.countDocuments()).toBe(1); // vẫn 1, không đè/không thêm
  });
});
```

- [ ] **Step 2: Chạy test — xác nhận FAIL**

Run:
```bash
cd services/api && npx jest course-content.service -i
```
Expected: FAIL — `Cannot find module '.../course-content.service'`.

- [ ] **Step 3: Tạo service**

Tạo `services/api/src/modules/course-content/course-content.service.js`:

```js
/**
 * Course-content import service. Orchestrates: parse .xlsx → validate structure
 * → check duplicate → validate references → build ONE embedded document → insert
 * (atomic). ALL validation happens BEFORE any write, and errors are gathered into
 * a single list (IS-7) so the whole file is rejected with 0 records on any error
 * (all-or-nothing / IS-5 / AC-4). Called by admin-api (delegated — no business
 * logic in the admin layer).
 */
const { parseXlsx } = require('./course-content.parser');
const { resolveReferences } = require('./course-content.references');
const { CourseStructure, CEFR_LEVELS, MODULE_CATEGORIES } = require('./course-content.model');
const ApiError = require('../../utils/apiError');
const HTTP = require('../../constants/http-status');
const CODES = require('../../constants/error-codes');

const URL_RE = /^https?:\/\//i;

/** Pure structural + Hướng B metadata/content validation on the IR (row-precise errors). */
function validateCourse(course) {
  const errors = [];
  const dup = (items, scopeMsg) => {
    const seen = new Set();
    for (const it of items) {
      if (seen.has(it.key)) {
        errors.push({ code: 'DUPLICATE_KEY', message: `Dòng ${it.row}: ${scopeMsg} trùng định danh '${it.key}'`, row: it.row });
      }
      seen.add(it.key);
    }
  };

  if (!course.phases.length) {
    errors.push({ code: 'EMPTY_CHILDREN', message: `Dòng ${course.row}: Lộ trình không có Chặng nào`, row: course.row });
  }
  dup(course.phases, 'Chặng');
  for (const p of course.phases) {
    if (!p.modules.length) errors.push({ code: 'EMPTY_CHILDREN', message: `Dòng ${p.row}: Chặng không có Chuyên đề nào`, row: p.row });
    // CEFR (Hướng B): value hợp lệ + thứ tự cefrTo >= cefrFrom.
    for (const [f, v] of [['cefrFrom', p.cefrFrom], ['cefrTo', p.cefrTo]]) {
      if (v && !CEFR_LEVELS.includes(v)) errors.push({ code: 'INVALID_CEFR', message: `Dòng ${p.row}: ${f}='${v}' không thuộc CEFR`, row: p.row });
    }
    if (p.cefrFrom && p.cefrTo && CEFR_LEVELS.includes(p.cefrFrom) && CEFR_LEVELS.includes(p.cefrTo)
        && CEFR_LEVELS.indexOf(p.cefrTo) < CEFR_LEVELS.indexOf(p.cefrFrom)) {
      errors.push({ code: 'INVALID_CEFR', message: `Dòng ${p.row}: cefrTo (${p.cefrTo}) phải ≥ cefrFrom (${p.cefrFrom})`, row: p.row });
    }
    dup(p.modules, 'Chuyên đề');
    for (const m of p.modules) {
      if (m.category && !MODULE_CATEGORIES.includes(m.category)) {
        errors.push({ code: 'INVALID_CATEGORY', message: `Dòng ${m.row}: category '${m.category}' không hợp lệ`, row: m.row });
      }
      if (!m.lessons.length) errors.push({ code: 'EMPTY_CHILDREN', message: `Dòng ${m.row}: Chuyên đề không có Bài học nào`, row: m.row });
      dup(m.lessons, 'Bài học');
      for (const l of m.lessons) {
        // Bài học không rỗng: ≥1 trong {theory, video, audio, exercise} (Hướng B).
        if (!l.theory && !l.videoUrl && !l.audioUrl && !l.exercises.length) {
          errors.push({ code: 'EMPTY_LESSON', message: `Dòng ${l.row}: Bài học không có lý thuyết/video/audio/bài tập nào`, row: l.row });
        }
        for (const [f, v] of [['videoUrl', l.videoUrl], ['audioUrl', l.audioUrl]]) {
          if (v && !URL_RE.test(v)) errors.push({ code: 'INVALID_URL', message: `Dòng ${l.row}: ${f} phải là http(s)://…`, row: l.row });
        }
      }
    }
  }
  return errors;
}

function buildDoc(course, user) {
  const strip = (n, keep) => keep.reduce((o, k) => ((o[k] = n[k]), o), {});
  return {
    slug: course.slug,
    title: course.title,
    description: course.description || '',
    centerId: null, // Phase 1 platform-shared (Q5)
    status: 'ready',
    importedBy: user.id,
    sourceMeta: { format: 'xlsx', nodeCount: course.nodeCount ?? null },
    phases: course.phases.map((p) => ({
      ...strip(p, ['key', 'title', 'order', 'cefrFrom', 'cefrTo', 'goalNote']),
      modules: p.modules.map((m) => ({
        ...strip(m, ['key', 'title', 'order', 'category']),
        lessons: m.lessons.map((l) => ({
          ...strip(l, ['key', 'title', 'order', 'theory', 'videoUrl', 'audioUrl']),
          exercises: l.exercises.map((e) => strip(e, ['type', 'refId', 'order', 'label'])),
        })),
      })),
    })),
  };
}

function toSummary(doc) {
  let modules = 0, lessons = 0, exercises = 0;
  for (const p of doc.phases) {
    modules += p.modules.length;
    for (const m of p.modules) {
      lessons += m.lessons.length;
      for (const l of m.lessons) exercises += l.exercises.length;
    }
  }
  return { id: String(doc._id), slug: doc.slug, title: doc.title, status: doc.status, counts: { phases: doc.phases.length, modules, lessons, exercises } };
}

async function importCourse({ buffer, user }) {
  // parseXlsx throws ApiError(PARSE_FAILED / EMPTY_COURSE_FILE) — propagate as-is.
  const { course, errors: parseErrors } = await parseXlsx(buffer);

  const errors = [...parseErrors, ...validateCourse(course)];

  if (course.slug && (await CourseStructure.exists({ slug: course.slug }))) {
    errors.push({ code: 'ALREADY_EXISTS', message: `Dòng ${course.row}: khóa '${course.slug}' đã tồn tại`, row: course.row });
  }
  errors.push(...(await resolveReferences(course)));

  if (errors.length) {
    throw new ApiError(HTTP.BAD_REQUEST, 'Import validation failed', CODES.IMPORT_VALIDATION_FAILED, errors);
  }

  try {
    const doc = await CourseStructure.create(buildDoc(course, user));
    return toSummary(doc);
  } catch (err) {
    // Safety net: slug race → duplicate key.
    if (err && err.code === 11000) {
      throw new ApiError(HTTP.BAD_REQUEST, 'Import validation failed', CODES.IMPORT_VALIDATION_FAILED, [
        { code: 'ALREADY_EXISTS', message: `Khóa '${course.slug}' đã tồn tại`, row: course.row },
      ]);
    }
    throw err;
  }
}

module.exports = { importCourse, validateCourse, buildDoc, toSummary };
```

- [ ] **Step 4: Chạy test — xác nhận PASS**

Run:
```bash
cd services/api && npx jest course-content.service -i
```
Expected: 10 test PASS.

- [ ] **Step 5: Commit**

```bash
cd services/api && git add src/modules/course-content/course-content.service.js src/__tests__/course-content.service.test.js
git commit -m "feat(course-content): importCourse service (validate-then-write, all-or-nothing)"
```

---

## Task 6: Upload middleware + admin route + API test

**Files:**
- Create: `services/api/src/middlewares/course-import.upload.js`
- Create: `services/api/src/admin-api/resources/course-content.admin.js`
- Modify: `services/api/src/admin-api/_router.js`
- Test: `services/api/src/__tests__/course-content.import.api.test.js`

- [ ] **Step 1: Viết test thất bại (API qua supertest)**

Tạo `services/api/src/__tests__/course-content.import.api.test.js`:

```js
'use strict';

/** API test import qua admin-api: 403 thiếu quyền, 200 + audit khi hợp lệ. In-memory Mongo. */
jest.mock('../config/logger', () => ({ info: jest.fn(), warn: jest.fn(), error: jest.fn(), debug: jest.fn(), http: jest.fn() }));
jest.mock('../utils/audit', () => ({ auditLog: jest.fn() }));

const express = require('express');
const request = require('supertest');
const mongoose = require('mongoose');
const ExcelJS = require('exceljs');
const { connectDb, clearDb, disconnectDb } = require('./helpers/db');
const User = require('../modules/user/user.model');
const { IpaLesson } = require('../modules/ipa/ipa-lesson.model');
const { CourseStructure } = require('../modules/course-content/course-content.model');
const { signAccessToken } = require('../utils/jwt');
const { auditLog } = require('../utils/audit');
const errorHandler = require('../middlewares/errorHandler');

const oid = () => new mongoose.Types.ObjectId();

function buildApp() {
  const app = express();
  app.use(express.json());
  app.use('/api/admin', require('../admin-api/_router'));
  app.use(errorHandler);
  return app;
}

async function seedUser(permissions = []) {
  const u = await User.create({ email: `u${oid()}@t.dev`, name: 'Staff', role: 'user', permissions, isActive: true });
  return { token: signAccessToken({ id: u._id.toString(), role: 'user' }) };
}

async function xlsxBuffer(rows) {
  const wb = new ExcelJS.Workbook();
  const ws = wb.addWorksheet('Course');
  ws.addRow(['Level', 'Key', 'Title', 'Type', 'RefId']);
  rows.forEach((r) => ws.addRow(r));
  return Buffer.from(await wb.xlsx.writeBuffer());
}

const XLSX_MIME = 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet';
const VALID = [
  ['ROADMAP', 'khoa-a2', 'Khóa A2', '', ''],
  ['PHASE', 'p1', 'Chặng 1', '', ''],
  ['MODULE', 'm1', 'Chuyên đề 1', '', ''],
  ['LESSON', 'l1', 'Bài 1', '', ''],
  ['EXERCISE', '', '', 'ipa', 'L1'],
];

let app;
beforeAll(async () => { await connectDb(); await CourseStructure.init(); app = buildApp(); });
afterEach(async () => { await clearDb(); jest.clearAllMocks(); });
afterAll(disconnectDb);

test('401 khi không có token', async () => {
  const res = await request(app).post('/api/admin/course-imports/_/import')
    .attach('file', await xlsxBuffer(VALID), { filename: 'c.xlsx', contentType: XLSX_MIME });
  expect(res.status).toBe(401);
});

test('403 khi thiếu coursecontent:write, không tạo document', async () => {
  const { token } = await seedUser([]); // không quyền
  const res = await request(app).post('/api/admin/course-imports/_/import')
    .set('Authorization', `Bearer ${token}`)
    .attach('file', await xlsxBuffer(VALID), { filename: 'c.xlsx', contentType: XLSX_MIME });
  expect(res.status).toBe(403);
  expect(await CourseStructure.countDocuments()).toBe(0);
});

test('200 + summary + audit khi hợp lệ', async () => {
  const { token } = await seedUser(['coursecontent:write']);
  await IpaLesson.create({ sectionId: oid(), code: 'L1', order: 1, targetIpa: 'ɪ', template: 'A', status: 'published' });
  const res = await request(app).post('/api/admin/course-imports/_/import')
    .set('Authorization', `Bearer ${token}`)
    .attach('file', await xlsxBuffer(VALID), { filename: 'c.xlsx', contentType: XLSX_MIME });
  expect(res.status).toBe(200);
  expect(res.body.data.summary.counts).toEqual({ phases: 1, modules: 1, lessons: 1, exercises: 1 });
  expect(auditLog).toHaveBeenCalledWith('admin.command.executed', expect.objectContaining({ command: 'coursecontent.import' }));
});

test('400 + errors[] khi ID không tồn tại', async () => {
  const { token } = await seedUser(['coursecontent:write']); // KHÔNG seed L1
  const res = await request(app).post('/api/admin/course-imports/_/import')
    .set('Authorization', `Bearer ${token}`)
    .attach('file', await xlsxBuffer(VALID), { filename: 'c.xlsx', contentType: XLSX_MIME });
  expect(res.status).toBe(400);
  expect(res.body.code).toBe('IMPORT_VALIDATION_FAILED');
  expect(res.body.errors[0]).toMatchObject({ code: 'REFERENCE_NOT_FOUND', refId: 'L1' });
});
```

- [ ] **Step 2: Chạy test — xác nhận FAIL**

Run:
```bash
cd services/api && npx jest course-content.import.api -i
```
Expected: FAIL — route `/api/admin/course-imports/_/import` chưa tồn tại (404), các test 200/400/403 đỏ.

- [ ] **Step 3: Tạo upload middleware**

Tạo `services/api/src/middlewares/course-import.upload.js`:

```js
/**
 * Multer upload for the course-import .xlsx — one field `file` into memory, ≤5MB
 * (Q6), only .xlsx mime. Maps multer errors → ApiError so errorHandler returns the
 * standard { message, code } body. Mirrors middlewares/upload.js.
 */
const multer = require('multer');
const ApiError = require('../utils/apiError');
const HTTP = require('../constants/http-status');
const CODES = require('../constants/error-codes');

const MAX_BYTES = 5 * 1024 * 1024;
const XLSX_MIME = 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet';

const uploader = multer({
  storage: multer.memoryStorage(),
  limits: { fileSize: MAX_BYTES, files: 1 },
  fileFilter: (req, file, cb) => {
    if (file.mimetype === XLSX_MIME) return cb(null, true);
    return cb(new ApiError(HTTP.BAD_REQUEST, 'File phải là định dạng .xlsx', CODES.INVALID_FILE_TYPE));
  },
});

const courseImportUpload = (req, res, next) => {
  uploader.single('file')(req, res, (err) => {
    if (!err) return next();
    if (err instanceof multer.MulterError) {
      if (err.code === 'LIMIT_FILE_SIZE') {
        return next(new ApiError(HTTP.BAD_REQUEST, 'File vượt quá 5MB', CODES.FILE_TOO_LARGE));
      }
      return next(new ApiError(HTTP.BAD_REQUEST, 'Upload file thất bại', CODES.PARSE_FAILED));
    }
    return next(err); // ApiError từ fileFilter hoặc lỗi khác
  });
};

module.exports = { courseImportUpload };
```

- [ ] **Step 4: Tạo admin router (hand-mount, delegate + audit)**

Tạo `services/api/src/admin-api/resources/course-content.admin.js`:

```js
/**
 * Admin route — course-structure import (Phase 1). Hand-mounted (NOT via the CRUD
 * factory actions) because it needs a multer middleware for multipart before the
 * handler. Follows the admin-api HARD RULE: no business logic here — delegates to
 * course-content.service.importCourse and emits the audit log manually (same shape
 * as /staff in _router.js). List/read of imported courses is Phase 2 (consume).
 */
const express = require('express');
const { verifyToken, verifyPermission } = require('../../middlewares/auth');
const asyncHandler = require('../../utils/asyncHandler');
const ApiError = require('../../utils/apiError');
const HTTP = require('../../constants/http-status');
const CODES = require('../../constants/error-codes');
const { PERMISSIONS } = require('../../constants/permissions');
const { auditLog } = require('../../utils/audit');
const { courseImportUpload } = require('../../middlewares/course-import.upload');
const courseContentService = require('../../modules/course-content/course-content.service');

const router = express.Router();

router.post(
  '/_/import',
  verifyToken,
  verifyPermission(PERMISSIONS.COURSECONTENT_WRITE),
  courseImportUpload,
  asyncHandler(async (req, res) => {
    if (!req.file) throw new ApiError(HTTP.BAD_REQUEST, 'Thiếu file import', CODES.PARSE_FAILED);
    const summary = await courseContentService.importCourse({ buffer: req.file.buffer, user: req.user });
    auditLog('admin.command.executed', {
      adminUser: req.user.id, command: 'coursecontent.import', target: summary.slug, ip: req.ip,
    });
    res.status(HTTP.OK).json({ data: { summary } });
  })
);

module.exports = router;
```

- [ ] **Step 5: Đăng ký route trong `_router.js`**

Trong `services/api/src/admin-api/_router.js`, thêm (cạnh các `router.use('/reviews', ...)` hand-mounted, trước
vòng lặp factory-generated):

```js
// ── Course-structure import (đội học thuật — content ingestion Phase 1) ──────
router.use('/course-imports', require('./resources/course-content.admin'));
```

- [ ] **Step 6: Chạy test — xác nhận PASS**

Run:
```bash
cd services/api && npx jest course-content.import.api -i
```
Expected: 5 test PASS.

- [ ] **Step 7: Commit**

```bash
cd services/api && git add src/middlewares/course-import.upload.js src/admin-api/resources/course-content.admin.js src/admin-api/_router.js src/__tests__/course-content.import.api.test.js
git commit -m "feat(admin): POST /api/admin/course-imports/_/import (xlsx course ingestion)"
```

---

## Task 7: Trang Import trên `exe-admin` (cross-repo consumer)

> Repo khác (`exe-admin`, Next.js) — không có jest backend; verify bằng chạy dev + smoke thủ công. Deploy SAU
> `exe-api` (design §11). Contract: `specs/course-content-import/contracts/admin-course-import.md`.

**Files:**
- Create: `exe-admin/src/services/course-content.service.ts`
- Create: `exe-admin/src/app/(admin)/course-import/page.tsx`

- [ ] **Step 1: Service gọi API (multipart)**

Tạo `exe-admin/src/services/course-content.service.ts` (theo mẫu `admin.service.ts` + `apiClient`):

```ts
import { apiClient } from "@/lib/axios";

export interface ImportSummary {
  id: string;
  slug: string;
  title: string;
  status: string;
  counts: { phases: number; modules: number; lessons: number; exercises: number };
}

export interface ImportError {
  code: string;
  message: string;
  row?: number;
  refType?: string;
  refId?: string;
}

/** POST /admin/course-imports/_/import — multipart. Trả summary; ném lỗi mang errors[] để UI liệt kê. */
export async function importCourse(file: File): Promise<ImportSummary> {
  const form = new FormData();
  form.append("file", file);
  const { data } = await apiClient.post<{ data: { summary: ImportSummary } }>(
    "/admin/course-imports/_/import",
    form,
    { headers: { "Content-Type": "multipart/form-data" } }
  );
  return data.data.summary;
}
```

- [ ] **Step 2: Trang Import (upload + hiển thị summary/lỗi)**

Tạo `exe-admin/src/app/(admin)/course-import/page.tsx`: form chọn file `.xlsx` → gọi `importCourse` →
- thành công: hiển thị `counts` (số Chặng/Chuyên đề/Bài học) — AC-5.
- thất bại: đọc `err.response.data.errors[]` render bảng (dòng + message); nếu không có `errors[]` thì hiển thị
  `err.response.data.message` (không hiện "import failed" chung chung) — AC-5.

(Component theo pattern UI có sẵn trong `src/components/`; dùng `useState` cho `summary`/`errors`/`loading`.)

- [ ] **Step 3: Smoke test thủ công**

Run (cần `exe-api` chạy + tài khoản có `coursecontent:write`):
```bash
cd exe-admin && npm run dev
```
Mở `/course-import`, thử:
- upload file `.xlsx` hợp lệ (mọi IPA ref đã publish) → thấy tóm tắt counts đúng.
- upload file có ID sai → thấy danh sách lỗi kèm số dòng, không có bản ghi nào được tạo (kiểm tra lại DB).
Expected: đúng như trên.

- [ ] **Step 4: Commit (trong repo exe-admin)**

```bash
cd exe-admin && git add src/services/course-content.service.ts "src/app/(admin)/course-import/page.tsx"
git commit -m "feat(course-import): admin Import page for static course .xlsx"
```

---

## Task 8: Chạy full suite + Self-Review

**Files:** (không sửa code) — verify + đối chiếu AC

- [ ] **Step 1: Chạy toàn bộ test của feature**

Run:
```bash
cd services/api && npx jest course-content -i
```
Expected: ALL PASS (model + parser + references + service + import.api).

- [ ] **Step 2: Chạy full suite — không hồi quy**

Run:
```bash
cd services/api && npm test
```
Expected: suite xanh; không có test cũ chuyển đỏ vì thay đổi này. (Nếu có test đỏ sẵn không liên quan — ghi chú lại.)

- [ ] **Step 3: Ghi chú deploy**

Nhắc người phụ trách deploy (design §11):
- Deploy `exe-api` trước `exe-admin`.
- **Seed `coursecontent:write` (hoặc `coursecontent:manage`)** cho tài khoản đội học thuật/platform admin — nếu
  không, trang Import trả 403.
- nginx: nếu có reverse proxy, đảm bảo `client_max_body_size ≥ 5m` cho `/api/admin/course-imports/*` (khớp giới
  hạn 5MB), nếu không upload file lớn bị 413 trước khi tới Node.

- [ ] **Step 4: Điền Self-Review bên dưới.**

---

## Self-Review

**Acceptance coverage** (đối chiếu `acceptance.md`):

- **AC-1** (import hợp lệ tạo đủ cấu trúc + summary counts) → Task 5 (`importCourse` happy) + Task 6 (API 200 counts). ✅
- **AC-2** (từ chối sai cấu trúc, lỗi chỉ rõ dòng, không ghi) → Task 3 (ORPHAN + row) + Task 5 (`validateCourse` EMPTY_CHILDREN). ✅
- **AC-3** (từ chối exercise ID không tồn tại, liệt kê đúng ID) → Task 4 (REFERENCE_NOT_FOUND) + Task 6 (API errors[]). ✅
- **AC-4** (all-or-nothing) → Task 5 (`countDocuments === 0` khi lỗi) + single-insert model Task 2. ✅
- **AC-5** (hiển thị kết quả trên exe-admin) → Task 7 (page render summary + errors[]). ✅
- **AC-6** (lưu CEFR Chặng + category Chuyên đề — IS-9) → Task 2 (model field + enum) + Task 3 (parser cột F/G/H/L) + Task 5 (validate INVALID_CEFR/INVALID_CATEGORY). ✅
- **AC-7** (theory/video/audio + Bài học lý thuyết-thuần — IS-10) → Task 2 (model field, exercises không bắt buộc) + Task 3 (parser cột I/J/K) + Task 5 (EMPTY_LESSON thay vì bắt ≥1 exercise). ✅
- **AC-E1** (file không parse được → lỗi rõ, không 500) → Task 3 (PARSE_FAILED). ✅
- **AC-E2** (file rỗng / không tầng) → Task 3 (EMPTY_COURSE_FILE). ✅
- **AC-E3** (trùng key nội bộ) → Task 5 (DUPLICATE_KEY). ✅
- **AC-E4** (thiếu quyền → từ chối, không ghi) → Task 6 (403 + countDocuments 0). ✅
- **AC-E5** (trùng khóa đã tồn tại — Q4/(a)) → Task 5 (ALREADY_EXISTS + safety net E11000, không đè). ✅
- **AC-E6** (bài tập chưa publish) → Task 4 (REFERENCE_NOT_PUBLISHED). ✅
- **AC-E7** (type quiz — hoãn Phase 2) → Task 2 (enum model) + Task 4 (UNSUPPORTED_EXERCISE_TYPE). ✅
- **AC-E8** (Bài học rỗng hoàn toàn — IS-10) → Task 5 (EMPTY_LESSON). ✅
- **AC-E9** (metadata/URL sai — IS-9/IS-10) → Task 5 (INVALID_CEFR / INVALID_CATEGORY / INVALID_URL). ✅
- **NFR Bảo mật** (permission platform-only, centerId từ server) → Task 1 (permission, không vào CENTER) + Task 5 (`centerId: null`) + Task 6 (403 test). ✅
- **NFR Audit** (audit hook mỗi import) → Task 6 (`auditLog` gọi với `coursecontent.import`). ✅
- **NFR Hiệu năng** (đồng bộ, ≤5MB) → Task 6 (multer 5MB) + Task 8 §3 (nginx note). ✅

**Placeholder scan:** không có TBD/TODO; mọi step có code/command cụ thể. ✅

**Type/name consistency:**
- `parseXlsx` (Task 3) trả `{ course, errors }` async → dùng đúng ở `importCourse` (Task 5, có `await`). ✅
- `resolveReferences` (Task 4) trả `errors[]` object `{ code, message, row, refType, refId }` → khớp cách
  `importCourse` gộp và assert ở Task 6. ✅
- `CourseStructure` + `EXERCISE_TYPES` + `CEFR_LEVELS` + `MODULE_CATEGORIES` (Task 2) khớp import ở service (Task 5, dùng cho validate CEFR/category) và test. ✅
- Field Hướng B (`cefrFrom/cefrTo/goalNote`, `category`, `theory/videoUrl/audioUrl`) đặt tên nhất quán giữa model (Task 2), parser IR (Task 3), `buildDoc`/`validateCourse` (Task 5) và cột Excel (design §3.2). ✅
- `courseImportUpload` (Task 6 middleware) khớp import ở admin router (Task 6). ✅
- `PERMISSIONS.COURSECONTENT_WRITE` = `'coursecontent:write'` (Task 1) khớp `verifyPermission` (Task 6) và test
  seed permission (Task 6). ✅
- Error code strings (`IMPORT_VALIDATION_FAILED`, `PARSE_FAILED`, `EMPTY_COURSE_FILE`, `UNSUPPORTED_EXERCISE_TYPE`,
  `INVALID_FILE_TYPE`) khớp giữa constants (Task 1), service/parser/middleware và assert trong test. ✅

**Ghi chú deviation so với design (đã cân nhắc):**
- Endpoint Import hand-mount thay vì dùng `actions` factory — lý do multer (ghi ở đầu file này + Task 6). Vẫn giữ
  nguyên: admin-api layer, verifyPermission, delegate service, audit thủ công.
- List/read khóa đã import **không** làm ở Phase 1 (không AC nào cần) — chuyển Phase 2 (tiêu thụ).
