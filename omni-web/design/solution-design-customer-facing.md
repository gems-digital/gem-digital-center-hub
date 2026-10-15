# OmniGem Gallery — Solution Design: Customer-Facing (Boundary 1)

Trạng thái: **DRAFT** — 2026-10-15. Tác giả: solution-architect.
Nguồn sự thật (source of truth): `omni-web/prd/PRD.md`, `omni-web/usecase/use-cases.md`, `omni-web/design/hld.md`.
Tham khảo phụ (không ràng buộc): `.chapter-forge/artifacts/design/*` (api-spec, erd, nfr-targets, adr…).

> **Phạm vi:** Tài liệu này chỉ thiết kế **Boundary 1 — Customer-facing** (website công khai Next.js + các client widget + các Route Handler mà widget gọi). **Boundary 2 — Internal** (React-Admin: built-in CMS + CRM, DSR back-office, maker–checker) chỉ xuất hiện ở đây dưới dạng **hợp đồng tích hợp** (contract) mà customer-facing phụ thuộc, không thiết kế chi tiết.

> **GHI CHÚ ĐỒNG BỘ (bắt buộc đọc):** Theo quyết định đã chốt trong PRD §8 (Câu hỏi mở 3, 4, 7) và GHI CHÚ ĐỒNG BỘ của `hld.md`, thiết kế này dùng:
> - **Gemini API (Google)** cho chatbot — KHÔNG phải Claude/Anthropic (dù sơ đồ HLD gốc còn vẽ "Claude"). Vòng tool-calling đổi sang **Gemini function calling**; nghĩa vụ xuyên biên giới PDPD Art. 25 vẫn áp cho Google.
> - **Built-in CMS (React-Admin + backend/DB của dự án)** — KHÔNG dùng Sanity/Payload.
> - **CRM built-in** làm kho lead — KHÔNG dùng Google Sheet.
>
> **Backend topology (ADR-001, ACCEPTED):** chọn **phương án A — gộp 1 backend service `omnigem-api` + RBAC**. Tầng hóa cố định: **web customer (Next.js) → `omnigem-api` → 1 Postgres**. Next.js **không bao giờ chọc DB trực tiếp**; nó gọi `omnigem-api`. Customer-facing và CMS/CRM dùng **chung một backend**, phân tách public / PII / admin bằng RBAC + DB role least-privilege (xem `adr/ADR-001-backend-topology.md`).

---

## 1. Mục tiêu & bối cảnh

OmniGem Gallery là **showroom số + trust engine + phễu dẫn về Zalo/Chatbot**, KHÔNG phải sàn TMĐT (PRD §1). Không checkout, không giỏ hàng, không thanh toán, không tài khoản người mua (PRD NG1–NG4).

Nhiệm vụ của phần customer-facing:
1. Trình bày catalog đẹp, chứng minh tính xác thực → **xây niềm tin** (Epic A).
2. Điều hướng/lọc theo **ngũ hành (mệnh)** để khớp sản phẩm với khách (G2, US-A2).
3. **Không có ngõ cụt**: mọi trang/empty-state/cuối bài blog đều dẫn vào phễu Zalo/chatbot/form (US-A2 AC3, US-I2 AC3).
4. Chuyển quan tâm thành **lead đủ điều kiện** qua deep-link Zalo, form lead, chatbot có **human handoff bắt buộc** (Epic B, C, D).
5. **SEO-first** (SSG/ISR, JSON-LD, sitemap) — kênh tăng trưởng cốt lõi (G4, Epic E, I).
6. **Tuân thủ PDPD** phía client: consent gate tái lập được, privacy notice, DSR intake (Epic H).
7. **Đo lường phễu** không chứa PII (Epic G).

### 1.1 Ràng buộc nền tảng (đội dev)
Đội là **React thuần**, ưu tiên stack quen thuộc (PRD §4.1). Public site giữ **Next.js** (vẫn 100% React) vì SEO đặt cược vào SSG/ISR. Đây là ràng buộc định hình toàn bộ lựa chọn kỹ thuật dưới đây: ưu tiên API/convention của Next.js App Router, tránh thư viện lạ khi một pattern React/Next chuẩn đã đủ.

---

## 2. Phạm vi customer-facing (bản đồ use case → thành phần)

| Nhóm UC | Use case | Thành phần customer-facing |
|---|---|---|
| Duyệt & khám phá | UC-01..04 (Epic A) | `/`, `/collections`, `/products/[slug]`, `/brand-story` |
| Đọc blog | UC-05..07 (Epic I) | `/blog`, `/blog/[slug]`, `/blog/category/[cat]` |
| Liên hệ qua phễu | UC-08..10 (Epic B, C) | `ContactCTA`, `ContactFloatingWidget`, `LeadForm` + `/api/lead` |
| Chatbot AI | UC-11..13 (Epic D) | `ChatbotWidget` + `/api/chat` (Gemini orchestration) |
| Quyền riêng tư & DSR | UC-14..16 (Epic H) | `/privacy`, `/privacy/dsr` + `/api/dsr`, `ConsentGate` |
| Analytics (cross-cutting) | UC-21 (Epic G) | `analytics` dispatcher + `/api/collect` (tùy chọn) |

**Ngoài phạm vi (Boundary 2, chỉ là contract):** UC-17 quản lý catalog, UC-18 quản lý blog, UC-19 CRM, UC-20 DSR back-office maker–checker.

---

## 3. Tech stack customer-facing

| Lớp | Lựa chọn | Lý do (bám PRD/HLD) |
|---|---|---|
| Framework | **Next.js (App Router)**, TypeScript, `react-server-components` | SSG/ISR cho SEO (NFR-3); file-based routing; `next/image` cho CWV (NFR-1). PRD §4.1 giữ Next.js. |
| Styling | Tailwind CSS + CSS variables (design tokens) | Mobile-first, kiểm soát CWV/CLS, không runtime CSS-in-JS nặng. |
| UI primitives | Radix UI / headless + component tự xây | A11y WCAG 2.1 AA (NFR-5): focus trap, ARIA sẵn. |
| Ảnh/Video | **Cloudinary** + `next/image` loader (`f_auto,q_auto`) | CDN, responsive, budget ảnh (nfr-targets §2). |
| Validation | **Zod** (dùng chung client ↔ server) | Fail-fast tại biên (HLD B.1); 1 schema, 2 nơi. |
| Data fetch (server) | `fetch` + ISR (`next: { revalidate }`) qua **ContentRepository** (HTTP client của public endpoint `omnigem-api`) | Đọc catalog/blog từ `omnigem-api`, KHÔNG chọc DB trực tiếp (ADR-001). |
| Chatbot client | React widget + streaming (SSE/`ReadableStream`) tới `/api/chat` | First-token ≤ 3s (nfr §4); server-authoritative state (H-2). |
| Analytics | GA4 + Meta Pixel (client), load **sau consent**; (tùy chọn) proxy `/api/collect` | Không PII trong event (HLD Flow 3). |
| Bot protection | Cloudflare Turnstile / Vercel bot protection | Chặn denial-of-wallet trên `/api/chat` (H-1). |
| Hosting | **Vercel** (edge/serverless) | SSG/ISR + Route Handlers `runtime=nodejs` cho route PII. |

**Nguyên tắc chốt cứng (FIRM):**
- Mọi route xử lý PII (`/api/lead`, `/api/chat`, `/api/dsr`) đặt `Cache-Control: no-store`, chạy `runtime = "nodejs"`, **không bao giờ** cache ở edge (HLD A.2).
- **Zero secret** ở client bundle: không `NEXT_PUBLIC_` cho bất kỳ secret nào. Chỉ khóa publishable (GA4 measurement ID, Pixel ID, Cloudinary cloud name) được phép public (HLD B.5).
- Trang/response chứa PII **không** index, **không** cache CDN.

---

## 4. Kiến trúc customer-facing (lát theo tầng)

Customer-facing sở hữu 3 tầng; tầng 4–5 (domain services + data store + CRM/CMS backend) là **ranh giới tích hợp** nó gọi tới.

```
┌──────────────────────────────────────────────────────────────┐
│ TẦNG 1 — PRESENTATION (SSG/ISR, App Router — server components)│
│  Pages: Home, Collections, Product, Brand, Blog×3, Privacy, DSR│
│  Server: đọc nội dung công khai qua ContentRepository (ISR)           │
├──────────────────────────────────────────────────────────────┤
│ TẦNG 1b — CLIENT WIDGETS ("use client")                        │
│  ConsentGate · LeadForm · ChatbotWidget · ZaloWidget           │
│  MệnhFilter · ProductGallery(zoom) · AnalyticsProvider         │
├──────────────────────────────────────────────────────────────┤
│ TẦNG 2 — BFF / ROUTE HANDLERS (server-only, no-store)          │
│  /api/lead · /api/chat · /api/dsr · /api/collect · /api/session│
│  Cắt ngang: originCheck · rateLimit · zodValidate · consentGate│
├──────────────────────────────────────────────────────────────┤
│  ───────── Ranh giới service: gọi omnigem-api (ADR-001) ────── │
│  omnigem-api (1 backend, RBAC): public / PII / admin endpoints │
│   TẦNG 3 Chatbot orchestration (Gemini) · TẦNG 4 Domain svc    │
│   (Lead/Consent/ProductQuery/DSR) · TẦNG 5 DB access → Postgres│
└──────────────────────────────────────────────────────────────┘
```

**Topology (ADR-001):** `web customer (Next.js) → omnigem-api → 1 Postgres`. Next.js (cả Server Component lẫn Route Handler) **không chọc DB trực tiếp** — Server Component gọi **public endpoint** (ISR), Route Handler PII gọi **PII endpoint** (server-to-server). `omnigem-api` là service **gộp chung** với CMS/CRM, phân tách bằng RBAC 3 lớp (route tách bạch · authz middleware · DB role least-privilege).

Ánh xạ sang HLD A.1: tầng 1/1b = Presentation; tầng 2 = BFF Route Handlers. Tầng 3–5 nằm trong `omnigem-api` (chỉ đổi vendor adapter Claude→**Gemini**).

### 4.1 Vì sao tách TẦNG 2 (BFF) khỏi widget
Widget chạy ở trình duyệt (untrusted). Mọi logic nhạy cảm — ký session, scrub PII, gọi Gemini, ghi consent/lead — **chỉ** chạy server trong Route Handler. Widget chỉ: (a) thu input, (b) hiển thị consent, (c) POST tối thiểu, (d) render phản hồi + fallback. Đây là hiện thực của trust boundary TB-1 (system-context §3).

---

## 5. Cấu trúc thư mục đề xuất (Next.js App Router)

```
omni-web/app-public/                      # Next.js app (customer-facing)
├─ app/
│  ├─ (marketing)/                        # nhóm route public, SEO
│  │  ├─ page.tsx                         # UC-01 Home (SSG+ISR 60m)
│  │  ├─ collections/page.tsx             # UC-02 (SSG+ISR 30m, lọc client)
│  │  ├─ products/[slug]/page.tsx         # UC-03 (SSG+ISR 30m, generateStaticParams)
│  │  ├─ brand-story/page.tsx             # UC-04 (SSG+ISR 60m)
│  │  └─ blog/
│  │     ├─ page.tsx                      # UC-05 listing (SSG+ISR 30m, paginate)
│  │     ├─ [slug]/page.tsx               # UC-06 detail (SSG+ISR 60m)
│  │     └─ category/[cat]/page.tsx       # UC-07 archive (generateStaticParams)
│  ├─ privacy/
│  │  ├─ page.tsx                         # UC-14 Privacy Policy (SSG versioned)
│  │  └─ dsr/page.tsx                     # UC-16 DSR intake (SSG shell + client form)
│  ├─ api/                                # TẦNG 2 — Route Handlers (runtime=nodejs)
│  │  ├─ lead/route.ts                    # UC-10 → /api/lead
│  │  ├─ chat/route.ts                    # UC-11..13 → Gemini loop
│  │  ├─ session/route.ts                 # cấp sessionToken ký HMAC (H-1)
│  │  ├─ dsr/route.ts                     # UC-16 → /api/dsr
│  │  └─ collect/route.ts                 # UC-21 proxy analytics (tùy chọn)
│  ├─ sitemap.ts · robots.ts              # SEO (Epic E)
│  ├─ layout.tsx                          # shell: header/footer, ConsentProvider, widgets
│  └─ opengraph-image.tsx                 # OG mặc định
├─ components/
│  ├─ product/ (ProductCard, ProductGallery, FengShuiMatch, SocialProof)
│  ├─ funnel/  (ContactCTA, ContactFloatingWidget, ContactChannelResolver, LeadForm, FunnelCTABlock)
│  ├─ chat/    (ChatbotWidget, ChatMessageList, ChatComposer, HandoffCard)
│  ├─ consent/ (ConsentGate, ConsentProvider, PrivacyNotice, AnalyticsLoader)
│  ├─ collections/ (MenhFilter, CategoryFilter, EmptyStateFunnel)
│  └─ seo/     (ProductJsonLd, ArticleJsonLd, OrganizationJsonLd)
├─ lib/
│  ├─ content/ (ContentRepository, types, mappers)    # HTTP client → omnigem-api /public/*
│  ├─ api-client/ (leadClient, chatClient, dsrClient) # gọi TẦNG 2
│  ├─ analytics/ (events.ts, dispatcher.ts)           # tên event chuẩn hóa
│  ├─ schemas/ (lead.ts, chat.ts, dsr.ts — Zod dùng chung)
│  ├─ security/ (origin.ts, rateLimit.ts, session.ts, scrubber.ts)
│  └─ fengshui/ (dobToMenh.ts — map DOB → ngũ hành)
└─ next.config.ts · middleware.ts         # CSP/security headers (M-3), origin
```

**Nguyên tắc tổ chức:** tổ chức theo **feature/domain** (product, funnel, chat, consent), file nhỏ 200–400 dòng (coding-style). Schema Zod đặt ở `lib/schemas` để import được ở **cả** client (validate trước submit) lẫn server (validate khi nhận).

---

## 6. Thiết kế từng trang

### 6.1 Home `/` (UC-01, US-A1)
- **Render:** SSG + ISR 60 phút (+ webhook on-publish). Hero là phần tử LCP.
- **Data:** `ContentRepository.getFeaturedProducts()`, `getHeroContent()` ở server component.
- **Component:** `HeroSection` (ảnh Cloudinary `priority`, preload, ≤ 200KB), `FeaturedGrid` → `ProductCard` link tới `/products/[slug]` (AC2).
- **CWV:** reserve kích thước ảnh (CLS ≤ 0.1); không autoplay video kèm audio.

### 6.2 Collections `/collections` (UC-02, US-A2)
- **Render:** SSG + ISR 30 phút, **lọc phía client** trên tập đã prefetch (HLD A.2). HTML nền crawl được, lọc là tương tác tức thì (INP ≤ 200ms).
- **Data:** server prefetch toàn bộ product summary + element tags; client lọc.
- **Component:** `CategoryFilter` (UC-02.1), `MenhFilter` (Kim/Mộc/Thủy/Hỏa/Thổ — UC-02.2), `ProductGrid`.
- **Empty state (UC-02.3, US-A2 AC3 — «extend»):** khi không khớp → `EmptyStateFunnel` với CTA "Hỏi chatbot" + "Nhắn Zalo" — **không ngõ cụt**.
- **Quyết định:** lọc client thay vì server để tránh round-trip và giữ trang tĩnh cache được; chỉ phù hợp vì tập catalog nhỏ (showroom, không phải sàn nghìn SKU).

### 6.3 Product Detail `/products/[slug]` (UC-03, US-A3)
- **Render:** SSG + ISR 30 phút, `generateStaticParams` từ `ContentRepository.getAllProductSlugs()`. Bề mặt SEO cốt lõi.
- **Data:** `ContentRepository.getProductBySlug(slug)`; `notFound()` nếu không có.
- **Component:**
  - `ProductGallery` (UC-03.1): ảnh/video độ phân giải cao, **zoom**; lazy-load ảnh dưới nếp gấp (budget ≤ 1.0MB initial viewport).
  - Thông số/xuất xứ/chứng nhận (UC-03.2); `FengShuiMatchBlock` (UC-03.3).
  - Giá **hoặc** "Liên hệ để được tư vấn" theo `contact_for_price` (mô hình hybrid, PRD §6a — ngưỡng VND do PO chốt, câu hỏi mở 6).
  - `SocialProof` (UC-03.4, «extend» chỉ khi có media chứng thực).
  - **2 CTA above-the-fold trên mobile (AC2):** `ContactCTA` (Messenger-first, ref mã SP) + "Hỏi AI về sản phẩm này" (mở `ChatbotWidget` với ngữ cảnh SP đã prefill).
- **SEO:** `ProductJsonLd` (name, image, description — US-E1 AC1).
- **Analytics:** bắn `view_product` (productId, sau consent analytics).

### 6.4 Brand & Certification `/brand-story` (UC-04, US-A4)
- SSG + ISR 60m. Câu chuyện nguồn gốc + giải thích kiểm định lab + link trang chứng nhận. Tái dùng ngôn ngữ niềm tin từ `documents/create-facebook-fanpage.md` §2 (nhất quán với chatbot FAQ — US-D2).

### 6.5 Blog (UC-05..07, Epic I)
- **Listing `/blog`** (US-I1): SSG + ISR 30m, **phân trang**, lọc danh mục/tag phía client; metadata + sitemap.
- **Detail `/blog/[slug]`** (US-I2): SSG + ISR 60m, `generateStaticParams`. Rich text + ảnh Cloudinary + mục lục (heading anchors), tác giả/ngày/thời gian đọc. `ArticleJsonLd` (US-I2 AC2). **`FunnelCTABlock` cuối bài** (US-I2 AC3, «extend») + liên kết nội bộ SP/collection. Thẻ OpenGraph (AC4).
- **Archive `/blog/category/[cat]`** (US-I3): `generateStaticParams` theo danh mục, index được; khối "bài liên quan".
- **Ghi chú:** blog **không chứa PII** → cache đầy đủ ở edge; đây là kênh SEO chi phí biên ~0 (HLD A.2 ghi chú blog).

### 6.6 Privacy & DSR (UC-14, UC-16, Epic H)
- **`/privacy`** (US-H1, UC-14): SSG **có versioning** (`PRIVACY_POLICY_VERSION`). Công khai: dữ liệu thu thập, mục đích, bên xử lý/nhận (Vercel, Cloudinary, **Gemini/Google**, GA4, Meta Pixel, Zalo), và việc xử lý ngoài Việt Nam. Version hiện hành là thứ `ConsentGate` ghim vào mọi consent record (US-H2).
- **`/privacy/dsr`** (US-H3, UC-16): vỏ SSG + form client → `/api/dsr`. Chọn loại quyền (access/withdraw/erasure/restrict/object/rectify/inform), nhập định danh (SĐT/Zalo/email). **Phía client chỉ intake**; xác minh OTP tới kênh đã lưu (UC-16.3, H-4) và thực thi maker–checker là ở Boundary 2.

---

## 7. Client widgets — thiết kế chi tiết

### 7.1 ConsentGate + ConsentProvider (Epic H, nền tảng phễu)
Trái tim tuân thủ PDPD phía client. Hai loại consent tách biệt:
1. **Consent PII** (Art. 11, US-H2): bắt buộc **trước mọi submit mang PII** (form lead, lượt chatbot đầu mang PII). Checkbox minh thị, **mặc định chưa tick**, gắn `policyVersion` hiện hành. Payload gửi lên: `consent: { policyVersion, granted: true }`.
2. **Consent analytics**: cổng riêng; GA4/Pixel **chỉ load sau khi** khách đồng ý analytics (HLD Flow 3 bước 2). `AnalyticsLoader` không inject script tới khi có consent.

`ConsentProvider` (React context) giữ trạng thái consent trong phiên; **không** tự ý coi là đã đồng ý. Bằng chứng tái lập được là trách nhiệm **server** (ghi `ConsentRecord` nguyên tử), client chỉ khẳng định.

### 7.2 LeadForm (UC-10, US-C1 + US-H2)
- **Trường:** name, phone|zaloId (ít nhất 1), **dob**, productInterest/productCode, consent.
- **Validate client:** Zod `LeadInput` (cùng schema server) — trường bắt buộc, regex SĐT VN. Fail-fast, báo lỗi tại chỗ, ARIA error announcement (a11y).
- **Luồng submit (bám HLD B.1 / UC-10 luồng chính):**
  1. Client validate → POST `/api/lead` (TLS, same-origin).
  2. Nếu `consent.granted !== true` → UI chặn, không gửi.
  3. Nhận `201 { leadId }` → hiện xác nhận; bắn `submit_lead_form` (no PII).
  4. Nhận `503 store_unavailable` → render **CTA fallback Zalo** kèm `zaloUrl` (UC-10.4 «extend», không commit một phần).
  5. `429` → thông báo thử lại sau; `400` → lỗi field.
- DOB có **privacy notice** giải thích dùng cho tư vấn phong thủy (US-C1 AC3).
- **Fixture test chỉ dùng dữ liệu tổng hợp** (US-C1 AC4) — không PII thật.

### 7.3 ChatbotWidget (UC-11..13, Epic D) — dùng **Gemini**
Đây là widget phức tạp nhất. Nguyên tắc: **server-authoritative** (H-2), **consent-first** (H-3), **không ngõ cụt** (fallback Zalo).

**Khởi tạo phiên (H-1):**
1. Khi mở widget → vượt **bot challenge** (Turnstile/Vercel).
2. Gọi `/api/session` → nhận **`sessionToken` ký HMAC** (server phát, bound với kết quả challenge; client không giả mạo/xoay vòng được).
3. Hiện **micro-notice consent** trước lượt mang PII đầu tiên.

**Mỗi lượt chat (contract `ChatInput` — api-spec):**
- Client POST **chỉ**: `{ sessionToken, consent, userMessage }` tới `/api/chat`.
- **TUYỆT ĐỐI KHÔNG** gửi lịch sử assistant trước đó (H-2 chống forged-history) — server tự nạp trạng thái phiên từ store riêng.
- `userMessage` ≤ 4000 ký tự.
- Nhận `ChatReply`: `{ reply, products[], handoff: { zaloUrl }|null, fallback }`. Render:
  - `products[]` → card gợi ý (UC-11.2, dữ liệu CMS thật, không bịa).
  - `handoff` → `HandoffCard` với nút mở Zalo (UC-13).
  - `fallback: true` → thông báo nhẹ + CTA Zalo (Gemini lỗi/timeout hoặc chạm trần chi tiêu — H-1/R-06).
- **Streaming:** phản hồi dạng stream để first-token ≤ 3s (nfr §4); widget có ARIA live region đọc tin nhắn bot (a11y).
- Khi mở từ Product Detail: prefill ngữ cảnh `productCode` vào lượt đầu (US-A3 AC2).
- Bắn `chatbot_handoff` khi có handoff (UC-21).

**Lưu ý vendor (reconcile):** tầng server (`/api/chat` → orchestration) dùng **Gemini function calling** với 2 tool `search_products` / `handoff_to_zalo`; input scrubber chạy **trước** cuộc gọi xuyên biên giới tới Google (C-1). Widget client **không biết** vendor nào — hợp đồng `ChatInput/ChatReply` ổn định bất kể Claude hay Gemini. Đây là lợi ích của việc giấu vendor sau BFF.

### 7.4 Kênh handoff — đa kênh, Messenger-first (UC-08, UC-09, Epic B; ADR-002)
Kênh do **`SiteConfig.primaryContactChannel`** (flag động, sửa qua React-Admin) quyết, giấu sau **`ContactChannelResolver`** → đổi/A-B test kênh không cần redeploy.
- **`ContactCTA(productCode)`** (UC-08, US-B1): một chạm mở kênh chat với ngữ cảnh SP.
  - **Messenger-first (mặc định):** `m.me/<page>?ref=<productCode>` — `ref` mang mã SP **chắc chắn** (Meta có tài liệu), Page miễn phí, gắn Meta Pixel/ads. Lead vẫn tạo qua form/chatbot ở P2; **webhook auto-tạo lead** (cần Meta app review `pages_messaging`) để P3/P4.
  - **Zalo OA (bật sau khi có doanh thu):** `zalo.me/<oa_id>`. ⚠️ Cú pháp prefill Zalo CHƯA kiểm chứng (PRD §8.1, US-B1 AC3) → xác nhận tài liệu Zalo Developer trước khi bật; thiết kế giảm cấp 3 tầng (prefill → ref → chat trống).
  - **Zalo cá nhân:** chỉ stopgap, có nợ governance (không audit, PII trên máy cá nhân) — tránh.
  - Bắn **`click_contact { channel, productId }`** (tổng quát hoá từ `click_zalo`, UC-08 AC2).
- **`ContactFloatingWidget`** (UC-09, US-B2): widget chat nổi mọi trang — Messenger Customer Chat plugin (mặc định), đổi sang Zalo widget qua cùng flag.
- Origin script/plugin kênh đang dùng phải nằm trong **CSP allowlist** (M-3).

### 7.5 MệnhFilter & EmptyStateFunnel
`MenhFilter` lọc client theo tag ngũ hành (ELEMENT_TAG). Khi rỗng → `EmptyStateFunnel` (triết lý không ngõ cụt).

---

## 8. Tầng dữ liệu & hợp đồng tích hợp

### 8.1 ContentRepository (HTTP client của public endpoint `omnigem-api` — ADR-001)
Customer-facing **chỉ đọc** nội dung công khai, qua một interface ổn định. Interface là một **module code trong app Next.js**; triển khai của nó là **HTTP client gọi public endpoint của `omnigem-api`** (KHÔNG chọc DB trực tiếp, KHÔNG gọi vendor ngoài — đã bỏ Sanity/Payload). Interface ổn định để mock được ở P0/P1 trong lúc `omnigem-api` đang build.

```ts
interface ContentRepository {
  getFeaturedProducts(): Promise<ProductSummary[]>;
  getAllProductSlugs(): Promise<string[]>;             // generateStaticParams
  getProductBySlug(slug: string): Promise<Product | null>;
  getCollections(): Promise<Collection[]>;
  getProductsByCollection(slug: string): Promise<ProductSummary[]>;
  getBlogPosts(page: number): Promise<Paginated<PostSummary>>;
  getPostBySlug(slug: string): Promise<Post | null>;
  getPostsByCategory(cat: string): Promise<PostSummary[]>;
  getFaqEntries(): Promise<FaqEntry[]>;                // dùng cho RAG chatbot + FAQ page
  getPrivacyPolicyCurrent(): Promise<PrivacyPolicyVersion>;
}
```
- Gọi qua `fetch(omnigemApiBaseUrl + "/public/...", { next: { revalidate, tags } })` với **token đọc public** (server-only env; không quyền ghi, không quyền chạm endpoint PII — RBAC/SoD HLD B.6, ADR-001).
- **ISR + on-publish webhook:** khi editor publish (UC-17.4/UC-18.4), `omnigem-api` gọi webhook `revalidateTag()` để bài/SP lên live trong cửa sổ revalidate mà không redeploy (US-F1 AC1, US-I4 AC1).
- Field register bám **ERD**: `Product(code, name, jade_type, origin, certification_ref, price_vnd, contact_for_price, media_refs)`, `ElementTag(menh)`, `Collection`. Tất cả **Public** (không PII).

### 8.2 API client (gọi TẦNG 2 từ widget)
`leadClient`, `chatClient`, `dsrClient` bọc `fetch` same-origin, gắn CSRF/origin header, parse theo schema `ChatReply`/`LeadAccepted`/`DsrAccepted`, chuẩn hóa lỗi (400/403/429/503) thành trạng thái UI.

### 8.3 Contract các Route Handler (TẦNG 2 — customer-facing sở hữu)
Bám **api-spec.yaml** (tham khảo) — giữ nguyên hợp đồng:

| Route | UC | Input | Output | Quy tắc |
|---|---|---|---|---|
| `POST /api/session` | chatbot init | botChallengeToken | `{ sessionToken }` | H-1: ký HMAC, bound challenge |
| `POST /api/lead` | UC-10 | `LeadInput` | `201 LeadAccepted` / 503 + zaloUrl | origin→rateLimit→Zod→consent→**ghi Consent+Lead nguyên tử** |
| `POST /api/chat` | UC-11..13 | `ChatInput` | `200 ChatReply` | verify challenge/sig → scrub PII → Gemini loop → fallback Zalo |
| `POST /api/dsr` | UC-16 | `DsrInput` | `202 DsrAccepted{verificationRequired}` | intake; không fulfil ở client-facing |
| `POST /api/collect` | UC-21 | `AnalyticsEvent` | `204` | consent-gated, no PII |

Middleware cắt ngang dùng chung: `assertSameOrigin`, `rateLimit`, `NO_STORE` header, CSP (middleware.ts). Các Route Handler PII này là **BFF forward** tới PII endpoint của `omnigem-api` (server-to-server); domain services (Lead/Consent/ProductQuery/DSR) + DB nằm **trong `omnigem-api`**, không hiện thực ở app Next.js (ADR-001).

---

## 9. SEO (Epic E, I) — kênh tăng trưởng cốt lõi

- **Render:** SSG/ISR mọi trang public để crawler nhận HTML đầy đủ (NFR-3).
- **Metadata:** dùng `generateMetadata` của Next cho title/description/canonical từng trang; OG/Twitter cho Product & Blog (US-I2 AC4).
- **JSON-LD:** `ProductJsonLd` (US-E1 AC1) mỗi Product; `OrganizationJsonLd` toàn site (US-E1 AC2); `Article/BlogPosting` mỗi bài (US-I2 AC2).
- **Sitemap/robots:** `app/sitemap.ts` sinh động từ ContentRepository (SP + bài + archive danh mục); `robots.ts` cho phép index trang public, **chặn** `/api/*` và trang chứa PII.
- **Liên kết nội bộ:** blog ↔ collections/sản phẩm (US-I2 AC3, US-I3) tăng topical authority.

---

## 10. Hiệu năng & Core Web Vitals (NFR-1, NFR-2; nfr-targets §2)

Mục tiêu p75 mobile 4G: **LCP ≤ 2.5s · INP ≤ 200ms · CLS ≤ 0.1**; Lighthouse ≥ 90 (CI budget trên Home/Collections/Product).

| Kỹ thuật | Áp dụng |
|---|---|
| `next/image` + Cloudinary `f_auto,q_auto` | Ảnh ngọc nặng; hero LCP ≤ 200KB, AVIF/WebP |
| `priority`/preload cho hero | Hạ LCP |
| Reserve dimension ảnh/video | Chống CLS |
| Lazy-load gallery dưới nếp gấp | Product ≤ 1.0MB initial viewport |
| Lọc client trên tập prefetch | INP thấp, không round-trip |
| Video: poster + click-to-play | Không autoplay audio |
| Code-split widget nặng (chatbot) | Dynamic import, không chặn LCP trang |

---

## 11. Accessibility (NFR-5; nfr-targets §5)
WCAG 2.1 AA (PROPOSED FIRM): label + error announcement cho LeadForm; CTA/chatbot keyboard operable, focus trap, **ARIA live region** cho tin nhắn bot; `MenhFilter` dùng được bằng bàn phím; contrast ≥ 4.5:1 trên palette tối/ngọc.

---

## 12. Bảo mật phía client (HLD B.5, M-3)
- **CSP chặt** (middleware/`next.config`): allowlist `self` + origin cần thiết (Cloudinary, Zalo widget, GA4, Meta Pixel, Turnstile). Không `unsafe-inline` nếu tránh được.
- HSTS, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy` hạn chế.
- Origin/CSRF check trên mọi POST (TB-1).
- **Không** PII trong URL/query string, log client, hay analytics event.
- Bot challenge + session ký HMAC bảo vệ `/api/chat` (H-1).

---

## 13. Analytics (Epic G, UC-21; HLD Flow 3)

`lib/analytics/events.ts` chuẩn hóa 4 event, **không PII** (chỉ productId + tên event):

| Event | Nguồn | Khi |
|---|---|---|
| `view_product` | UC-03 | Product Detail tải (sau consent analytics) |
| `click_contact` {channel} | UC-08, UC-09 | Chạm CTA/widget chat (Messenger/Zalo) |
| `submit_lead_form` | UC-10 | Lead 201 |
| `chatbot_handoff` | UC-13 | Chatbot tạo handoff |

Tag GA4/Pixel **chỉ load sau consent analytics**. Tùy chọn proxy `/api/collect` (first-party relay) giảm rò rỉ định danh client.

---

## 14. Trạng thái UI & triết lý "không ngõ cụt"
Mọi trạng thái loading/empty/error đều có lối vào phễu:
- Empty collection → `EmptyStateFunnel` (Zalo/chatbot).
- Lead store down (503) → CTA Zalo fallback.
- Gemini down/trần chi tiêu → chatbot `fallback: true` → CTA Zalo.
- Cuối bài blog → `FunnelCTABlock`.
- 404 SP → gợi ý collection + CTA, không dead-end.

---

## 15. Chiến lược test (customer-facing)
- **Unit:** component (ProductCard, MenhFilter, ConsentGate), `dobToMenh`, Zod schema, mappers ContentRepository. Mục tiêu ≥ 80% (testing rule).
- **Integration:** Route Handlers `/api/lead` (atomic consent+lead, reject khi consent≠true, 503 fallback), `/api/chat` (scrub trước gọi vendor, fallback khi vendor lỗi), `/api/dsr` (intake + verificationRequired).
- **E2E (Playwright):** luồng phễu then chốt — Product→Zalo CTA, LeadForm submit, chatbot handoff, consent gate chặn khi chưa tick.
- **SEO/CWV:** Lighthouse CI budget; kiểm JSON-LD; snapshot sitemap.
- **Quy ước PII:** **chỉ dữ liệu tổng hợp** trong fixture/doc (US-C1 AC4; api-spec "all examples SYNTHETIC").

---

## 16. Phân rã theo giai đoạn (bám roadmap PRD §4)

| Phase | Customer-facing deliverable |
|---|---|
| **P0 Foundation** | Khung Next.js App Router, design system/tokens, shell layout, ContentRepository interface (mock), CSP/security headers, sitemap/robots, deploy Vercel |
| **P1 Gallery** | Home, Collections + MệnhFilter, Product Detail (gallery zoom, FengShuiMatch, JSON-LD), Brand story, ảnh Cloudinary, SEO cơ bản |
| **P2 Funnel** | ContactCTA + ContactFloatingWidget (Messenger-first) + ContactChannelResolver, LeadForm + `/api/lead`, ConsentGate, GA4/Pixel + 4 event, hạ tầng blog (`/blog`, `/blog/[slug]`, archive, Article JSON-LD) |
| **P3 Chatbot** | ChatbotWidget + `/api/session` + `/api/chat` (Gemini function calling + RAG trên CMS FAQ/catalog + scrubber + handoff), micro-notice consent, fallback Zalo |
| **P4 Growth** | Sản xuất nội dung blog SEO quy mô, remarketing, A/B test CTA, tối ưu CWV sâu |

Privacy/DSR (Epic H): `/privacy` + ConsentGate đi cùng **P2** (phụ thuộc lead form); `/privacy/dsr` + `/api/dsr` intake cùng **P2/P3**.

---

## 17. Rủi ro & câu hỏi mở còn ảnh hưởng customer-facing

| # | Vấn đề | Trạng thái / hành động |
|---|---|---|
| 1 | **Cú pháp prefill deep-link Zalo** chưa kiểm chứng | **BLOCKING US-B1** — xác nhận tài liệu Zalo Developer trước khi build P2 (PRD §8.1) |
| 2 | UX consent cho DOB | Chờ compliance xác nhận nội dung notice (PRD §8.2) |
| 3 | **Reconcile vendor Gemini** trong orchestration/threat-model | Server `/api/chat` phải dùng Gemini + DPA no-training/zero-retention với Google (PRD §8.4). Widget client không đổi |
| 4 | Ngưỡng giá hybrid (hiện giá vs "liên hệ") | Cần mốc VND từ PO (PRD §8.6) — ảnh hưởng render Product Detail |
| 5 | Mục tiêu CWV/uptime/a11y/ngân sách | nfr-targets là PROPOSED, chờ PO chốt (PRD §8.8) |
| 6 | Base URL + schema public endpoint của `omnigem-api` | Topology đã chốt (ADR-001: gộp 1 backend + RBAC); schema cụ thể chốt ở Design — `ContentRepository` giữ interface ổn định + mock để không chặn P0/P1 |

---

## 18. Ma trận truy vết (US → thành phần chính)

| US | Thành phần customer-facing |
|---|---|
| US-A1 Home | `app/(marketing)/page.tsx`, HeroSection, FeaturedGrid |
| US-A2 Collections/lọc | `collections/page.tsx`, MenhFilter, CategoryFilter, EmptyStateFunnel |
| US-A3 Product Detail | `products/[slug]/page.tsx`, ProductGallery, FengShuiMatch, 2×CTA, ProductJsonLd |
| US-A4 Brand | `brand-story/page.tsx` |
| US-B1/B2 handoff | ContactCTA, ContactFloatingWidget, ContactChannelResolver (Messenger-first) |
| US-C1 Lead | LeadForm, `/api/lead`, ConsentGate |
| US-D1/2/3 Chatbot | ChatbotWidget, `/api/session`, `/api/chat` (Gemini), HandoffCard |
| US-E1/E2 SEO | seo/*, sitemap.ts, robots.ts, generateMetadata |
| US-G1 Analytics | analytics/*, `/api/collect` |
| US-H1/2/3 Privacy | `/privacy`, `/privacy/dsr`, `/api/dsr`, ConsentProvider |
| US-I1..4 Blog | `blog/*`, ArticleJsonLd, FunnelCTABlock |
