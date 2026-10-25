# Thiết kế: Ẩn món trang sức (xóa mềm) — UC-06 / US-B3

Trạng thái: **APPROVED** — 2026-10-25. Nguồn: phiên brainstorm 2026-10-25 (OmniGem Kho & Giá).
Liên quan: `inventory-app/prd/PRD.md` (US-B3, §6 data model, §7 API), `inventory-app/usecase/use-cases.md` (UC-06).
Mockup UX: https://claude.ai/artifact/W6ptzcPSXcLtJaokh1LyTx

## 1. Bối cảnh & vấn đề

App nội bộ "Kho & Giá" cần cho phép **xóa món khỏi danh sách**. Câu hỏi mở Q4 của PRD: xóa mềm (ẩn, giữ vết) hay xóa cứng? Món trang sức là hàng giá trị cao, có audit (`created_by`/`updated_by`) và có thể map tới `PRODUCT` public qua `product_id` → mất vết do xóa nhầm là rủi ro thật, trong khi hệ thống chỉ 2 người dùng nên lỗi thao tác dễ xảy ra.

## 2. Quyết định đã chốt

| # | Quyết định | Lý do |
|---|---|---|
| D1 | **Xóa mềm** (`deleted_at`), không xóa cứng ở MVP | Giữ audit + ảnh + link `product_id`; khôi phục được |
| D2 | **Cả staff + admin** đều ẩn được | Thao tác an toàn, có xác nhận + hoàn tác; chị Trang là người nắm hàng |
| D3 | Cử chỉ **vuốt trái thẻ → xác nhận → ẩn** | Nhanh trên điện thoại, khó bấm nhầm hơn vuốt-xoá-thẳng |
| D4 | **Snackbar HOÀN TÁC** vài giây sau khi ẩn | Khôi phục tức thời, không cần màn thùng rác ở MVP |
| D5 | **Giữ nguyên ảnh Cloudinary** khi ẩn | Khôi phục chính xác; chỉ dọn ảnh khi xóa cứng (backlog) |
| D6 | **MVP chưa có màn "Món đã ẩn" / restore screen / xóa cứng** | Thu hẹp phạm vi; đưa vào backlog |

## 3. Thay đổi dữ liệu

Bảng `inventory_item` thêm 2 cột:

| Cột | Kiểu | Ghi chú |
|---|---|---|
| `deleted_at` | timestamptz nullable | NULL = đang hiện; có giá trị = đã ẩn |
| `deleted_by` | uuid → app_user nullable | ai đã ẩn (audit) |

- Partial index đề xuất: `CREATE INDEX ... ON inventory_item (...) WHERE deleted_at IS NULL` để list nhanh.
- `code UNIQUE`: MVP giữ unique toàn cục (đơn giản). Nếu sau này cần tái dùng mã sau khi ẩn → chuyển partial unique `WHERE deleted_at IS NULL`.

## 4. Hợp đồng API

| Method | Path | Hành vi |
|---|---|---|
| GET | `/inventory/items` | Chỉ trả `deleted_at IS NULL` (mặc định) |
| GET | `/inventory/items/:id` | Món đã ẩn → `404` |
| DELETE | `/inventory/items/:id` | Soft delete: set `deleted_at=now()`, `deleted_by=<user>`; staff+admin; **idempotent** |
| POST | `/inventory/items/:id/restore` | Gỡ `deleted_at`/`deleted_by`; phục vụ nút HOÀN TÁC |

Mọi endpoint: envelope `{ success, data, error }`, `Cache-Control: no-store`, HTTPS, cần JWT.

## 5. Luồng UX (client Android)

1. **Vuốt trái thẻ món** → lộ nền đỏ "⌫ Ẩn".
2. Thả tay → **hộp thoại xác nhận** nêu rõ: ẩn khỏi danh sách, giữ ảnh/giá/lịch sử, hoàn tác được. Nút chính "Ẩn món" màu cảnh báo.
3. Xác nhận → gọi `DELETE`, món **biến khỏi danh sách tức thì** (optimistic update).
4. **Snackbar** "Đã ẩn <mã> · <tên>" + nút **HOÀN TÁC** (vài giây) → gọi `restore`, món hiện lại.
5. Nếu API lỗi → rollback optimistic, hiện lại thẻ + báo lỗi.

## 6. Kiểm thử

- **Unit:** guard soft-delete filter (list/detail loại `deleted_at IS NOT NULL`); RBAC cho phép staff+admin gọi DELETE/restore.
- **Integration (Postgres testcontainer):** ẩn → mất khỏi list & chi tiết 404; restore → hiện lại; DELETE idempotent; ảnh Cloudinary không đổi.
- **Client:** optimistic remove + rollback khi API lỗi; snackbar undo gọi đúng `restore`.
- Coverage mục tiêu ≥ 80% (rule chung).

## 7. Ngoài phạm vi (backlog)

- Màn "Món đã ẩn" (xem lại + khôi phục có chủ đích).
- Xóa cứng vĩnh viễn (chỉ admin) + dọn ảnh Cloudinary.
- Partial-unique cho `code` để tái dùng mã.

## 8. Phụ thuộc (quan trọng)

Tại thời điểm chốt (2026-10-25) repo **chưa có code** — backend `inventory-pricing`, app Android và schema DB đều chưa tồn tại. Spec này là thiết kế cho feature; để implement cần dựng nền app trước (hoặc chờ `omnigem-api`). Không có hạ tầng/credentials AWS, và AWS chưa nằm trong kiến trúc PRD (đang Node + PostgreSQL + Cloudinary) — chọn deploy target là quyết định riêng chưa được thực hiện.
