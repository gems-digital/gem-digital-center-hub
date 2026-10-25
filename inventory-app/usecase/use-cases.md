# OmniGem Kho & Giá — Tài liệu Use Case

Trạng thái: **DRAFT** — 2026-10-21 (cập nhật 2026-10-25: UC-06 ẩn món / xóa mềm, mở quyền staff). Nguồn: `inventory-app/prd/PRD.md` (Epic A, B, C).
Mục đích: cái nhìn tổng quan *ai (actor) làm gì (use case)* với app nội bộ, phân rã use case lớn → use case con, kèm quan hệ `«include»` (bắt buộc) / `«extend»` (có điều kiện).

> Ghi chú: Mermaid không có sơ đồ UML use-case riêng — ở đây mô phỏng bằng `flowchart`: actor là hình tròn, use case là hình bo tròn, use case con màu cam.

---

## 1. Phạm vi & triết lý

App nội bộ = **sổ nhập kho + giá** cho 2 người. **Chỉ có một boundary**: toàn bộ hệ thống nằm sau đăng nhập — không có bề mặt công khai. Mọi use case đọc/ghi dữ liệu giá đều **include** bước kiểm tra quyền (RBAC). Dữ liệu giá nhập/giá sàn là nhạy cảm → không bao giờ lộ ra ngoài.

Mục tiêu nghiệp vụ cốt lõi của owner — *"giảm X% còn an toàn không"* — **không** là một use case tính toán riêng, mà đạt được bằng việc **xem danh sách/chi tiết** có hiển thị giá sàn cạnh giá bán + margin (UC-02 / UC-03).

## 2. Boundary (ranh giới hệ thống)

| Boundary | Hệ thống kỹ thuật | Actor tương tác | Đặc tính |
|---|---|---|---|
| **Internal (duy nhất)** | App Kotlin (Android) + Node.js API + PostgreSQL | Chị Trang, Owner | Có xác thực, phân quyền (staff/admin), dữ liệu giá nhạy cảm, no-store |

## 3. Danh sách Actor

### Actor chính (Primary)
| Actor | Vai trò hệ thống | Mô tả |
|---|---|---|
| **Chị Trang** (Staff) | `staff` | NV cửa hàng; nhập/sửa món, chụp ảnh, nhập 3 giá + số lượng tại shop (điện thoại) |
| **Owner** (Leonard) | `admin` | Xem toàn bộ danh sách + margin từ xa để quyết định giảm giá; quản lý tài khoản |

### Actor phụ / hệ thống ngoài (Secondary / External)
**Cloudinary** — lưu & phục vụ ảnh món hàng (client upload trực tiếp bằng signature server ký).

> ⚠️ Giả định đang dùng (PRD §11): owner dùng chính app Android; `staff` xem + sửa tất cả. Nếu đổi, phân quyền UC-02/UC-06/UC-07 sẽ chỉnh theo.

---

## 4. Sơ đồ Use Case TỔNG QUAN

```mermaid
flowchart TB
    staff(("Chị Trang<br/>(staff)")):::actor
    owner(("Owner<br/>(admin)")):::actor
    cloud(("Cloudinary<br/>(ngoài)")):::ext

    subgraph INT["BOUNDARY — INTERNAL (sau đăng nhập)"]
      direction TB
      uc_auth(["UC-01<br/>Đăng nhập"]):::uc
      uc_list(["UC-02<br/>Xem danh sách & lọc hàng"]):::uc
      uc_detail(["UC-03<br/>Xem chi tiết món"]):::uc
      uc_add(["UC-04<br/>Thêm món"]):::uc
      uc_edit(["UC-05<br/>Sửa món"]):::uc
      uc_del(["UC-06<br/>Ẩn món (xóa mềm)"]):::uc
      uc_user(["UC-07<br/>Quản lý người dùng"]):::uc
      uc_rbac(["UC-08<br/>Kiểm tra quyền (RBAC) — cross-cutting"]):::ucx
    end

    staff --> uc_auth & uc_list & uc_detail & uc_add & uc_edit & uc_del
    owner --> uc_auth & uc_list & uc_detail & uc_add & uc_edit & uc_del & uc_user

    uc_add -. "upload ảnh" .-> cloud
    uc_edit -. "upload ảnh" .-> cloud
    uc_detail -. "tải ảnh" .-> cloud

    uc_list -. include .-> uc_rbac
    uc_add -. include .-> uc_rbac
    uc_edit -. include .-> uc_rbac
    uc_del -. include .-> uc_rbac
    uc_user -. include .-> uc_rbac

    classDef actor fill:#e3f2fd,stroke:#1565c0;
    classDef ext fill:#eceff1,stroke:#455a64;
    classDef uc fill:#f3e5f5,stroke:#6a1b9a;
    classDef ucx fill:#ede7f6,stroke:#6a1b9a,stroke-dasharray:4 3;
```

**Đọc sơ đồ:** actor nằm ngoài boundary. UC-08 (RBAC) vẽ nét đứt vì là chức năng cross-cutting, `«include»` vào mọi use case đọc/ghi. Cloudinary là hệ thống ngoài cho ảnh. `staff` và `owner` dùng chung phần lớn use case — **gồm cả UC-06 (ẩn món / xóa mềm)**, vì thao tác an toàn & hoàn tác được; chỉ **UC-07 (quản lý user)** còn giới hạn `admin`. (Xóa cứng vĩnh viễn — nếu làm ở bản sau — mới giới hạn `admin`.)

---

## 5. Phân rã Use Case chi tiết

### UC-01 — Đăng nhập (Epic A)

```mermaid
flowchart TB
    a(("Chị Trang<br/>/ Owner")):::actor
    subgraph G1["Xác thực"]
      u1(["UC-01 Đăng nhập"]):::uc
      u11(["UC-01.1 Nhập username + password"]):::sub
      u12(["UC-01.2 Cấp JWT (hết hạn theo cấu hình)"]):::sub
      u13(["UC-01.3 Rate-limit chống dò mật khẩu"]):::sub
      u14(["UC-01.4 Lưu phiên trên thiết bị (DataStore)"]):::sub
    end
    a --> u1
    u1 -. include .-> u11 & u12 & u13
    u1 --> u14
    classDef actor fill:#e3f2fd,stroke:#1565c0;
    classDef uc fill:#f3e5f5,stroke:#6a1b9a;
    classDef sub fill:#fff3e0,stroke:#ef6c00;
```

### UC-02..03 — Xem danh sách & chi tiết (Epic B/C) — màn hình chính của owner

```mermaid
flowchart TB
    a(("Chị Trang<br/>/ Owner")):::actor
    subgraph G2["Tra cứu hàng"]
      u2(["UC-02 Xem danh sách hàng"]):::uc
      u21(["UC-02.1 Lọc theo loại SP<br/>(nhẫn/vòng/chuỗi/mặt dây)"]):::sub
      u22(["UC-02.2 Lọc theo mệnh (ngũ hành)"]):::sub
      u23(["UC-02.3 Tìm theo mã / tên"]):::sub
      u24(["UC-02.4 Xem giá bán · giá sàn · margin<br/>(quyết định giảm giá)"]):::sub
      u25(["UC-02.5 Cảnh báo chạm/dưới sàn (badge đỏ)"]):::sub

      u3(["UC-03 Xem chi tiết món"]):::uc
      u31(["UC-03.1 Xem ảnh lớn + đặc điểm đầy đủ"]):::sub
      u32(["UC-03.2 Xem 3 giá + margin tính sẵn"]):::sub
    end
    a --> u2 & u3
    u2 --> u21 & u22 & u23 & u24
    u2 -. extend .-> u25
    u3 --> u31 & u32
    u24 -. "mở 1 món" .-> u3
    u32 -. include .-> calc(["Tính margin = giá bán − giá nhập (lúc đọc)"]):::inc
    u24 -. include .-> calc
    classDef actor fill:#e3f2fd,stroke:#1565c0;
    classDef uc fill:#f3e5f5,stroke:#6a1b9a;
    classDef sub fill:#fff3e0,stroke:#ef6c00;
    classDef inc fill:#e8f5e9,stroke:#2e7d32;
```

### UC-04..05 — Thêm / Sửa món (Epic B)

```mermaid
flowchart TB
    s(("Chị Trang<br/>/ Owner")):::actor
    subgraph G3["Nhập liệu"]
      u4(["UC-04 Thêm món"]):::uc
      u41(["UC-04.1 Nhập mã (unique) + tên"]):::sub
      u42(["UC-04.2 Chọn loại món (độc bản/hàng loạt)<br/>+ số lượng tồn"]):::sub
      u43(["UC-04.3 Nhập đặc điểm<br/>(loại SP, ngọc, màu, size, mệnh, ghi chú)"]):::sub
      u44(["UC-04.4 Nhập 3 giá<br/>(giá nhập · giá bán · giá sàn)"]):::sub
      u45(["UC-04.5 Chụp / chọn ảnh món"]):::sub

      u5(["UC-05 Sửa món"]):::uc
      u51(["UC-05.1 Cập nhật đặc điểm / giá / số lượng"]):::sub
    end
    sign(["«include» Ký upload Cloudinary"]):::inc
    valid(["«include» Validate schema<br/>(mã trùng → báo lỗi)"]):::inc

    s --> u4 & u5
    u4 --> u41 & u42 & u43 & u44 & u45
    u4 -. include .-> valid
    u45 -. include .-> sign
    u5 --> u51
    u5 -. include .-> valid
    u5 -. extend .-> u45
    classDef actor fill:#e3f2fd,stroke:#1565c0;
    classDef uc fill:#f3e5f5,stroke:#6a1b9a;
    classDef sub fill:#fff3e0,stroke:#ef6c00;
    classDef inc fill:#e8f5e9,stroke:#2e7d32;
```

### UC-06 — Ẩn món (xóa mềm) — staff + admin

```mermaid
flowchart TB
    s(("Chị Trang<br/>/ Owner")):::actor
    subgraph G4a["Ẩn món (soft delete)"]
      u6(["UC-06 Ẩn món"]):::uc
      u61(["UC-06.1 Vuốt trái thẻ món<br/>→ lộ hành vi Ẩn"]):::sub
      u62(["UC-06.2 Xác nhận ẩn<br/>(hộp thoại nêu hệ quả)"]):::sub
      u63(["UC-06.3 Đánh dấu deleted_at/deleted_by<br/>(KHÔNG xóa cứng, giữ ảnh)"]):::sub
      u64(["UC-06.4 Ẩn khỏi danh sách & chi tiết<br/>(đọc lọc deleted_at IS NULL)"]):::sub
      u65(["UC-06.5 Hoàn tác qua snackbar<br/>→ gỡ deleted_at (restore)"]):::sub
    end
    rbac(["«include» Kiểm tra quyền (đã đăng nhập)"]):::inc
    s --> u6
    u6 --> u61 & u63 & u64
    u61 -. include .-> u62
    u6 -. extend .-> u65
    u6 -. include .-> rbac
    classDef actor fill:#e3f2fd,stroke:#1565c0;
    classDef uc fill:#f3e5f5,stroke:#6a1b9a;
    classDef sub fill:#fff3e0,stroke:#ef6c00;
    classDef inc fill:#e8f5e9,stroke:#2e7d32;
```

> **Backlog (non-MVP):** màn "Món đã ẩn" cho phép xem lại + khôi phục, và **xóa cứng vĩnh viễn** (chỉ `admin`, kèm dọn ảnh Cloudinary). Chưa thuộc phạm vi bản này.

### UC-07 — Quản lý người dùng (chỉ admin)

```mermaid
flowchart TB
    o(("Owner<br/>(admin)")):::actor
    subgraph G4b["Quản trị"]
      u7(["UC-07 Quản lý người dùng"]):::uc
      u71(["UC-07.1 Tạo tài khoản staff"]):::sub
      u72(["UC-07.2 Đổi vai trò / mật khẩu"]):::sub
      u73(["UC-07.3 Vô hiệu hóa tài khoản"]):::sub
    end
    rbac(["«include» Kiểm tra quyền admin"]):::inc
    o --> u7
    u7 --> u71 & u72 & u73
    u7 -. include .-> rbac
    classDef actor fill:#e3f2fd,stroke:#1565c0;
    classDef uc fill:#f3e5f5,stroke:#6a1b9a;
    classDef sub fill:#fff3e0,stroke:#ef6c00;
    classDef inc fill:#e8f5e9,stroke:#2e7d32;
```

---

## 6. Bảng truy vết Use Case → PRD

| UC | Tên | Actor | User story (PRD) |
|---|---|---|---|
| UC-01 | Đăng nhập | staff, admin | US-A1 |
| UC-02 | Xem danh sách & lọc hàng | staff, admin | US-B4, US-C1 |
| UC-03 | Xem chi tiết món | staff, admin | US-B5, US-C1 |
| UC-04 | Thêm món | staff, admin | US-B1 |
| UC-05 | Sửa món | staff, admin | US-B2 |
| UC-06 | Ẩn món (xóa mềm) | staff, admin | US-B3 |
| UC-07 | Quản lý người dùng | admin | US-A2 |
| UC-08 | Kiểm tra quyền (RBAC) — cross-cutting | (hệ thống) | US-A2 |

## 7. Ghi chú thiết kế

- **UC-08 (RBAC)** là cross-cutting: không phải màn hình riêng, mà là guard `«include»` ở mọi endpoint đọc/ghi (middleware ở backend + ẩn/hiện thao tác ở app).
- **Tính margin** luôn ở tầng đọc (lúc hiển thị), không lưu trong DB — nhất quán với PRD §6.
- Phần **mô phỏng giảm giá tự động** là Non-Goal (PRD NG1); owner tự đọc giá sàn vs giá bán.
- **UC-06 là xóa mềm** (`deleted_at`): mọi truy vấn đọc (UC-02/03) `«include»` ngầm điều kiện `deleted_at IS NULL`. Ẩn giữ nguyên ảnh Cloudinary và `product_id`; hoàn tác qua snackbar (UC-06.5). Quyết định Q4 chốt 2026-10-25 — xem `docs/superpowers/specs/2026-10-25-inventory-soft-delete-design.md`.
