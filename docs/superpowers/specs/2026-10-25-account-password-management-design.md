# Thiết kế: Quản lý mật khẩu (đổi MK tự phục vụ + Owner reset)

Trạng thái: **APPROVED** — 2026-10-25. Nguồn: brainstorm 2026-10-25.
App: `gem-inventory` (Fastify + pg backend, React/Vite PWA). Mockup: https://claude.ai/artifact/RKfmo1eVVgU6RsqDcwq2kV

## 1. Bối cảnh
App nội bộ 2 người, **không có hạ tầng email/SMS**, bảng `app_user` không có cột email. Chị Trang (staff) cần tự đổi mật khẩu; nếu quên và bị khoá ngoài, Owner (admin) đặt lại hộ. Hiện backend chỉ có `POST /auth/login` (argon2, JWT). `requireRole('admin')` đã có trong `guard.ts`.

## 2. Quyết định
| # | Quyết định | Lý do |
|---|---|---|
| D1 | Đổi MK tự phục vụ (đang đăng nhập): nhập MK cũ + mới | Chị Trang quản lý acc của mình, an toàn |
| D2 | Owner (admin) reset MK cho user khác (không cần MK cũ) | Giải quyết "quên" mà không cần email |
| D3 | Không làm reset tự phục vụ qua email/OTP | Không có hạ tầng, thừa cho 2 người |
| D4 | Không thu hồi JWT khi reset (stateless) | Chấp nhận cho MVP; MK mới áp dụng lần đăng nhập sau |
| D5 | MK mới ≥ 8 ký tự; xác nhận nhập lại ở client; lỗi chung không lộ chi tiết | Bảo mật cơ bản |

## 3. Backend (mở rộng module `auth` + bề mặt `users`)
- Repository thêm: `findById(id)` (lấy hash+role), `updatePasswordHash(id, hash)`, `listUsers()` (id, username, role, created_at — KHÔNG trả hash).
- Service: `changePassword(userId, current, new)` — verify `current` (argon2); sai → `InvalidPasswordError`; đúng → hash+update. `resetPassword(targetId, new)` (admin). `listUsers()`.
- Schema Zod: `changePasswordSchema { currentPassword: min1, newPassword: min8 }`; `resetPasswordSchema { newPassword: min8 }`.
- Routes:
  - `PATCH /auth/password` (requireAuth) — body `{currentPassword,newPassword}`; sai MK cũ → `400` (lỗi chung "Mật khẩu hiện tại không đúng"); OK → `204`.
  - `GET /users` (requireAuth + requireRole('admin')) — `{success,data:[{id,username,role,createdAt}]}`.
  - `PATCH /users/:id/password` (requireAuth + requireRole('admin')) — body `{newPassword}`; user không tồn tại → `404`; OK → `204`.
- Tests (integration, Postgres testcontainer, theo harness hiện có): đổi MK thành công rồi login bằng MK mới; MK cũ sai → 400; non-admin gọi GET /users hoặc reset → 403; admin reset rồi login MK mới OK; reset user lạ → 404; validate < 8 ký tự → 400; không route nào trả `password_hash`.

## 4. Web PWA
- Route mới `/account` (RequireAuth) → `AccountPage`:
  - Card user hiện tại (username + role: "Nhân viên"/"Chủ cửa hàng").
  - **Đổi mật khẩu**: form MK cũ / MK mới / nhập lại → `PATCH /auth/password`; toast thành công; sai MK cũ hoặc nhập lại không khớp → lỗi inline.
  - Nếu `user.role === 'admin'`: mục **Người dùng** → `GET /users`; mỗi user khác có nút **"Đặt lại MK"** → dialog (MK mới + nhập lại) → `PATCH /users/:id/password`; toast.
  - Nút **Đăng xuất** (gọi `signOut` của AuthContext).
- `api/auth.ts`: `changePassword(current,new)`. `api/users.ts` (mới): `listUsers()`, `resetUserPassword(id,new)`.
- Lối vào: icon tài khoản ở header ListPage → `navigate('/account')`. Thêm `PersonIcon` vào `icons.tsx`. Route khai báo trong `App.tsx`.
- Dùng lại `ConfirmDialog`/`Snackbar` + tokens hiện có; không tạo design system mới.

## 5. Ngoài phạm vi (backlog)
Email/OTP reset; thu hồi phiên sau reset; ép đổi MK lần đăng nhập đầu; tạo/xoá user, đổi vai trò (UC-07 còn lại).
