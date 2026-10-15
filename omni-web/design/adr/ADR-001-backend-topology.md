# ADR-001 — Backend topology: một service `omnigem-api` gộp + RBAC

Trạng thái: **ACCEPTED** — 2026-10-15.
Người quyết: Product Owner + Solution Architect.
Liên quan: `omni-web/prd/PRD.md` §4.1, §8 (câu hỏi mở 3 & 7), `omni-web/design/hld.md` (B.6 SoD, A.2), `omni-web/design/solution-design-customer-facing.md`.

## Bối cảnh

Sau khi chốt bỏ headless CMS bên thứ ba (Sanity/Payload) và bỏ Google Sheet (PRD §8.3, §8.7), toàn bộ dữ liệu (catalog, blog, lead, consent, chat transcript, DSR) thuộc **một backend/DB do dự án kiểm soát**. Cần quyết định tầng backend tổ chức thế nào để:
- customer-facing (Next.js) đọc nội dung công khai,
- React-Admin ghi/quản lý catalog/blog + quản lý lead (CRM),
- xử lý PII (lead/consent/chat/DSR) theo PDPD với SoD (HLD B.6).

Ba lựa chọn đã cân nhắc:
- **A — gộp 1 backend service + RBAC** (customer-facing backend và CMS/CRM backend là một `omnigem-api`).
- **B — tách 2 service** (`content-api` công khai vs `crm-api` PII).
- **C — Next.js chọc thẳng DB** (loại: khó quản connection pool ở serverless, coupling schema, dễ phá ranh giới PII).

## Quyết định

Chọn **A**: một backend service duy nhất `omnigem-api` sở hữu **toàn bộ** quyền truy cập DB và domain logic. Tầng hóa cố định:

```
Web customer (Next.js)  ─┐
React-Admin (client)    ─┼──►  omnigem-api  ──►  1 Postgres
Next.js Route Handler   ─┘      (RBAC)            (public + PII tables)
   (BFF, server-only)
```

- Next.js **không bao giờ** chọc DB trực tiếp — chỉ gọi `omnigem-api`.
- `omnigem-api` phân 3 bề mặt qua RBAC: **public** (GET catalog/blog, no-auth, cacheable), **PII** (lead/consent/dsr/chat, server-to-server), **admin** (CRUD, auth + role).

## Lý do

- **Team nhỏ, React thuần, 1 thành viên vừa trở lại** (PRD §4.1) → ưu tiên ít moving part → 1 deploy, 1 CI.
- **Trần chi phí thấp** (NFR-6) → 1 service rẻ hơn, vận hành đơn giản.
- **Tải read công khai đã được CDN/ISR hấp thụ** → không cần scale backend read riêng, nên lợi ích chính của phương án B (scale độc lập) chưa cần tới.
- SoD và cô lập PII **đạt được trong monolith** bằng 3 lớp (xem "Hệ quả").

## Hệ quả

**Bất biến bắt buộc — cô lập PII 3 lớp** (giữ nguyên dù sau này tách service):
1. **Route/endpoint tách bạch:** public vs PII là bề mặt API khác nhau.
2. **Authz middleware (RBAC):** không có đường từ public endpoint tới handler PII.
3. **DB role least-privilege (R-19):** role phục vụ public endpoint chỉ `SELECT` bảng public (collection, product, element_tag, product_element, blog, faq, privacy_policy_version); role PII riêng, audited. Kể cả app lỗi, DB vẫn chặn.

**Ranh giới module trong `omnigem-api`** (để tách thành phương án B sau này gần như không phải viết lại — evolutionary architecture): module `content` (catalog/blog) tách khỏi module `crm/pii` (lead/consent/dsr/chat). Không import chéo quyền dữ liệu.

**Ảnh hưởng customer-facing:**
- `CmsAdapter` trở thành một **HTTP client** gọi **public endpoint** của `omnigem-api` (đổi tên đề xuất: `ContentRepository`), đọc qua `fetch(..., { next: { revalidate, tags } })` cho ISR + `revalidateTag` qua webhook on-publish.
- Route Handler PII (`/api/lead`, `/api/chat`, `/api/dsr`) là **BFF forward** tới PII endpoint của `omnigem-api` (server-to-server), vẫn `runtime=nodejs` + `Cache-Control: no-store`.

**Điều kiện tách sang B về sau:** team tách đội, tải read backend vượt ngưỡng ISR hấp thụ, hoặc audit PDPD đòi cô lập vật lý PII.

## Trạng thái các tài liệu cập nhật theo ADR này
- `solution-design-customer-facing.md` — §3, §4, §8.
- `solution-design-epic-a-browse-discover.md` — D2, D5.
