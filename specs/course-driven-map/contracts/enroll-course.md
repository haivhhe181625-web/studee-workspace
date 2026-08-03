# Contract: enrollCourse — Auto-append Course to Roadmap

- **Loại:** Cross-repo (exe-web learner → exe-api course-content enroll)
- **Bên cung cấp (provider):** `exe-api` (course-content.controller, progress.service)
- **Bên tiêu thụ (consumer):** `exe-web` (course listing, roadmap explorer CTA)
- **Trạng thái:** Live (spec hồi tố)

## Endpoint

```
POST /api/courses/:slug/enroll
```

- **Auth:** JWT learner token (verifyToken)
- **Permission required:** Không (learner self-only)

## Request

```json
{}
```

Không có body (learner ID từ JWT).

| Field | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| (empty) | — | — | tất cả state từ JWT + path |

## Response — thành công

```json
{
  "enrollment": {
    "id": "507f1f77bcf86cd799439012",
    "userId": "507f1f77bcf86cd799439013",
    "courseId": "507f1f77bcf86cd799439014",
    "courseSlug": "ielts-5-6",
    "status": "active",
    "enrolledAt": "2026-08-03T12:00:00Z"
  },
  "segment": {
    "courseSlug": "ielts-5-6",
    "courseMapVersion": 1,
    "appendedAt": "2026-08-03T12:00:00Z",
    "status": "active"
  }
}
```

| Field | Kiểu | Ghi chú |
|---|---|---|
| `enrollment.id` | String (ObjectId) | CourseEnrollment._id; track learner's enroll history |
| `enrollment.userId` | String | từ JWT |
| `enrollment.courseId` | String | CourseStructure._id |
| `enrollment.courseSlug` | String | course.slug (denorm) |
| `enrollment.status` | String | 'active' (lúc enroll); future 'completed', 'archived' |
| `enrollment.enrolledAt` | Date | ISO timestamp |
| `segment.courseSlug` | String | = course.slug; segment nối vào LearnerPath |
| `segment.courseMapVersion` | Number | version pin (lúc enroll resolve template version) |
| `segment.appendedAt` | Date | lúc append (may khác enrolledAt nếu non-blocking delay) |
| `segment.status` | String | 'active' (newly appended) |

## Response — lỗi

| Mã HTTP | error code | Khi nào |
|---|---|---|
| 400 | INVALID_COURSE | course slug không tồn tại |
| 400 | COURSE_NOT_PUBLISHED | course status ≠ 'published' |
| 409 | ALREADY_ENROLLED | learner đã enroll khóa này (idempotent → return 200 + existing enrollment) |
| 409 | NEEDS_ASSESSMENT | **precondition placement**: learner chưa StudentModel; CTA client "làm bài kiểm tra" (optional; system có thể skip check) |
| 403 | NOT_ALLOWED | learner disabled / center access restriction (future) |
| 500 | APPEND_FAILED | appendCourseSegment fail (non-blocking; enroll VẪN succeed, segment missing) |
| 401 | UNAUTHORIZED | no JWT or invalid token |

## Precondition & Idempotency

### Placement Check

- **Hard gate** (recommended): nếu learner không StudentModel → 409 NEEDS_ASSESSMENT trước enroll.
- **Soft gate** (current): enroll succeed; appendCourseSegment check StudentModel lúc append (bỏ qua); getMap kế tiếp sẽ throw 409 lúc start node.

**Document decision**: spec chọn **soft gate** (enroll idempotent; getMap sẽ catch).

### Duplicate Enroll

- Enroll 2 lần **cùng course**: HTTP 200 (idempotent); return existing enrollment (KHÔNG duplicate segment).
- Check: `LearnerPath.segments.find(slug)` trước append → skip nếu exist.

---

## Sequencing triển khai

1. **BE**: course-content.controller POST /api/courses/:slug/enroll; progress.service.enroll() + appendCourseSegment (non-blocking).
2. **FE**: enroll button ("/courses/:slug") → POST /api/courses/:slug/enroll → on success → navigate to /adaptive (getMap auto-resolve).

---

## Async Side Effects

### appendCourseSegment (Non-blocking)

- **Timeout**: 3s (fallthrough if exceed; enroll response 200 regardless).
- **Retry**: exponential backoff 1s → 2s → 4s; give up after 3 tries.
- **Failure recovery**: getMap/ensureActivePath reconcile segment missing → auto-bootstrap.

```javascript
// pseudo-code
enroll(userId, slug) {
  courseEnrollment = createEnrollment(userId, slug);
  
  // non-blocking
  appendCourseSegment(userId, slug)
    .timeout(3s)
    .retry(3)
    .catch(err => {
      log.warn('append failed', err);
      // DO NOT propagate; enroll success regardless
      // ensureActivePath will fix on next getMap call
    });
  
  return courseEnrollment; // HTTP 200 immediate
}
```

---

## Ví dụ gọi thực tế

```bash
# Learner enroll khóa IELTS 5-6
curl -X POST http://localhost:3000/api/courses/ielts-5-6/enroll \
  -H "Authorization: Bearer <JWT>" \
  -H "Content-Type: application/json" \
  -d '{}'

# Response (success)
{
  "enrollment": {
    "id": "507f...",
    "userId": "507g...",
    "courseId": "507h...",
    "courseSlug": "ielts-5-6",
    "status": "active",
    "enrolledAt": "2026-08-03T12:00:00Z"
  },
  "segment": {
    "courseSlug": "ielts-5-6",
    "courseMapVersion": 1,
    "appendedAt": "2026-08-03T12:00:00Z",
    "status": "active"
  }
}

# Re-enroll (idempotent)
curl -X POST http://localhost:3000/api/courses/ielts-5-6/enroll \
  -H "Authorization: Bearer <JWT>" \
  -d '{}'

# Response (same as above, no duplicate)
{
  "enrollment": { ... }
}

# Error: course not published
curl -X POST http://localhost:3000/api/courses/draft-course/enroll \
  -H "Authorization: Bearer <JWT>"

# Response 400
{
  "code": "COURSE_NOT_PUBLISHED",
  "message": "Course must be published to enroll"
}

# Error: needs placement (soft gate)
curl -X POST http://localhost:3000/api/courses/ielts-5-6/enroll \
  -H "Authorization: Bearer <JWT>"

# Response 200 (segment append in background; getMap will catch need-assessment)
{
  "enrollment": { ... },
  "segment": { "status": "pending_append" } // or similar indicator
}
```

---

## Usage Context (FE)

```typescript
// CourseCard.tsx
const handleEnroll = async (slug: string) => {
  try {
    const result = await enrollCourse(slug);
    toast.success('Ghi danh thành công');
    
    // Navigate to map
    navigate('/adaptive');
    
    // getMap will trigger; if needs-assessment → show CTA
  } catch (err) {
    if (err.code === 'NEEDS_ASSESSMENT') {
      toast.info('Hãy hoàn thành bài kiểm tra trước');
      navigate('/assessment/intake');
    } else {
      toast.error('Ghi danh thất bại');
    }
  }
};
```

---

## Notes

### Why Non-blocking Append?

- **User UX**: enroll response fast (≤100ms); không wait segment append (KHÔNG block 3s MongoDB).
- **Failure resilience**: if append fail → getMap/ensureActivePath auto-repair (reconcile enrollment↔segment).

### Race Condition: Concurrent Enroll 2 Courses

```
Learner POST /api/courses/A + POST /api/courses/B concurrently
├─ appendCourseSegment(A) → E11000 duplicate? NO, different course
├─ appendCourseSegment(B) → E11000 duplicate? NO, different course
└─ $push A + $push B happen; both segment persist (order by appendedAt)
```

- **Verify**: test concurrent POST; DB assert 2 segment present; getMap order maintained.

### Mapping: CourseStructure → CourseMapTemplate → Segment

```
Course /{slug}
  ├─ POST /api/courses/{slug}/enroll
  │   └─ resolve CourseStructure by slug
  │       └─ resolve CourseMapTemplate (latest published version)
  │           └─ create segment {courseMapVersion:N}
  │               └─ LearnerPath.segments.$push
  │                   └─ seed nodeStates from CourseMapTemplate.nodes
```

### Reconcile Logic (ở getMap)

```javascript
ensureActivePath(userId) {
  path = LearnerPath.findOne(userId);
  enrollments = CourseEnrollment.find(userId, status:'active');
  
  // diff: enrollment exist but not in path.segments
  missing = enrollments.filter(e => !path.segments.some(s => s.courseSlug === e.courseSlug));
  
  if (missing.length) {
    // append those segments (recover from append-crash)
    missing.forEach(e => appendSegmentOcc(path, e.courseSlug));
    path.save();
  }
}
```
