# OmniGem Kho & Giá (nội bộ) — PRD

Trạng thái: **DRAFT** — 2026-10-20. Tác giả: brainstorm cùng Product Owner (Leonard). Chờ PO duyệt.
Nguồn: phiên brainstorm 2026-10-20 (bài toán giá vốn/giá sàn cho quyết định giảm giá campaign).
Liên quan: `omni-web/design/adr/ADR-001-backend-topology.md` (đường ghép vào `omnigem-api`), `omni-web/design/adr/ADR-002-contact-channel-and-dynamic-config.md` (band giá 500k–5tr), `omni-web/design/solution-design-epic-a-browse-discover.md` (bảng `PRODUCT` public).

> ⚠️ **Đây là app NỘI BỘ, tách biệt hoàn toàn với OmniGem Gallery (web công khai).** Dữ liệu giá vốn/giá sàn là dữ liệu kinh doanh nhạy cảm nhất — KHÔNG bao giờ xuất hiện trên bề mặt public. Đây không phải PII khách hàng, nhưng được bảo vệ ở mức tương đương (auth bắt buộc, no-store).

---

## 1. Tóm tắt điều hành & Luận chứng kinh doanh

Khi lên campaign hoặc tư vấn giảm giá (ví dụ "20/12 giảm 5–10%"), owner cần biết **ngay** mức giá đó còn trong vùng an toàn không — tức ở giá sau giảm thì **còn lời bao nhiêu, hay đã chạm sàn/lỗ**. Để trả lời được, cần biết **giá nhập (giá vốn)** và **giá sàn** của từng món.

Hiện chỉ có **chị Trang** (nhân viên cửa hàng) nắm các con số này; hỏi từng món rất mất thời gian và không tra được từ xa.

**Giải pháp:** một **app nội bộ nhỏ, kiểu sổ nhập kho + giá**, để chị Trang nhập thông tin món hàng (mã, ảnh, đặc điểm, số lượng, 3 mức giá) tại cửa hàng; owner xem được toàn bộ danh sách (gồm giá vốn & margin) từ nhà và tự ra quyết định giảm giá an toàn.

**Luận chứng kinh doanh:**
- Rút thời gian quyết định campaign từ "gọi hỏi chị Trang từng món" xuống "mở app xem danh sách".
- Tập trung đúng nhu cầu: **không** làm máy mô phỏng giảm giá phức tạp — owner chỉ cần nhìn giá sàn cạnh giá bán là tự biết giảm bao nhiêu còn an toàn.
- **Tận dụng lại về sau:** backend viết thành module `inventory-pricing` có thể ghép thẳng vào `omnigem-api` (ADR-001) khi hệ thống lớn được dựng → không phải làm lại.

**Thesis:** Đây là **sổ giá/kho nội bộ** cho 2 người (chị Trang + owner), KHÔNG phải hệ thống ERP/quản lý kho đầy đủ.

## 2. Mục tiêu & Phi mục tiêu

### Mục tiêu (Goals)
- **G1:** Chị Trang nhập/sửa được món hàng tại cửa hàng bằng điện thoại, có chụp ảnh món.
- **G2:** Owner xem được toàn bộ danh sách hàng (đặc điểm + giá nhập + giá bán + giá sàn + margin + số lượng tồn) từ xa.
- **G3:** Hiển thị margin (giá bán − giá nhập) và giá sàn cạnh nhau để quyết định giảm giá an toàn.
- **G4:** Hỗ trợ cả hàng **độc bản** (mỗi món một giá) lẫn hàng **loạt/model** (nhiều món chung giá).
- **G5:** Backend thiết kế sẵn để ghép vào `omnigem-api` (module `inventory-pricing`) mà không phải viết lại.

### Phi mục tiêu (Non-Goals)
- **NG1:** Không có máy mô phỏng giảm giá tự động / tính campaign hàng loạt (owner tự nhìn số).
- **NG2:** Không có lịch sử thay đổi giá, báo cáo doanh thu/lãi lỗ theo kỳ.
- **NG3:** Không quản lý đa kho/đa chi nhánh.
- **NG4:** Không nhập liệu offline + sync (giả định cửa hàng có mạng).
- **NG5:** Không tích hợp sàn TMĐT / web gallery ở bản này (chỉ chừa `product_id` để map sau).
- **NG6:** Không phải PII khách hàng — không thu dữ liệu khách ở app này.

## 3. Người dùng mục tiêu / Vai trò

1. **Chị Trang (`staff`)** — nhân viên cửa hàng, dùng điện thoại Android tại shop. Nhập/sửa món, chụp ảnh, nhập đủ 3 giá + số lượng. *Giả định (⚠️ chờ PO xác nhận): staff xem + sửa được tất cả món, vì chị Trang vốn là người nắm giá.*
2. **Owner (Leonard, `admin`)** — xem toàn bộ (gồm giá vốn & margin) từ nhà, quản lý tài khoản người dùng. *Giả định (⚠️): owner dùng CHÍNH app Android này để xem, không cần web riêng.*

## 4. Phạm vi

- **Phạm vi hàng hóa:** **toàn bộ hàng trong cửa hàng** (kể cả món chưa lên web). Web catalog (`PRODUCT`) là tập con; món nào lên web thì map qua `product_id` sau.
- **Loại sản phẩm:** nhẫn, vòng, chuỗi hạt, mặt dây (khớp catalog scope đã chốt; không có tượng).
- **Band giá tham chiếu:** 500k–5tr VND (ADR-002).

### 4.1 Quyết định tech stack
- **Backend: Node.js + PostgreSQL**, REST + OpenAPI. Lý do: (a) dễ làm nhanh; (b) nếu `omnigem-api` cũng là Node thì ghép gần như không phải port; (c) Postgres trùng DB đã chốt cho `omnigem-api` (ADR-001).
- **Client: app Kotlin (Android)** — hợp với việc chị Trang dùng điện thoại tại cửa hàng, chụp ảnh món ngay. *Giả định: Jetpack Compose + Retrofit.*
- **Ảnh: Cloudinary** — nhất quán với omni-web (đã chọn Cloudinary) → ghép sau khỏi đổi. *Giả định (⚠️): nếu chưa có Cloudinary, thay bằng object storage/S3-compatible.*
- Client upload ảnh trực tiếp lên Cloudinary bằng signature do server ký (server không proxy file).

## 5. Yêu cầu chức năng — User Stories & Acceptance Criteria

### Epic A — Xác thực & phân quyền

**US-A1 — Đăng nhập**
- AC1: Người dùng đăng nhập bằng username + password; nhận JWT.
- AC2: Mật khẩu lưu dạng hash (bcrypt hoặc argon2), không lưu plaintext.
- AC3: Có rate-limit ở endpoint login (chống brute force).
- AC4: Mọi endpoint dữ liệu yêu cầu JWT hợp lệ; thiếu/sai token → 401.

**US-A2 — Phân quyền theo vai trò (RBAC)**
- AC1: 2 vai trò `staff` và `admin`.
- AC2: `staff` tạo/sửa/xem/ẩn món (ẩn = soft delete, có hoàn tác — xem US-B3; ⚠️ xem giả định §3.1); `admin` làm tất cả + quản lý user.
- AC3: Chỉ `admin` tạo/sửa/xóa tài khoản người dùng; xóa cứng món (nếu có, backlog) cũng chỉ `admin`.

### Epic B — Quản lý món hàng (CRUD)

**US-B1 — Thêm món**
- AC1: Form gồm: mã (bắt buộc, unique), tên, loại món (`unique`/`batch`), loại SP (nhẫn/vòng/chuỗi/mặt dây), loại ngọc/chất liệu, màu, kích thước/size, ghi chú, mệnh (tùy chọn, nhiều giá trị), số lượng tồn, giá nhập, giá bán, giá sàn, ảnh (bắt buộc).
- AC2: Validate schema trước khi lưu; mã trùng → báo lỗi rõ ràng.
- AC3: Món `unique` → số lượng mặc định 1 (còn/hết = 1/0); món `batch` → nhập số nguyên N.
- AC4: Ảnh upload lên Cloudinary; lưu `photo_url`.
- AC5: Lưu `created_by`, `created_at`.

**US-B2 — Sửa món**
- AC1: Sửa được mọi trường của một món; cập nhật `updated_by`, `updated_at`.
- AC2: Đổi mã sang mã đã tồn tại → báo lỗi.

**US-B3 — Ẩn món (xóa mềm)**
- AC1: Ẩn một món là **xóa mềm** (soft delete): đánh dấu `deleted_at`/`deleted_by`, KHÔNG xóa cứng khỏi DB. Bản ghi, ảnh và audit được giữ nguyên (quyết định Q4).
- AC2: Cả `staff` lẫn `admin` đều ẩn được món, vì thao tác an toàn và hoàn tác được (nới so với bản thảo cũ "chỉ admin").
- AC3: Món đã ẩn KHÔNG xuất hiện ở danh sách (US-B4) và không mở được ở chi tiết (US-B5) qua đường thường; mặc định mọi truy vấn đọc lọc `deleted_at IS NULL`.
- AC4: Cử chỉ: **vuốt trái thẻ món** để lộ hành vi Ẩn → **hộp thoại xác nhận** nêu rõ hệ quả → ẩn.
- AC5: Sau khi ẩn hiện **snackbar có nút HOÀN TÁC** trong vài giây; bấm hoàn tác → gỡ `deleted_at` (khôi phục). Hết thời gian snackbar vẫn chỉ là soft delete (chưa có màn khôi phục riêng ở MVP).
- AC6: Soft delete KHÔNG đụng tới ảnh Cloudinary (giữ nguyên để khôi phục chính xác).
- AC7: Quan hệ `product_id` → `PRODUCT` public không bị phá khi ẩn.
- AC8 *(non-MVP, backlog):* màn "Món đã ẩn" cho phép xem lại + khôi phục; xóa cứng vĩnh viễn (chỉ `admin`).

**US-B4 — Danh sách & tìm/lọc (màn hình chính)**
- AC1: Hiển thị danh sách dạng thẻ/hàng: ảnh, mã, tên, loại SP, đặc điểm, số lượng, **giá bán, giá sàn, margin**.
- AC2: Lọc theo loại SP và theo mệnh; tìm theo mã/tên.
- AC3: `admin` (owner) thấy đủ giá nhập + giá sàn + margin. (Hiển thị cho `staff` theo giả định §3.1.)

**US-B5 — Chi tiết món**
- AC1: Xem đầy đủ một món gồm ảnh lớn, mọi đặc điểm, 3 giá và margin tính sẵn.

### Epic C — Tính margin

**US-C1 — Hiển thị margin & vùng an toàn**
- AC1: `margin_vnd = sale_price_vnd − cost_price_vnd`, tính lúc đọc (KHÔNG lưu trong DB).
- AC2: Hiển thị thêm `margin_pct = margin_vnd / sale_price_vnd`.
- AC3: Hiển thị cảnh báo trực quan khi `sale_price_vnd ≤ floor_price_vnd` (chạm/dưới sàn) — ví dụ badge đỏ.

## 6. Mô hình dữ liệu

Bảng chính `inventory_item`:

| Cột | Kiểu | Ghi chú |
|---|---|---|
| id | uuid PK | |
| code | text UNIQUE NOT NULL | mã tự đặt |
| name | text | |
| item_kind | enum `unique`\|`batch` | độc bản / hàng loạt |
| product_type | enum `ring`\|`bracelet`\|`beads`\|`pendant` | loại SP |
| jade_type | text | loại ngọc/chất liệu |
| color | text | |
| size | text | để text cho linh hoạt |
| menh | text[] nullable | Kim/Mộc/Thủy/Hỏa/Thổ (tùy chọn) |
| note | text | ghi chú tự do |
| photo_url | text NOT NULL | ảnh Cloudinary |
| quantity | int NOT NULL | tồn: unique 0/1, batch N |
| **cost_price_vnd** | bigint NOT NULL | **giá nhập (nhạy cảm)** |
| sale_price_vnd | bigint NOT NULL | giá bán niêm yết |
| **floor_price_vnd** | bigint NOT NULL | **giá sàn (nhạy cảm)** |
| product_id | uuid nullable | map tới `PRODUCT` public (để sau) |
| created_at / updated_at | timestamptz | |
| created_by / updated_by | uuid → app_user | audit |
| **deleted_at** | timestamptz nullable | **soft delete: NULL = đang hiện; có giá trị = đã ẩn** (US-B3) |
| **deleted_by** | uuid → app_user nullable | ai đã ẩn món (audit) |

Bảng `app_user`: `id uuid PK`, `username text UNIQUE`, `password_hash text`, `role enum staff|admin`, `created_at`.

Ghi chú:
- Giá VND dùng `bigint` (đơn vị đồng), tránh số thập phân.
- `margin` KHÔNG lưu — tính lúc đọc (tránh dữ liệu trùng/lệch, giữ immutability).
- **Soft delete qua `deleted_at`**: mọi truy vấn đọc mặc định lọc `deleted_at IS NULL`. Nên có partial index `WHERE deleted_at IS NULL` để list nhanh. Ràng buộc `code UNIQUE` cân nhắc là partial unique (`WHERE deleted_at IS NULL`) nếu muốn tái dùng mã sau khi ẩn — mặc định MVP giữ UNIQUE toàn cục cho đơn giản.

## 7. API (REST — sẽ thành module `inventory-pricing` của `omnigem-api`)

- `POST /auth/login` → JWT.
- `GET /inventory/items` — list; query: `product_type`, `menh`, `q` (search code/name); **mặc định chỉ trả món `deleted_at IS NULL`**; trả margin tính sẵn.
- `GET /inventory/items/:id` — món đã ẩn → `404` (coi như không tồn tại qua đường thường).
- `POST /inventory/items` — tạo (validate schema, vd Zod).
- `PATCH /inventory/items/:id` — sửa.
- `DELETE /inventory/items/:id` — **soft delete**: set `deleted_at = now()`, `deleted_by = <user>`; staff + admin đều gọi được; idempotent (ẩn món đã ẩn → vẫn `200/204`).
- `POST /inventory/items/:id/restore` — **hoàn tác**: gỡ `deleted_at`/`deleted_by`; phục vụ nút HOÀN TÁC của snackbar.
- `POST /inventory/photos/sign` — trả Cloudinary signature cho client upload trực tiếp.
- `GET/POST/PATCH/DELETE /users` — quản lý user (admin).
- Response theo envelope nhất quán: `{ success, data, error }`.
- Mọi endpoint dữ liệu: `Cache-Control: no-store`, chỉ HTTPS.

## 8. Bảo mật

- JWT; hash mật khẩu (bcrypt/argon2); rate-limit login.
- Giá nhập/giá sàn chỉ trả qua endpoint có auth; không có bề mặt public nào lộ các cột này.
- Không hardcode secret (Cloudinary key, JWT secret, DB URL) — dùng biến môi trường; validate có mặt lúc khởi động.
- Khi ghép vào `omnigem-api`: module này nằm ở nhóm admin (không phải public), tuân cô lập theo ADR-001 — role DB public KHÔNG được `SELECT` bảng `inventory_item`.

## 9. Đường ghép vào `omnigem-api`/CMS (tương lai)

- Viết backend thành module độc lập `inventory-pricing` (routes + service + repository + schema + migration riêng).
- Khi `omnigem-api` dựng: copy module, trỏ vào Postgres chung, bọc RBAC/auth chung của hệ thống.
- `inventory_item.product_id` liên kết tùy chọn tới `PRODUCT` public: món lên web thì map; giá vốn/sàn vẫn ở bảng nội bộ, không lộ public.
- Điều kiện giữ bất biến: cô lập 3 lớp (route/RBAC/DB-role) như ADR-001.

## 10. Kiểm thử

- **Unit:** logic margin, validate schema, RBAC guard, soft-delete filter (list/detail bỏ `deleted_at IS NOT NULL`).
- **Integration:** CRUD + auth trên Postgres (testcontainer); soft delete → món biến khỏi list & chi tiết 404; restore → món hiện lại; DELETE idempotent.
- **Coverage mục tiêu ≥ 80%** (theo rule chung).

## 11. Câu hỏi mở / Giả định chờ PO xác nhận

- **Q1 (⚠️):** Owner dùng chính app Android để xem (mặc định) hay cần web read-only riêng?
- **Q2 (⚠️):** Ảnh dùng Cloudinary (mặc định) hay S3/khác?
- **Q3 (⚠️):** Quyền `staff`: chị Trang xem + sửa tất cả (mặc định) hay chỉ nhập, không xem giá của món người khác?
- **Q4 — ĐÃ CHỐT (2026-10-25):** Dùng **xóa mềm** (`deleted_at`), KHÔNG xóa cứng ở MVP; cả staff + admin ẩn được; hoàn tác qua snackbar; màn khôi phục + xóa cứng là backlog. Xem US-B3 và `docs/superpowers/specs/2026-10-25-inventory-soft-delete-design.md`.
- **Q5:** `omnigem-api` sẽ là Node thật chứ (ảnh hưởng mức độ "zero-port")?

## 12. Bước tiếp theo
1. PO trả lời Q1–Q3 (các giả định đang dùng mặc định).
2. Chuyển sang lập kế hoạch triển khai (writing-plans): chia phase backend trước (API + DB + auth) rồi app Kotlin.
