# OmniGem Gallery — Chia việc Epic A: Duyệt & Khám phá sản phẩm (UC-01..04)

Trạng thái: **DỰ THẢO** — 2026-10-15. Tác giả: PM.
Nguồn sự thật: `omni-web/prd/PRD.md` (Epic A, US-A1..A4), `omni-web/usecase/use-cases.md` (UC-01..04), `omni-web/design/solution-design-epic-a-browse-discover.md`.

> **Triết lý chia việc: lát cắt dọc, cuốn chiếu (vertical slice).**
> Mỗi lát là một đường xuyên suốt `route → model (chỉ phần cần) → API call (chỉ method cần) → UI → SEO → test`, **kết thúc bằng một kết quả demo/kiểm thử được ngay**. KHÔNG gom "domain types" hay "ContentRepository" thành một pha nền tảng tách rời (tránh kiểu thác nước, khó nghiệm thu, khó đóng gói).

---

## 0. Nguyên tắc chung

1. **Mỗi lát tự mang theo đúng phần model + repo method nó cần, không hơn.** Model/API lớn dần theo lát.
2. **`ContentRepository` không build trọn gói** — chỉ thêm method khi lát đó gọi tới. Interface giữ ổn định làm ranh giới chống vỡ khi schema `omnigem-api` đổi.
3. **Mock cuốn chiếu:** mỗi method mới đi kèm mock adapter để FE không bị chặn bởi schema backend chưa chốt.
4. **Mỗi lát đóng được = demo được cho PO** theo đúng Definition of Done chung (§1).
5. **Rework nhẹ có chủ đích:** chấp nhận sửa type/field khi lát sau cần thêm, đổi lại không over-design.

---

## 1. Definition of Done chung (áp dụng MỌI lát)

Một lát chỉ "xong" khi:

- [ ] Route chạy thật với mock adapter (không lỗi hydrate/console).
- [ ] Chỉ thêm model/API đúng phạm vi lát; `ContentRepository` interface không vỡ.
- [ ] Unit/integration test của lát xanh.
- [ ] Core Web Vitals của trang trong lát đạt ngân sách p75 mobile 4G: **LCP ≤ 2.5s · INP ≤ 200ms · CLS ≤ 0.1**.
- [ ] JSON-LD (nếu trang có) valid trên Rich Results Test.
- [ ] "Không ngõ cụt": trang có ≥ 1 lối ra tới phễu hoặc Product Detail.
- [ ] Demo được: mở đúng URL thấy kết quả, PO nghiệm thu.

---

## 2. Lát 1 — Home hiển thị được (UC-01, US-A1)

**🎯 Kết quả demo:** mở `/`, thấy hero + lưới sản phẩm nổi bật, bấm card điều hướng đi, widget liên hệ nổi hoạt động.

| Task | Chi tiết | AC |
|---|---|---|
| App shell | `layout.tsx`: Header/Footer, `ConsentProvider`, `ContactFloatingWidget` (Messenger-first), `ChatbotWidget` (dynamic import, không chặn LCP) | — |
| Route + UI | `app/(marketing)/page.tsx` (RSC, ISR 60m), `HeroSection` (LCP, `next/image priority` + preload), `FeaturedGrid` + `ProductCard` (bọc `<Link>`) | US-A1 AC1/AC2 |
| **Model thêm (chỉ phần cần)** | `ProductSummary` (field card cần: slug/code/name/jadeType/cover(MediaRef)/priceVnd\|contactForPrice/menhTags/featured), `MediaRef`, enum `Menh` | — |
| **API thêm (chỉ method cần)** | `ContentRepository.getFeaturedProducts()` + mock adapter | — |
| SEO | `OrganizationJsonLd`, `generateMetadata` home; khởi tạo `sitemap.ts` + `robots.ts` | US-E1 AC2 |
| ISR | Bật ISR + đặt **convention tag** (`product:<slug>`) để các lát sau dùng lại | — |
| Test | render home; card → link; CWV home | — |

> **Tracer bullet:** nên để ít nhất 1 lần gọi `omnigem-api` thật ở lát này (dù chỉ 1 endpoint) để lộ vấn đề tích hợp sớm, không để mock che tới cuối.

---

## 3. Lát 2 — Duyệt & lọc collections (UC-02, US-A2)

**🎯 Kết quả demo:** mở `/collections`, lọc theo danh mục + mệnh thấy lưới đổi tức thì; lọc rỗng → empty-state có CTA phễu.

| Task | Chi tiết | AC |
|---|---|---|
| Route + prefetch | `collections/page.tsx` (RSC, ISR 30m) prefetch toàn bộ `ProductSummary` + categories + tags | — |
| UI lọc client | `CollectionsBoard` (`"use client"`) + `useFilter()` (useMemo, giao `category ∩ mệnh`) — state machine | US-A2 AC1/AC2 |
| UI bộ lọc | `CategoryFilter` (UC-02.1), `MenhFilter` Kim/Mộc/Thủy/Hỏa/Thổ (UC-02.2) | US-A2 AC1/AC2 |
| UI kết quả | `ProductGrid`, `EmptyStateFunnel` (UC-02.3 extend → CTA Messenger/chatbot) | US-A2 AC3 |
| **Model thêm** | `Collection`; bổ sung field tags vào `ProductSummary` nếu còn thiếu | — |
| **API thêm** | `getCollections()`, `getProductsByCollection()` + mock | — |
| Test | `useFilter` (giao category∩mệnh); state machine → empty; E2E "empty không ngõ cụt" | US-A2 AC3 |

---

## 4. Lát 3 — Chi tiết sản phẩm (UC-03, US-A3) — chẻ 3 lát con

Bề mặt lớn nhất + SEO cốt lõi → chẻ nhỏ để mỗi lát con vẫn demo được.
**Ưu tiên 3a + 3c** (đọc được + vào phễu = giá trị kinh doanh cốt lõi); 3b có thể trượt lịch mà không chặn phễu.

### 4.1 Lát 3a — Detail đọc được + giá + SEO
**🎯 Demo:** `/products/[slug]` render spec/xuất xứ/chứng nhận + giá-hoặc-liên-hệ; slug sai → 404 có CTA (không dead-end).

| Task | Chi tiết | AC |
|---|---|---|
| Route | `products/[slug]/page.tsx` (RSC, ISR 30m) + `generateStaticParams` + `notFound()` + `not-found.tsx` | US-A3 |
| UI | `SpecPanel` (jadeType/size/origin/certRef), `PriceOrContact` (cây quyết định giá hybrid) | US-A3 AC1 |
| **Model thêm** | `Product` (full), `SiteConfig` (đọc `price_contact_threshold_vnd`) | — |
| **API thêm** | `getProductBySlug()`, `getAllProductSlugs()` + mock | — |
| SEO | `ProductJsonLd`, metadata/canonical, `opengraph-image`, thêm entry vào `sitemap.ts` | US-E1 AC1 |
| Webhook | `POST /api/revalidate?tag=product:<slug>` → `revalidateTag` (hỗ trợ UC-17.4 publish không redeploy) | US-F1 AC1 |

### 4.2 Lát 3b — Gallery + khối phong thủy
**🎯 Demo:** xem/zoom ảnh, video poster click-to-play; nhập năm sinh xem hợp mệnh (tính client, không gửi server).

| Task | Chi tiết | AC |
|---|---|---|
| UI gallery | `ProductGallery` (UC-03.1, state máy zoom, ảnh zoom **on-demand** ngoài budget, video poster + click-to-play) | US-A3 AC1 |
| UI phong thủy | `FengShuiMatchBlock` (UC-03.3) | US-A3 AC1 |
| **Lib thêm** | `lib/fengshui/dobToMenh.ts` — **thuần client, không lưu, không gửi server → không kích consent** | — |
| Test | `dobToMenh`; CWV budget ảnh (viewport ≤ 1.0MB, cover ≤ 200KB) | NFR-1/2 |

### 4.3 Lát 3c — CTA phễu + social proof + analytics
**🎯 Demo:** 2 CTA above-the-fold (Messenger + Hỏi AI); social proof khi có media; bắn event analytics.

| Task | Chi tiết | AC |
|---|---|---|
| UI CTA | `CtaBar` above-the-fold mobile (UC-03.5): `ContactCTA` + `AskAiButton(ctx=code)` | US-A3 AC2 |
| Channel resolver | `ContactChannelResolver` — Messenger-first `m.me/page?ref=code`, Zalo OA bật sau theo flag `SiteConfig.primary_contact_channel` | US-A3 AC2 |
| UI proof | `SocialProof` (UC-03.4 extend) — chỉ khi có media chứng thực | US-A3 AC3 |
| **Model thêm** | `Testimonial` (đọc bản `status="published"`) | — |
| **Analytics thêm** | `lib/analytics/events.ts`: `view_product`, `click_contact {channel, productId}` — bắn **sau** consent analytics, **không PII** | UC-21 |

---

## 5. Lát 4 — Brand & Certification (UC-04, US-A4)

**🎯 Kết quả demo:** `/brand-story` hiển thị câu chuyện nguồn gốc + giải thích kiểm định lab + trust copy đồng bộ chatbot.

| Task | Chi tiết | AC |
|---|---|---|
| Route + UI | `brand-story/page.tsx` (RSC, ISR 60m); `OriginStory` (UC-04.1), `LabCertExplainer` (UC-04.2) + link trang chứng nhận | US-A4 AC1 |
| Trust nhất quán | `TrustCopy` — **nguồn chung** với chatbot FAQ (US-D2) + copy fanpage, tránh lệch thông điệp "ngọc tự nhiên 100% + chứng nhận lab" | US-A4 AC1 |
| **API thêm** | `getBrandContent()` + mock | — |
| SEO | `OrganizationJsonLd`, `generateMetadata` | US-E1 AC2 |

---

## 6. Thứ tự & tính độc lập

```
Lát 1 (Home) ──► Lát 2 (Collections) ──► Lát 3a ─► 3b ─► 3c ──► Lát 4 (Brand)
     │
     └─ sau Lát 1: Lát 2 và Lát 4 có thể chạy song song nếu đủ người
```

- Mỗi mũi tên = một lần **demo được cho PO** (không đợi hết sprint mới thấy kết quả).
- **3a + 3c** ưu tiên hơn 3b (phễu là giá trị cốt lõi).

---

## 7. Thứ "mọc dần" qua từng lát (KHÔNG phải pha riêng)

Đây là lý do không có nhóm cross-cutting tách rời — chúng nằm trong DoD của từng lát:

| Hạng mục | Cách mọc dần |
|---|---|
| ISR + webhook revalidate | Bật ISR từ Lát 1; mỗi route tự khai `revalidate` + `tag`; webhook hoàn thiện ở Lát 3a |
| SEO / JSON-LD | Mỗi lát kèm JSON-LD + metadata cho trang của nó; `sitemap/robots` khởi tạo Lát 1, thêm entry dần |
| CWV budget | Là tiêu chí nghiệm thu của chính lát, không phải task cuối |
| Analytics | Chỉ xuất hiện ở lát đầu tiên cần (Lát 3c) |

---

## 8. Rủi ro & điểm cần chốt

| # | Điểm | Ghi chú |
|---|---|---|
| 1 | Schema/base URL `omnigem-api /public/*` | Còn chốt ở Design (ADR-001). Giảm thiểu bằng interface ổn định + mock cuốn chiếu + tracer bullet Lát 1. |
| 2 | **Default `price_contact_threshold_vnd`** | ✅ **CHỐT (PO, 2026-10-15): band giá 500k–5tr.** Mặc định HIỆN GIÁ; `contact_for_price` override cho hàng độc bản; threshold = flag động `SITE_CONFIG`, đặt sao cho hầu hết SP trong band rơi vào nhánh "hiện giá". |
| 3 | Convention tag ISR | Nhiều người làm song song dễ lệch tag — đặt convention ở Lát 1, một người giữ nhất quán. |
| 4 | `ContactChannelResolver` theo flag | Messenger-first ngay từ đầu, Zalo OA bật sau — điều khiển bằng flag từ Lát 3c để tránh refactor. |
