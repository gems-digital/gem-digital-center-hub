# OmniGem Gallery — Solution Design chi tiết: Epic A — Duyệt & Khám phá sản phẩm (UC-01..04)

Trạng thái: **ĐÃ CHỐT (APPROVED)** — 2026-10-15 (PO duyệt). Tác giả: solution-architect. Áp dụng ADR-001 (backend topology) + ADR-002 (kênh handoff & cấu hình động).
Nguồn sự thật: `omni-web/prd/PRD.md` (Epic A, US-A1..A4), `omni-web/usecase/use-cases.md` (UC-01..04), `omni-web/design/hld.md` (A.2 render strategy).
Tài liệu cha: `omni-web/design/solution-design-customer-facing.md` (§6).

> **Quy ước:** Tài liệu này ưu tiên **visualize** — mỗi phần thiết kế đi kèm ít nhất một sơ đồ. Vendor chatbot đã reconcile sang **Gemini**; nội dung đọc qua `ContentRepository` → **public endpoint của `omnigem-api`** (ADR-001), KHÔNG chọc DB trực tiếp.

---

## 0. Mục lục sơ đồ (diagram index)

| # | Sơ đồ | Mục |
|---|---|---|
| D1 | Bản đồ Epic A (cây use case) | §1 |
| D2 | Route map + chiến lược render | §2 |
| D3 | Mô hình dữ liệu Epic A (ERD subset) | §3 |
| D4 | TypeScript domain types (class) | §3.1 |
| D5 | Data fetching ISR + on-publish webhook (sequence) | §4 |
| D6 | UC-01 Home — cây component | §5 |
| D7 | UC-01 Home — sequence render + LCP | §5 |
| D8 | UC-02 Collections — cây component | §6 |
| D9 | UC-02 Collections — state machine bộ lọc | §6 |
| D10 | UC-02 Collections — sequence lọc client | §6 |
| D11 | UC-03 Product Detail — cây component | §7 |
| D12 | UC-03 Product Detail — sequence SSG/ISR | §7 |
| D13 | UC-03 ProductGallery — state máy zoom | §7 |
| D14 | UC-03 — cây quyết định hiển thị giá hybrid | §7 |
| D15 | UC-04 Brand — cây component | §8 |
| D16 | Navigation graph giữa các UC | §9 |
| D17 | "Không ngõ cụt" — cây quyết định phễu | §10 |
| D18 | Bản đồ SEO/JSON-LD | §11 |
| D19 | Ngân sách CWV chú giải trên trang | §12 |
| D20 | Cấu trúc thư mục lát cắt Epic A | §13 |

---

## 1. Bản đồ Epic A (D1)

```mermaid
flowchart TB
    v(("Khách<br/>truy cập")):::actor

    subgraph EPICA["EPIC A — DUYỆT & KHÁM PHÁ (Boundary 1, SSG/ISR)"]
      direction TB
      u1(["UC-01 Xem trang chủ<br/>/ (ISR 60m)"]):::uc
      u11(["UC-01.1 Hero + Featured"]):::sub
      u12(["UC-01.2 → chi tiết SP"]):::sub

      u2(["UC-02 Collections + lọc mệnh<br/>/collections (ISR 30m)"]):::uc
      u21(["UC-02.1 Chọn danh mục"]):::sub
      u22(["UC-02.2 Lọc ngũ hành"]):::sub
      u23(["UC-02.3 Empty → CTA phễu"]):::subx

      u3(["UC-03 Chi tiết SP<br/>/products/[slug] (ISR 30m)"]):::uc
      u31(["UC-03.1 Gallery zoom"]):::sub
      u32(["UC-03.2 Thông số/xuất xứ/chứng nhận"]):::sub
      u33(["UC-03.3 Khối khớp phong thủy"]):::sub
      u34(["UC-03.4 Social proof"]):::subx
      u35(["UC-03.5 CTA Chat / Hỏi AI"]):::sub

      u4(["UC-04 Brand & Certification<br/>/brand-story (ISR 60m)"]):::uc
      u41(["UC-04.1 Câu chuyện nguồn gốc"]):::sub
      u42(["UC-04.2 Giải thích kiểm định lab"]):::sub
    end

    funnel{{"Phễu: Messenger/Zalo · Chatbot(Gemini) · LeadForm"}}:::funnel

    v --> u1 & u2 & u3 & u4
    u1 --> u11 & u12
    u2 --> u21 & u22
    u2 -. extend .-> u23
    u3 --> u31 & u32 & u33 & u35
    u3 -. extend .-> u34
    u4 --> u41 & u42
    u12 -. tới .-> u3
    u23 -. tới .-> funnel
    u35 -. tới .-> funnel

    classDef actor fill:#e3f2fd,stroke:#1565c0;
    classDef uc fill:#f3e5f5,stroke:#6a1b9a;
    classDef sub fill:#fff3e0,stroke:#ef6c00;
    classDef subx fill:#fff3e0,stroke:#ef6c00,stroke-dasharray:4 3;
    classDef funnel fill:#e8f5e9,stroke:#2e7d32;
```

**Đọc:** UC-01..04 đều là trang **public SSG/ISR** (không PII, cache CDN được). Hai nhánh nét đứt (`extend`) là empty-state và social-proof — chỉ xuất hiện có điều kiện. Mọi đường đều đổ về **phễu** hoặc sang Product Detail.

---

## 2. Route map + chiến lược render (D2)

```mermaid
flowchart LR
    subgraph ROUTES["app/(marketing)/ (RSC)"]
      r1["/ page.tsx"]:::ssg
      r2["/collections page.tsx"]:::ssg
      r3["/products/[slug] page.tsx"]:::ssg
      r4["/brand-story page.tsx"]:::ssg
    end
    cr["ContentRepository<br/>(HTTP client, token đọc public)"]:::fn
    subgraph API["omnigem-api — public endpoints (ADR-001)"]
      e1["GET /public/products"]:::api
      e2["GET /public/collections"]:::api
      rbac["RBAC: chỉ bảng public"]:::fn
    end
    db[("1 Postgres<br/>(public + PII tables)")]:::db
    gsp["generateStaticParams()"]:::fn
    isr["ISR revalidate + webhook revalidateTag"]:::fn

    r1 -->|getFeaturedProducts| cr
    r2 -->|getCollections + getProductsByCollection| cr
    r3 -->|getProductBySlug| cr
    r3 --- gsp
    gsp -->|getAllProductSlugs| cr
    r4 -->|getBrandContent| cr
    cr --> e1 & e2 --> rbac --> db
    r1 & r2 & r3 & r4 --- isr

    classDef ssg fill:#e8f5e9,stroke:#2e7d32;
    classDef api fill:#e3f2fd,stroke:#1565c0;
    classDef db fill:#eceff1,stroke:#455a64;
    classDef fn fill:#f3e5f5,stroke:#6a1b9a;
```

> **Topology (ADR-001):** `page.tsx (RSC) → ContentRepository → omnigem-api /public/* → Postgres`. Next.js **không chọc DB trực tiếp**; endpoint public của `omnigem-api` chạy dưới DB role least-privilege (chỉ `SELECT` bảng public — không bao giờ chạm bảng PII trong cùng DB).

| Route | UC | Strategy | Revalidate | Lý do (HLD A.2) |
|---|---|---|---|---|
| `/` | UC-01 | SSG + ISR | 60m (+webhook) | Nội dung tuyển chọn, cần CWV cao nhất |
| `/collections` | UC-02 | SSG + ISR, **lọc client** | 30m | Lọc tức thì, HTML nền crawl được |
| `/products/[slug]` | UC-03 | SSG + ISR + `generateStaticParams` | 30m (+webhook) | Bề mặt SEO cốt lõi, Product JSON-LD |
| `/brand-story` | UC-04 | SSG + ISR | 60m | Trang nội dung |

**Quy tắc cứng:** không trang nào trong Epic A chứa PII → cache CDN đầy đủ. Đây là nền tảng SEO chi phí biên ~0.

---

## 3. Mô hình dữ liệu Epic A (D3 — ERD subset)

```mermaid
erDiagram
    COLLECTION ||--o{ PRODUCT : contains
    ELEMENT_TAG ||--o{ PRODUCT_ELEMENT : classifies
    PRODUCT ||--o{ PRODUCT_ELEMENT : "tagged with"

    COLLECTION {
        uuid   id PK
        string slug
        string name
        string description
    }
    PRODUCT {
        uuid    id PK
        uuid    collection_id FK
        string  slug
        string  code
        string  name
        string  jade_type
        string  origin
        string  certification_ref
        decimal price_vnd "nullable = contact-for-price"
        boolean contact_for_price
        json    media_refs "Cloudinary ids (Public)"
        boolean featured "hiển thị ở Home"
    }
    ELEMENT_TAG {
        uuid   id PK
        string menh "Kim|Moc|Thuy|Hoa|Tho"
    }
    PRODUCT_ELEMENT {
        uuid product_id FK
        uuid element_tag_id FK
    }
    PRODUCT ||--o{ TESTIMONIAL : "có bằng chứng (CHỐT: bảng riêng)"
    TESTIMONIAL {
        uuid    id PK
        uuid    product_id FK "nullable — testimonial cấp brand dùng chung"
        string  quote
        string  author_display "ẩn danh hoá, vd 'Chị N., Hà Nội'"
        json    media_refs "Cloudinary ảnh/video khách (Public)"
        int     rating "1..5 (nullable)"
        string  source "manual|zalo|facebook"
        string  consent_ref "release/đồng ý đăng của khách (bắt buộc nếu có tên/ảnh thật)"
        string  status "draft|published (duyệt trước khi hiện)"
        datetime published_at
    }
    SITE_CONFIG {
        string key PK "vd price_contact_threshold_vnd | primary_contact_channel"
        string value "flag động, sửa qua React-Admin, không redeploy"
    }
```

> Thực thể catalog Epic A là **Public**. **CHỐT (điểm 3): `social proof` tách thành bảng `TESTIMONIAL` riêng** — có `status` để duyệt, `source`, `media_refs`, và `consent_ref` (bắt buộc khi testimonial chứa tên/ảnh thật của khách → công bố dữ liệu cá nhân người khác cần release, PDPD). `SITE_CONFIG` giữ các **flag động** (điểm 2 & 4): ngưỡng giá "liên hệ" và kênh handoff chính.

### 3.1 TypeScript domain types (D4)

```mermaid
classDiagram
    class ProductSummary {
      +string slug
      +string code
      +string name
      +string jadeType
      +MediaRef cover
      +number|null priceVnd
      +boolean contactForPrice
      +Menh[] menhTags
      +boolean featured
    }
    class Product {
      +string slug
      +string code
      +string name
      +string jadeType
      +string origin
      +string certificationRef
      +number|null priceVnd
      +boolean contactForPrice
      +MediaRef[] media
      +Menh[] menhTags
      +string fengShuiNote
      +Testimonial[] testimonials
    }
    class Collection {
      +string slug
      +string name
      +string description
      +ProductSummary[] products
    }
    class MediaRef {
      +string cloudinaryId
      +"image"|"video" kind
      +number width
      +number height
      +string? posterId
    }
    class Menh {
      <<enum>>
      Kim
      Moc
      Thuy
      Hoa
      Tho
    }
    Collection "1" o-- "*" ProductSummary
    Product "1" o-- "*" MediaRef
    Product ..> Menh
    ProductSummary ..> Menh
```

`MediaRef` giữ `width/height` để **reserve dimension** (chống CLS). `priceVnd=null & contactForPrice=true` → render "Liên hệ để được tư vấn".

---

## 4. Data fetching: ISR + on-publish webhook (D5)

```mermaid
sequenceDiagram
    autonumber
    participant Cr as Crawler/Khách
    participant Edge as Vercel Edge/CDN
    participant RSC as Server Component (page.tsx)
    participant CR as ContentRepository
    participant API as omnigem-api /public/*
    participant DB as Postgres (role public_read)
    participant Ed as Content Editor (React-Admin)

    Note over Cr,DB: Đường đọc (cache hit — thường gặp)
    Cr->>Edge: GET /products/jade-bangle-01
    Edge-->>Cr: HTML tĩnh đã render (CWV tối ưu)

    Note over RSC,DB: Build/Revalidate (cache miss hoặc hết hạn)
    RSC->>CR: getProductBySlug(slug)
    CR->>API: fetch(/public/products/slug, {next:{revalidate:1800, tags:["product:slug"]}})
    API->>DB: SELECT (chỉ bảng public — RBAC)
    DB-->>API: row
    API-->>CR: Product JSON (Public)
    CR-->>RSC: Product (đã map → domain type)
    RSC-->>Edge: HTML + ISR cache

    Note over Ed,Edge: Editor publish (UC-17.4) → lên live không redeploy
    Ed->>API: Publish sản phẩm (admin endpoint, auth+RBAC)
    API->>RSC: webhook POST /api/revalidate?tag=product:slug
    RSC->>Edge: revalidateTag("product:slug")
    Edge-->>Edge: Xoá cache, lần GET sau render mới
```

**Điểm chốt:** khách luôn nhận HTML tĩnh (bước 2); gọi `omnigem-api` chỉ xảy ra khi miss/hết hạn/được webhook kích (US-F1 AC1 — publish lên live trong cửa sổ revalidate, không redeploy). Next.js không chọc DB; `omnigem-api` chạm DB dưới role `public_read` nên đường render public **không thể** đọc bảng PII (ADR-001).

---

## 5. UC-01 — Home `/` (US-A1)

### 5.1 Cây component (D6)

```mermaid
flowchart TB
    page["page.tsx (RSC, async)"]:::rsc
    layout["layout.tsx — Header/Footer, ConsentProvider, ContactFloatingWidget(Messenger-first), ChatbotWidget"]:::rsc
    hero["HeroSection (LCP)"]:::rsc
    featured["FeaturedGrid"]:::rsc
    card["ProductCard × N"]:::rsc
    img["next/image (Cloudinary, priority)"]:::cli
    orgld["OrganizationJsonLd"]:::seo

    layout --> page
    page --> hero --> img
    page --> featured --> card
    card -->|link| detail["→ /products/[slug] (UC-01.2)"]:::link
    page --> orgld

    classDef rsc fill:#e8f5e9,stroke:#2e7d32;
    classDef cli fill:#fff3e0,stroke:#ef6c00;
    classDef seo fill:#e3f2fd,stroke:#1565c0;
    classDef link fill:#ede7f6,stroke:#6a1b9a;
```

### 5.2 Sequence render + LCP (D7)

```mermaid
sequenceDiagram
    autonumber
    participant B as Browser (mobile 4G)
    participant E as Vercel Edge
    participant CL as Cloudinary

    B->>E: GET /
    E-->>B: HTML (hero markup + preload hero img)
    B->>CL: GET hero.avif (f_auto,q_auto, ≤200KB)
    CL-->>B: Ảnh hero → LCP paint ≤ 2.5s (p75)
    Note over B: FeaturedGrid ảnh lazy (dưới nếp gấp)
    B->>B: Hydrate widget (dynamic import, không chặn LCP)
```

**AC → thiết kế:** AC1 (hero + featured trong budget CWV) → §12; AC2 (mỗi card link tới detail) → `ProductCard` bọc `<Link>`.

---

## 6. UC-02 — Collections + lọc mệnh (US-A2)

### 6.1 Cây component (D8)

```mermaid
flowchart TB
    page["collections/page.tsx (RSC)<br/>prefetch toàn bộ ProductSummary + tags"]:::rsc
    board["CollectionsBoard ('use client')"]:::cli
    cat["CategoryFilter (UC-02.1)"]:::cli
    menh["MenhFilter (UC-02.2) Kim/Mộc/Thủy/Hỏa/Thổ"]:::cli
    grid["ProductGrid"]:::cli
    card["ProductCard × N"]:::cli
    empty["EmptyStateFunnel (UC-02.3, extend)"]:::funnel

    page --> board
    board --> cat & menh
    board --> grid --> card
    board -.->|0 kết quả| empty
    empty -->|CTA| funnel{{"Hỏi chatbot / Nhắn Messenger"}}:::funnel

    classDef rsc fill:#e8f5e9,stroke:#2e7d32;
    classDef cli fill:#fff3e0,stroke:#ef6c00;
    classDef funnel fill:#e8f5e9,stroke:#2e7d32;
```

**Quyết định kiến trúc:** server prefetch tập sản phẩm (nhỏ, showroom) → `CollectionsBoard` lọc **phía client** → INP ≤ 200ms, không round-trip, HTML nền vẫn crawl được.

### 6.2 State machine bộ lọc (D9)

```mermaid
stateDiagram-v2
    [*] --> AllProducts: tải trang (toàn bộ SP)
    AllProducts --> Filtered: chọn danh mục / mệnh
    Filtered --> Filtered: đổi bộ lọc (giao nhau category ∩ mệnh)
    Filtered --> AllProducts: xoá bộ lọc
    Filtered --> Empty: không SP khớp
    Empty --> Filtered: đổi/nới bộ lọc
    Empty --> Funnel: chạm CTA phễu
    AllProducts --> Detail: chạm ProductCard
    Filtered --> Detail: chạm ProductCard
    Funnel --> [*]
    Detail --> [*]

    note right of Empty
      US-A2 AC3: empty = CTA phễu,
      KHÔNG phải ngõ cụt
    end note
```

### 6.3 Sequence lọc client (D10)

```mermaid
sequenceDiagram
    autonumber
    participant U as Khách
    participant Board as CollectionsBoard (client)
    participant F as useFilter() (useMemo)
    participant Grid as ProductGrid

    U->>Board: chọn mệnh = "Moc"
    Board->>F: apply({category, menh:"Moc"})
    F->>F: products.filter(p => p.menhTags.includes("Moc") && catMatch)
    alt có kết quả
      F-->>Grid: danh sách đã lọc
      Grid-->>U: render (INP ≤ 200ms, không gọi mạng)
    else rỗng
      F-->>Board: []
      Board-->>U: EmptyStateFunnel (CTA Messenger/chatbot)
    end
```

**AC → thiết kế:** AC1 (lọc danh mục) → `CategoryFilter`; AC2 (lọc mệnh) → `MenhFilter` giao với category; AC3 (empty) → `EmptyStateFunnel`.

---

## 7. UC-03 — Product Detail `/products/[slug]` (US-A3)

### 7.1 Cây component (D11)

```mermaid
flowchart TB
    page["products/[slug]/page.tsx (RSC)<br/>getProductBySlug · notFound()"]:::rsc
    gallery["ProductGallery (UC-03.1, zoom)"]:::cli
    spec["SpecPanel (UC-03.2)<br/>jadeType·size·origin·certRef"]:::rsc
    price["PriceOrContact (hybrid)"]:::rsc
    fengshui["FengShuiMatchBlock (UC-03.3)"]:::cli
    proof["SocialProof (UC-03.4, extend)"]:::rsc
    cta["CtaBar above-the-fold (UC-03.5)"]:::cli
    contact["ContactCTA(productCode)<br/>→ ContactChannelResolver"]:::cli
    askai["AskAiButton → ChatbotWidget(ctx=code)"]:::cli
    prodld["ProductJsonLd (US-E1)"]:::seo

    page --> gallery & spec & price & fengshui & cta & prodld
    page -.->|có media chứng thực| proof
    cta --> contact & askai
    contact -->|"Messenger-first: m.me/page?ref=code"| mm["Messenger"]:::ext
    contact -.->|"bật sau qua flag"| zalo["Zalo OA"]:::ext
    contact -->|click_contact {channel, productId}| ga["analytics"]:::seo
    gallery --> img["next/image Cloudinary"]:::cli

    classDef ext fill:#e1f5fe,stroke:#0277bd;

    classDef rsc fill:#e8f5e9,stroke:#2e7d32;
    classDef cli fill:#fff3e0,stroke:#ef6c00;
    classDef seo fill:#e3f2fd,stroke:#1565c0;
```

### 7.2 Sequence SSG/ISR + generateStaticParams (D12)

```mermaid
sequenceDiagram
    autonumber
    participant Build as Build/ISR
    participant GSP as generateStaticParams
    participant CR as ContentRepository → omnigem-api
    participant Page as page.tsx
    participant B as Browser

    Build->>GSP: liệt kê params
    GSP->>CR: getAllProductSlugs()
    CR-->>GSP: ["jade-bangle-01", ...]
    loop mỗi slug
      Build->>Page: render(slug)
      Page->>CR: getProductBySlug(slug)
      alt tồn tại
        CR-->>Page: Product
        Page-->>Build: HTML + ProductJsonLd (ISR 30m)
      else null
        Page-->>Build: notFound() (404 có CTA, không dead-end)
      end
    end
    B->>Build: GET /products/jade-bangle-01
    Build-->>B: HTML tĩnh (ISR cache)
```

### 7.3 ProductGallery — state máy zoom (D13)

```mermaid
stateDiagram-v2
    [*] --> Thumbnails: tải (ảnh cover ưu tiên, còn lại lazy)
    Thumbnails --> ActiveImage: chọn thumbnail
    ActiveImage --> Zoomed: chạm/giữ để zoom (US-A3 AC1)
    Zoomed --> ActiveImage: thả / đóng zoom
    ActiveImage --> VideoPlay: chọn media video
    VideoPlay --> ActiveImage: dừng (poster + click-to-play, không autoplay audio)
    ActiveImage --> [*]

    note right of Zoomed
      Ảnh zoom tải độ phân giải cao
      ON-DEMAND (không nằm trong
      budget 1.0MB initial viewport)
    end note
```

### 7.4 Cây quyết định hiển thị giá (D14 — CHỐT: mặc định hiện giá, threshold động)

Band giá thực tế **500k–5tr** → **mặc định HIỆN GIÁ**; "liên hệ" chỉ là ngoại lệ cho hàng độc bản. Thứ tự ưu tiên: override per-product trước, rồi mới tới threshold động.

```mermaid
flowchart TD
    start["Render Product (US-A3 AC1)"] --> q0{contact_for_price<br/>đặt tường minh?}
    q0 -->|"true (override độc bản)"| contact["'Liên hệ để được tư vấn'<br/>→ CTA Messenger / Hỏi AI"]:::contact
    q0 -->|"false (override hiện giá)"| showprice["Hiển thị giá VND"]:::price
    q0 -->|"null (để trống)"| q2{price_vnd ≥ threshold?}
    q2 -->|Yes| contact
    q2 -->|"No (mặc định band 500k–5tr)"| showprice
    cfg[("SiteConfig.price_contact_threshold_vnd<br/>⚙️ flag động — React-Admin, không redeploy")]:::cfg -.->|cấp threshold| q2
    contact --> cta["CtaBar (UC-03.5)"]
    showprice --> cta

    classDef contact fill:#fff3e0,stroke:#ef6c00;
    classDef price fill:#e8f5e9,stroke:#2e7d32;
    classDef cfg fill:#e3f2fd,stroke:#1565c0;
```

> **CHỐT (điểm 2):** threshold là **flag động** trong `SITE_CONFIG`, không hard-code. Ở band 500k–5tr, hầu hết SP rơi vào nhánh "hiện giá"; `contact_for_price` chỉ override cho hàng hiếm. (PRD §8.6 — không còn chờ một con số cứng từ PO vì giờ là cấu hình động.)

### 7.5 Ghi chú FengShuiMatchBlock (UC-03.3) — ranh giới PII
- Mặc định **chỉ hiển thị** tag ngũ hành của SP + ghi chú khớp mệnh (dữ liệu Public, không PII).
- Tùy chọn progressive-enhancement: ô "nhập năm sinh để xem có hợp" → tính **thuần client** bằng `lib/fengshui/dobToMenh.ts`, **không lưu, không gửi server** → *không* kích hoạt ConsentRecord.
- Muốn tư vấn sâu/cá nhân hóa → điều hướng sang **ChatbotWidget (UC-11)**, nơi DOB được xử lý **có consent** (US-H2). Ranh giới này giữ Product Detail là trang tĩnh, không PII.

**AC → thiết kế:** AC1 (gallery/spec/origin/cert/giá-hoặc-liên-hệ/khối phong thủy) → §7.1+7.4; AC2 (2 CTA above-the-fold mobile) → `CtaBar`; AC3 (social proof có điều kiện) → `SocialProof` «extend».

---

## 8. UC-04 — Brand & Certification `/brand-story` (US-A4, D15)

```mermaid
flowchart TB
    page["brand-story/page.tsx (RSC, ISR 60m)"]:::rsc
    origin["OriginStory (UC-04.1)"]:::rsc
    lab["LabCertExplainer (UC-04.2)"]:::rsc
    certlink["→ trang trưng bày Chứng nhận"]:::link
    trust["TrustCopy (tái dùng fanpage §2 FAQ Q1)"]:::rsc
    orgld["OrganizationJsonLd"]:::seo

    page --> origin & lab & trust & orgld
    lab --> certlink

    classDef rsc fill:#e8f5e9,stroke:#2e7d32;
    classDef seo fill:#e3f2fd,stroke:#1565c0;
    classDef link fill:#ede7f6,stroke:#6a1b9a;
```

**Nhất quán niềm tin (US-A4 AC1):** `TrustCopy` dùng chung một nguồn với chatbot FAQ (US-D2) và copy fanpage — tránh lệch thông điệp "ngọc tự nhiên 100% + chứng nhận lab".

---

## 9. Navigation graph giữa các UC (D16)

```mermaid
flowchart LR
    home["UC-01 Home /"]:::p
    coll["UC-02 Collections"]:::p
    prod["UC-03 Product Detail"]:::p
    brand["UC-04 Brand"]:::p
    funnel{{"Phễu: Messenger/Zalo·Chatbot·Lead"}}:::f

    home -->|featured card| prod
    home -->|nav| coll
    home -->|nav| brand
    coll -->|card| prod
    coll -->|empty| funnel
    prod -->|CTA| funnel
    prod -->|liên quan| prod
    brand -->|cert / CTA| prod
    brand --> funnel

    classDef p fill:#f3e5f5,stroke:#6a1b9a;
    classDef f fill:#e8f5e9,stroke:#2e7d32;
```

Mọi node đều có ≥1 đường ra tới **phễu** hoặc Product Detail — hiện thực triết lý "không ngõ cụt" ở cấp điều hướng.

---

## 10. "Không ngõ cụt" — cây quyết định phễu (D17)

```mermaid
flowchart TD
    any["Khách ở bất kỳ trang Epic A nào"] --> q{Trạng thái?}
    q -->|Xem bình thường| cta1["CTA thường trực: ContactFloatingWidget(Messenger-first) + ChatbotWidget"]
    q -->|Collections rỗng| cta2["EmptyStateFunnel → Messenger/chatbot (US-A2 AC3)"]
    q -->|Product 404| cta3["notFound page → gợi ý collection + CTA"]
    q -->|Quan tâm 1 SP| cta4["CtaBar: ContactCTA (Messenger) / Hỏi AI (US-A3 AC2)"]
    cta1 & cta2 & cta3 & cta4 --> done["Vào phễu → lead"]

    classDef x fill:#e8f5e9,stroke:#2e7d32;
```

---

## 11. Bản đồ SEO / JSON-LD (D18)

```mermaid
flowchart TB
    subgraph SITE["Toàn site"]
      org["OrganizationJsonLd (US-E1 AC2)"]:::seo
      sm["sitemap.ts (SP + collections)"]:::seo
      rb["robots.ts (index public, chặn /api)"]:::seo
    end
    home["/ "]:::p --> org
    brand["/brand-story"]:::p --> org
    prod["/products/[slug]"]:::p --> prodld["ProductJsonLd name/image/desc (US-E1 AC1)"]:::seo
    prod --> ogimg["opengraph-image"]:::seo
    home & brand & prod --> meta["generateMetadata title/desc/canonical"]:::seo
    prod --> sm

    classDef p fill:#f3e5f5,stroke:#6a1b9a;
    classDef seo fill:#e3f2fd,stroke:#1565c0;
```

---

## 12. Ngân sách Core Web Vitals chú giải (D19)

```mermaid
flowchart LR
    subgraph P["Product Detail — budget ảnh"]
      hero["cover ≤ 200KB (AVIF/WebP)"]:::g
      vp["initial viewport ≤ 1.0MB"]:::g
      lazy["gallery dưới nếp gấp: lazy"]:::g
      zoom["ảnh zoom: on-demand (ngoài budget)"]:::g
    end
    subgraph M["Mục tiêu p75 mobile 4G"]
      lcp["LCP ≤ 2.5s"]:::m
      inp["INP ≤ 200ms"]:::m
      cls["CLS ≤ 0.1"]:::m
    end
    hero --> lcp
    lazy & vp --> lcp
    zoom --> inp
    rd["reserve width/height từ MediaRef"]:::g --> cls
    filt["lọc client (useMemo)"]:::g --> inp

    classDef g fill:#fff3e0,stroke:#ef6c00;
    classDef m fill:#e8f5e9,stroke:#2e7d32;
```

| Kỹ thuật | Chống | Component |
|---|---|---|
| `next/image priority` + preload hero | LCP | HeroSection, ProductGallery cover |
| Cloudinary `f_auto,q_auto` | LCP/bandwidth | mọi ảnh |
| `width/height` từ `MediaRef` | CLS | mọi `next/image` |
| Lọc client `useMemo` | INP (không round-trip) | CollectionsBoard |
| Dynamic import chatbot | TBT/LCP | layout widgets |
| Video poster + click-to-play | CLS/bandwidth | ProductGallery |

---

## 13. Cấu trúc thư mục lát cắt Epic A (D20)

```mermaid
flowchart TB
    subgraph APP["app/(marketing)/"]
      a1["page.tsx (UC-01)"]
      a2["collections/page.tsx (UC-02)"]
      a3["products/[slug]/page.tsx + not-found.tsx (UC-03)"]
      a4["brand-story/page.tsx (UC-04)"]
    end
    subgraph COMP["components/"]
      p["product/ ProductCard·ProductGallery·SpecPanel·PriceOrContact·FengShuiMatchBlock·SocialProof·CtaBar"]
      col["collections/ CollectionsBoard·CategoryFilter·MenhFilter·ProductGrid·EmptyStateFunnel"]
      mk["marketing/ HeroSection·FeaturedGrid·OriginStory·LabCertExplainer·TrustCopy"]
      seo["seo/ ProductJsonLd·OrganizationJsonLd"]
    end
    subgraph LIB["lib/"]
      cms["content/ ContentRepository·types·mappers (HTTP client → omnigem-api)"]
      fs["fengshui/ dobToMenh.ts"]
      an["analytics/ events.ts (view_product, click_contact)"]
    end
    APP --> COMP --> LIB
```

---

## 14. Hợp đồng props các component chính

```ts
// components/product/ProductCard.tsx
interface ProductCardProps {
  product: ProductSummary;         // slug, name, cover(MediaRef), price|contact, menhTags
  // bọc <Link href={`/products/${slug}`}> (UC-01.2 / UC-02 → UC-03)
}

// components/collections/CollectionsBoard.tsx  ("use client")
interface CollectionsBoardProps {
  products: ProductSummary[];      // prefetch từ server (RSC)
  categories: { slug: string; name: string }[];
  // state: { category?: string; menh?: Menh } → useMemo filter → ProductGrid | EmptyStateFunnel
}

// components/product/CtaBar.tsx  ("use client")
interface CtaBarProps {
  productCode: string;             // prefill ref kênh chat + ngữ cảnh chatbot (US-A3 AC2)
  // ContactCTA(productCode) → ContactChannelResolver + AskAiButton(ChatbotWidget, ctx=productCode)
}

// components/product/PriceOrContact.tsx
interface PriceOrContactProps {
  priceVnd: number | null;
  contactForPrice: boolean;        // cây quyết định D14
}
```

---

## 15. Ma trận truy vết AC → thiết kế/sơ đồ

| US / AC | Thiết kế | Sơ đồ |
|---|---|---|
| US-A1 AC1 hero+featured trong CWV | HeroSection priority + budget | D6, D7, D19 |
| US-A1 AC2 card → detail | ProductCard `<Link>` | D6, D16 |
| US-A2 AC1 lọc danh mục | CategoryFilter | D8, D9, D10 |
| US-A2 AC2 lọc mệnh | MenhFilter (giao category) | D8, D9, D10 |
| US-A2 AC3 empty → CTA | EmptyStateFunnel | D8, D9, D17 |
| US-A3 AC1 gallery/spec/giá/phong thủy | ProductGallery·SpecPanel·PriceOrContact·FengShuiMatchBlock | D11, D13, D14 |
| US-A3 AC2 2 CTA above-the-fold | CtaBar (Messenger + Hỏi AI) | D11, D17 |
| US-A3 AC3 social proof có điều kiện | SocialProof «extend» | D11 |
| US-A4 AC1 brand + cert + trust | OriginStory·LabCertExplainer·TrustCopy | D15 |
| US-E1 AC1/AC2 JSON-LD | ProductJsonLd·OrganizationJsonLd | D18 |
| NFR-1/2 CWV mobile | budget ảnh + ISR + lọc client | D19 |

---

## 16. Điểm cần chốt trước khi code Epic A

| # | Điểm | Trạng thái |
|---|---|---|
| 1 | Schema/base URL public endpoint của `omnigem-api` | **CHỐT topology** (ADR-001: gộp 1 backend + RBAC). `ContentRepository` giữ interface ổn định + mock ở P0/P1; schema cụ thể chốt ở Design. |
| 2 | Ngưỡng giá "liên hệ" | **CHỐT:** mặc định hiện giá (band 500k–5tr); `contact_for_price` override; threshold = flag động `SITE_CONFIG` (ADR-002). |
| 3 | Social proof / testimonial | **CHỐT:** bảng `TESTIMONIAL` riêng (`status` duyệt, `source`, `consent_ref`) — xem D3. |
| 4 | Kênh handoff | **CHỐT:** **Messenger-first** (`m.me/page?ref=code`) qua `ContactChannelResolver` + flag; Zalo OA bật sau khi có doanh thu; webhook auto-lead để P3/P4 (ADR-002). |
| 5 | Reconcile PRD §1 | **CHỐT (PO duyệt, bản sửa R1):** PRD §1 + §8.6 đã cập nhật giá 500k–5tr + lý do "giai đoạn đầu bán qua người, online tính sau". Mô hình phễu/NG1 giữ nguyên. |

**Mô hình vận hành đã xác nhận:** thu lead → chăm sóc + chốt qua người (Messenger) → ship / kéo về cửa hàng → *(tương lai)* bán online. NG1 (không checkout) giữ nguyên ở giai đoạn này.
