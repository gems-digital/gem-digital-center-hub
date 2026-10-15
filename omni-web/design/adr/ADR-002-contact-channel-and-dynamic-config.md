# ADR-002 — Kênh handoff Messenger-first & cấu hình động qua `SiteConfig`

Trạng thái: **ACCEPTED** — 2026-10-15.
Người quyết: Product Owner (business) + Solution Architect.
Liên quan: `omni-web/prd/PRD.md` §1, §6a, §8.1/§8.6, Epic B (US-B1/B2); `omni-web/design/solution-design-customer-facing.md` §7.4; `omni-web/design/solution-design-epic-a-browse-discover.md` D11/D14; ADR-001.

## Bối cảnh

Hai quyết định vận hành nảy sinh khi làm rõ thực tế kinh doanh:
1. **Kênh handoff:** Zalo OA yêu cầu **thuê bao trả phí hàng tháng**. Giai đoạn đầu chưa có doanh thu → muốn dùng **Facebook Messenger (Page đã có, miễn phí)** trước, khi có doanh thu mới đăng ký thêm Zalo OA.
2. **Hiển thị giá:** giá thực tế **500k–5tr VND** (không phải 20–200tr như PRD §1 mô tả) → phần lớn sản phẩm nên **hiện giá trực tiếp**; "liên hệ để biết giá" chỉ là ngoại lệ cho hàng độc bản. Cần ngưỡng điều chỉnh được mà không redeploy.

## Quyết định

**1. Kênh handoff Messenger-first, đa kênh, cấu hình động.**
- Trừu tượng hoá qua **`ContactChannelResolver`**; kênh chính do flag **`SiteConfig.primary_contact_channel`** quyết (`messenger` | `zalo_oa` | `zalo_personal`).
- **Mặc định `messenger`:** `m.me/<page>?ref=<productCode>` — `ref` mang mã SP chắc chắn (Meta documented), gắn Meta Pixel/ads (Epic G, P4).
- **Zalo OA bật sau** khi có doanh thu bằng cách đổi flag (có thể chạy song song / A-B test), không sửa code. Cú pháp prefill Zalo vẫn phải xác minh tài liệu Zalo Developer trước khi bật (PRD §8.1) — thiết kế giảm cấp 3 tầng prefill→ref→chat-trống.
- **Zalo cá nhân:** chỉ stopgap, không khuyến nghị (không audit, PII trên máy cá nhân — xung đột R-03/PDPD).
- Event `click_zalo` tổng quát hoá → **`click_contact { channel, productId }`** (giữ ánh xạ UC-08 AC2).
- **Lead-capture theo giai đoạn:** P2 chỉ cần deep-link mở chat; **lead tạo qua form/chatbot** như hiện tại. **Webhook auto-tạo lead** (Messenger `pages_messaging` cần Meta app review; Zalo OA API) để **P3/P4**.

**2. Hiển thị giá mặc-định-hiện, ngưỡng động.**
- Thứ tự: `contact_for_price` (override per-product) → nếu trống thì so `price_vnd ≥ SiteConfig.price_contact_threshold_vnd`.
- Band 500k–5tr ⇒ **mặc định hiện giá**; override chỉ cho hàng hiếm.
- Ngưỡng là **flag động** trong `SITE_CONFIG`, sửa qua React-Admin, không redeploy (giải quyết PRD §8.6, không còn chờ con số cứng).

**3. `SITE_CONFIG` — bảng key/value cấu hình động** (`omnigem-api`, sửa qua React-Admin, đẩy ra public qua ISR/revalidate). Giữ tối thiểu: `primary_contact_channel`, `price_contact_threshold_vnd`, (tương lai) các flag A-B CTA.

## Lý do
- **Chi phí:** Messenger 0đ/tháng, hợp giai đoạn tiền doanh thu; Page đã có (`documents/create-facebook-fanpage.md`).
- **Kỹ thuật:** `m.me?ref=` mang mã SP chắc chắn + hợp Meta ads; prefill Zalo còn là ẩn số.
- **Linh hoạt:** flag động cho phép bật Zalo OA / đổi ngưỡng giá / A-B test mà không đụng code (hợp P4 Growth).

## Hệ quả
- Model vận hành giữ nguyên: **thu lead → chăm sóc + chốt qua người (Messenger) → ship / kéo về cửa hàng**; bán online (checkout) là **phase tương lai**, **NG1 giữ nguyên** hiện tại.
- `ContactCTA`/`ContactFloatingWidget` ổn định bất kể kênh; chỉ `ContactChannelResolver` + CSP allowlist đổi theo kênh.
- CSP phải allowlist origin của kênh đang bật (Messenger plugin / Zalo widget).
- Cross-border: Messenger = egress Meta (đã khai báo Pixel ở Privacy Policy) — bổ sung Messenger vào danh sách bên nhận (US-H1).

## Reconcile PRD §1 — ĐÃ XỬ LÝ (PO duyệt 2026-10-15)
Giá thực **500k–5tr** và lý do giữ model "chốt qua người" (giai đoạn đầu bán qua người, online tính sau — KHÔNG phải vì giá 20–200tr) đã được cập nhật vào **PRD §1 (bản sửa R1)** và **PRD §8.6**. Mô hình phễu/NG1 không đổi.

## Tài liệu cập nhật theo ADR này
- `solution-design-customer-facing.md` — §7.4 (kênh handoff), §13 (event `click_contact`), scope/folder/traceability.
- `solution-design-epic-a-browse-discover.md` — D3 (`TESTIMONIAL`, `SITE_CONFIG`), D11 (`ContactCTA`), D14 (giá động), §16.
