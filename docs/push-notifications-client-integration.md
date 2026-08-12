# Tích hợp Push Notification — Web · Android · iOS

> Hướng dẫn client (exe-web, mobile) tích hợp thông báo đẩy với BE `exe-api`. Áp dụng cho
> mọi tính năng dùng push (nhắc học M4-5, streak, SRS due, flashcard...), không riêng lịch học.
> Transport dùng chung: Firebase Cloud Messaging (FCM) — một token/thiết bị, BE multicast tới
> mọi token của user trong 1 lần gửi.

## 1. Tổng quan luồng

```
Client (web/android/ios)
  1. Xin quyền thông báo (OS/browser)
  2. Lấy FCM registration token (SDK Firebase)
  3. POST /users/me/fcm-token { token, platform }      ── đăng ký token lên BE
        │
        ▼
BE exe-api: lưu vào User.fcmTokens[] { token, platform, addedAt }
        │
Tính năng (vd reminder) → addPushUserJob(userId, notification, data)
        │  notification.worker: nạp User.fcmTokens → fcm.adapter.sendMulticast
        ▼
Google FCM ── gửi tới từng thiết bị (web qua VAPID, android trực tiếp, ios qua APNs)
        │
Client hiển thị (background: service worker / OS; foreground: handler của app)
```

- **1 tin = 1 multicast** tới **tất cả** `fcmTokens` của user (web + android + ios cùng lúc).
- Token invalid (`registration-token-not-registered`...) được `notification.worker` **tự dọn** khỏi `User.fcmTokens`.

## 2. API đăng ký / gỡ token (dùng chung mọi nền tảng)

Cả hai đều yêu cầu đăng nhập (`verifyToken` — gửi kèm JWT của BE ở header `Authorization: Bearer <token>`).

### `POST /users/me/fcm-token` — đăng ký token thiết bị hiện tại
```json
{ "token": "<FCM registration token>", "platform": "web" | "android" | "ios" }
```
- `token`: chuỗi 20–4096 ký tự (bắt buộc).
- `platform`: `web` | `android` | `ios` (bắt buộc).
- Gọi **sau khi login** và **mỗi khi token refresh**. BE upsert vào `User.fcmTokens[]` (idempotent theo token).

### `DELETE /users/me/fcm-token` — gỡ token (tắt nhắc / logout)
```json
{ "token": "<FCM registration token>" }
```
- Gọi khi user tắt thông báo hoặc logout. Nên gỡ ở **cả** FCM SDK (client) lẫn BE.

## 3. Prefs liên quan (bật/tắt mềm + múi giờ)

Qua `PUT /adaptive/schedule/prefs` (body linh hoạt — gửi field nào cập nhật field đó):

| Field | Kiểu | Ý nghĩa |
|---|---|---|
| `notificationPrefs.study` | `boolean` (top-level, **không** trong `studyPlan`) | Tắt/bật nhắc học ở **cấp server**. `null`/vắng = **mặc định bật**; chỉ `false` mới tắt (dù vẫn còn token). |
| `studyPlan.timezone` | `string` (IANA, vd `"Asia/Saigon"`) | Múi giờ tính giờ nhắc. **Client nên gửi** khi mở app/đăng nhập. Thiếu → BE fallback `Asia/Saigon` (nhắc sai giờ với user lệch múi giờ). |

> Phân biệt 2 mức tắt nhắc: **cấp thiết bị** = gỡ token (`DELETE fcm-token`); **cấp server** =
> `notificationPrefs.study=false` (giữ token nhưng BE không gửi). Client chọn UX phù hợp.

## 4. Payload BE gửi (client parse để hiển thị + điều hướng)

```jsonc
{
  "notification": { "title": "Luyện tập hôm nay", "body": "Hôm nay bạn có N bài cần hoàn thành" },
  "data": { "type": "study-reminder", "slot": "morning" | "evening", "route": "/schedule" }
}
```
- `data.route` = màn client mở khi chạm vào tin (deep-link).
- FCM yêu cầu `data` toàn giá trị **string**.

## 5. Tích hợp theo nền tảng

### 5.1 Web (exe-web)
**Prerequisite:** `NEXT_PUBLIC_FIREBASE_VAPID_KEY` (Firebase Console → Project Settings → Cloud
Messaging → **Web Push certificates** → Key pair) + service worker `public/firebase-messaging-sw.js`.

**Luồng** (`src/services/firebase-messaging.service.ts` + `src/hooks/use-push-notifications.ts`):
1. `isPushSupported()` — cần `Notification` + `serviceWorker` + **VAPID key đã cấu hình**.
2. `Notification.requestPermission()` → phải `granted`.
3. `getToken(messaging, { vapidKey, serviceWorkerRegistration })`.
4. `POST /users/me/fcm-token { token, platform: "web" }`, lưu token vào `localStorage` để lúc tắt biết gỡ token nào.

**Hiển thị:**
- **Background** (tab không focus/đóng): SW `onBackgroundMessage` → `showNotification` — hoạt động sẵn.
- **Foreground** (đang mở tab): ⚠️ **hiện chưa có handler `onMessage`** → tin **không hiện** khi user đang ở trong app. Cần thêm in-app toast/badge (xem §7).

**Deep-link web:** ⚠️ `fcm.adapter` hiện chỉ gửi `{notification, data}`. Muốn click-through chuẩn trên
web cần BE thêm `webpush.fcmOptions.link` (xem §7); hiện web dựa vào `notificationclick` trong SW (đã có, mở theo `data.url`).

### 5.2 Android
**Prerequisite:** `google-services.json` trong app; FCM SDK.

**Luồng:**
1. (Android 13+) xin runtime permission `POST_NOTIFICATIONS`.
2. `FirebaseMessaging.getInstance().token` (Kotlin) / `messaging().getToken()` (RN).
3. `POST /users/me/fcm-token { token, platform: "android" }`.
4. Nhận tin: background → hệ thống tự hiện (payload `notification`); foreground → `onMessageReceived` tự hiển thị.
5. Điều hướng theo `data.route` khi user chạm tin.

**BE:** không cần thay đổi — FCM gửi thẳng.

### 5.3 iOS
**Prerequisite (quan trọng):** upload **APNs Auth Key (`.p8`)** lên Firebase Console → Project Settings
→ Cloud Messaging → **Apple app configuration**. Kèm `GoogleService-Info.plist` trong app. **Không có
APNs key thì iOS không nhận được tin** dù BE gửi thành công.

**Luồng:**
1. Xin quyền qua `UNUserNotificationCenter.requestAuthorization`.
2. Đăng ký APNs (`registerForRemoteNotifications`), Firebase tự map APNs token ↔ FCM token.
3. `Messaging.messaging().token` → lấy FCM token.
4. `POST /users/me/fcm-token { token, platform: "ios" }`.
5. Điều hướng theo `data.route`.

**BE:** không cần thay đổi — FCM lo phần APNs. iOS chỉ phụ thuộc APNs key trong Firebase project (prerequisite hạ tầng, không phải code).

## 6. Prerequisite tổng hợp

| Nền tảng | Config bắt buộc | Lấy ở đâu |
|---|---|---|
| **BE (chung)** | `FIREBASE_PROJECT_ID`, `FIREBASE_CLIENT_EMAIL`, `FIREBASE_PRIVATE_KEY` | Firebase Console → Service Accounts → Generate private key |
| **Web** | `NEXT_PUBLIC_FIREBASE_VAPID_KEY` + Firebase web config (`NEXT_PUBLIC_FIREBASE_*`) | Cloud Messaging → Web Push certificates |
| **Android** | `google-services.json` | Firebase Console → thêm Android app |
| **iOS** | APNs Auth Key `.p8` (upload lên Firebase) + `GoogleService-Info.plist` | Apple Developer → Keys; Firebase → thêm iOS app |

## 7. Khoảng trống đã biết (backlog)

- **Web foreground `onMessage`:** thiếu handler → đang mở app không thấy nhắc. Nên thêm in-app toast/badge.
- **Web deep-link click-through:** cân nhắc BE thêm `webpush.fcmOptions.link` trong `fcm.adapter` để chuẩn hóa link mở.
- **`studyPlan.timezone` chưa được client persist:** exe-web hiện chỉ gửi `tz` làm query param cho "hôm nay",
  chưa lưu vào profile → reminder fallback `Asia/Saigon`. Cần client `PUT prefs { studyPlan: { timezone } }`
  khi mở app/đăng nhập để hỗ trợ đa múi giờ.

## 8. Nguồn (BE — exe-api)

- Endpoint token: `services/api/src/modules/user/user.routes.js` (`POST|DELETE /me/fcm-token`), schema `user.schema.js` (`addFcmToken`/`removeFcmToken`).
- Transport: `services/api/src/adapters/fcm.adapter.js` (`sendMulticast`) + `firebase.adapter.js` (init) + `queues/notification.worker.js` (multicast + dọn token chết).
- Prefs: `services/api/src/modules/adaptive/adaptive.routes.js` (`PUT /adaptive/schedule/prefs`) → contract `exe-api/docs/api/adaptive-schedule-fe-contract.md` §5.
- Reminder gửi tin: `services/api/src/modules/adaptive/schedule/reminder.service.js`.
- Client web: `exe-web/src/services/firebase-messaging.service.ts`, `src/hooks/use-push-notifications.ts`, `public/firebase-messaging-sw.js`.
