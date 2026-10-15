# OmniGem Gallery — Solution Design chi tiết: Epic B & C — Liên hệ qua phễu (UC-08..10)

Trạng thái: **DRAFT** — 2026-10-15. Tác giả: solution-architect. Áp dụng **ADR-001** (backend topology) + **ADR-002** (kênh handoff Messenger-first & cấu hình động).
Nguồn sự thật: `omni-web/prd/PRD.md` (Epic B — US-B1/B2; Epic C — US-C1; Epic H — US-H2; Epic G — US-G1), `omni-web/usecase/use-cases.md` (UC-08..10, UC-15), `omni-web/design/hld.md` (A.3 Luồng 1).
Tài liệu cha: `omni-web/design/solution-design-customer-facing.md` (§7.2, §7.4, §8, §13). Tài liệu anh/em: `solution-design-epic-a-browse-discover.md` (D11/D14/D17 — ContactCTA, giá động, "không ngõ cụt").

> **Quy ước:** Bám đúng phong cách tài liệu Epic A — **visualize-first**: mỗi phần thiết kế đi kèm ít nhất một sơ đồ. Mọi điểm chạm PII đi qua **TẦNG 2 (BFF Route Handler)** → forward tới PII endpoint của `omnigem-api` (ADR-001); widget client **không bao giờ** chạm DB hay giữ secret.

> **GHI CHÚ ĐỒNG BỘ (bắt buộc đọc):** Theo **ADR-002 (ACCEPTED)**, kênh handoff chính là **Facebook Messenger** (`m.me/<page>?ref=<code>`), KHÔNG phải Zalo. Zalo OA **bật sau** khi có doanh thu bằng cách đổi flag `SiteConfig.primaryContactChannel` — không sửa code. Mọi chỗ PRD/HLD gốc ghi "Zalo"/`click_zalo` hiểu theo nghĩa **"kênh chat handoff (Messenger-first, Zalo OA sau)"** và event đã tổng quát hoá thành **`click_contact { channel, productId }`**. **Webhook auto-tạo lead** từ Messenger/Zalo là **P3/P4** (cần Meta app review `pages_messaging`); ở **P2**, lead chỉ tạo qua **LeadForm (UC-10)** và chatbot handoff (UC-13).

---

## 0. Mục lục sơ đồ (diagram index)

| # | Sơ đồ | Mục |
|---|---|---|
| D1 | Bản đồ Epic B/C (cây use case + phễu) | §1 |
| D2 | Route/endpoint map + topology điểm chạm phễu | §2 |
| D3 | Mô hình dữ liệu phễu (ERD subset — PII tables) | §3 |
| D4 | Domain types + Zod schema dùng chung (class) | §3.1 |
| D5 | `ContactChannelResolver` — cây quyết định kênh + giảm cấp 3 tầng | §4 |
| D6 | UC-08 `ContactCTA` — sequence build deep-link + `click_contact` | §5 |
| D7 | UC-09 `ContactFloatingWidget` — cây component + CSP allowlist | §6 |
| D8 | UC-10 `LeadForm` — cây component | §7 |
| D9 | UC-10 — sequence submit (consent+lead nguyên tử, 503 fallback) | §7 |
| D10 | UC-10 — state machine của form | §7 |
| D11 | UC-15 Consent — hai loại consent & ConsentGate | §8 |
| D12 | Ghi nguyên tử ConsentRecord + Lead (transaction) | §8 |
| D13 | "Không ngõ cụt" — cây quyết định phễu (Epic B/C) | §9 |
| D14 | Analytics phễu (`click_contact`, `submit_lead_form`) | §10 |
| D15 | Cấu trúc thư mục lát cắt Epic B/C | §11 |

---

## 1. Bản đồ Epic B/C (D1)

```mermaid
flowchart TB
    v(("Khách<br/>truy cập")):::actor

    subgraph EPICBC["EPIC B & C — LIÊN HỆ QUA PHỄU (Boundary 1)"]
      direction TB
      u8(["UC-08 Nhắn kênh chat về 1 SP<br/>ContactCTA (deep-link)"]):::uc
      u9(["UC-09 Widget chat nổi chung<br/>ContactFloatingWidget"]):::uc
      u10(["UC-10 Gửi form tư vấn/lead<br/>LeadForm + /api/lead"]):::uc
      u101(["UC-10.1 Nhập liên hệ (name, phone/zalo, dob, SP)"]):::sub
      u102(["UC-10.2 Cấp đồng ý (gắn policyVersion)"]):::sub
      u103(["UC-10.3 Xác thực & lưu lead vào CRM"]):::sub
      u104(["UC-10.4 Lỗi lưu → fallback kênh chat"]):::subx
      u15(["UC-15 Cấp đồng ý (Art.11, tái lập được)"]):::ucshared
    end

    resolver["ContactChannelResolver<br/>(SiteConfig.primaryContactChannel)"]:::fn
    mm(("Messenger<br/>m.me?ref=code")):::ext
    zalo(("Zalo OA<br/>bật sau — flag")):::ext
    crm[("CRM built-in<br/>(omnigem-api PII)")]:::db

    v --> u8 & u9 & u10
    u8 --> resolver
    u9 --> resolver
    resolver -->|"mặc định"| mm
    resolver -.->|"flag"| zalo
    u10 --> u101 & u102 & u103
    u10 -. extend .-> u104
    u102 -. include .-> u15
    u103 -->|"lead_form"| crm
    u104 -.-> resolver

    classDef actor fill:#e3f2fd,stroke:#1565c0;
    classDef uc fill:#f3e5f5,stroke:#6a1b9a;
    classDef sub fill:#fff3e0,stroke:#ef6c00;
    classDef subx fill:#fff3e0,stroke:#ef6c00,stroke-dasharray:4 3;
    classDef ucshared fill:#e8f5e9,stroke:#2e7d32;
    classDef fn fill:#f3e5f5,stroke:#6a1b9a;
    classDef ext fill:#e1f5fe,stroke:#0277bd;
    classDef db fill:#eceff1,stroke:#455a64;
```

**Đọc:** Epic B (UC-08/09) = **lối ra kênh chat** (không PII phía web — chỉ mở app chat ngoài, mang `ref=code`). Epic C (UC-10) = **thu lead có PII** → bắt buộc «include» UC-15 (consent Art.11) và ghi nguyên tử vào CRM. Cả hai nhánh đều do **`ContactChannelResolver`** quyết kênh (ADR-002), kể cả nhánh fallback UC-10.4.

**Phân biệt bản chất 2 Epic:**

| | Epic B (UC-08, UC-09) | Epic C (UC-10) |
|---|---|---|
| Bản chất | Điều hướng ra kênh chat ngoài | Thu thập PII tại chỗ |
| PII trên web? | **Không** (chỉ `ref=code`) | **Có** (name, phone/zalo, **dob**) |
| Consent? | Không cần (không thu PII) | **Bắt buộc** (UC-15, Art.11) |
| Endpoint | Không (deep-link client) | `POST /api/lead` (BFF → PII) |
| Lead tạo ở đâu? | P2: không (webhook auto-lead là P3/P4) | Ngay khi submit (`source="lead_form"`) |
| Event | `click_contact{channel,productId}` | `submit_lead_form` |

---

## 2. Route/endpoint map + topology điểm chạm phễu (D2)

```mermaid
flowchart LR
    subgraph CLIENT["TẦNG 1b — CLIENT WIDGETS ('use client')"]
      cta["ContactCTA(productCode) — UC-08"]:::cli
      fw["ContactFloatingWidget — UC-09"]:::cli
      lf["LeadForm — UC-10"]:::cli
      cg["ConsentGate — UC-15"]:::cli
      res["ContactChannelResolver"]:::fn
    end
    subgraph BFF["TẦNG 2 — ROUTE HANDLER (runtime=nodejs, no-store)"]
      api["POST /api/lead"]:::api
      mw["middleware: originCheck · rateLimit · Zod · consentGate"]:::fn
    end
    subgraph APIBE["omnigem-api — PII endpoint (ADR-001)"]
      pii["POST /pii/lead (server-to-server, token PII)"]:::api
      svc["LeadService + ConsentService (1 transaction)"]:::svc
      rbac["RBAC: role PII (không chạm bảng public)"]:::fn
    end
    db[("Postgres — bảng PII<br/>(lead, consent_record)")]:::db
    cfg[("SiteConfig<br/>primaryContactChannel")]:::cfg
    ext["Messenger / Zalo OA (kênh ngoài)"]:::ext

    cta --> res
    fw --> res
    res -.->|"đọc flag (ISR)"| cfg
    res -->|deep-link| ext
    lf --> cg
    lf -->|POST same-origin| api --> mw --> pii --> svc --> rbac --> db
    svc -. 503 .-> lf
    lf -. fallback .-> res

    classDef cli fill:#fff3e0,stroke:#ef6c00;
    classDef api fill:#e3f2fd,stroke:#1565c0;
    classDef svc fill:#f3e5f5,stroke:#6a1b9a;
    classDef fn fill:#f3e5f5,stroke:#6a1b9a;
    classDef db fill:#eceff1,stroke:#455a64;
    classDef cfg fill:#e3f2fd,stroke:#1565c0;
    classDef ext fill:#e1f5fe,stroke:#0277bd;
```

> **Topology (ADR-001):** `LeadForm → /api/lead (BFF) → omnigem-api /pii/lead → Postgres`. Route Handler PII chạy `runtime="nodejs"` + `Cache-Control: no-store`, **không bao giờ** cache edge. `LeadService`/`ConsentService` + DB nằm **trong `omnigem-api`**, không hiện thực ở app Next.js. Nhánh UC-08/09 (Messenger/Zalo) **không đi qua BFF** — chỉ mở deep-link phía client (không PII).

| Route / điểm chạm | UC | Loại | Cache | Lý do |
|---|---|---|---|---|
| `ContactCTA` (client) | UC-08 | Client deep-link | — | Không PII; chỉ build URL `m.me?ref=code` |
| `ContactFloatingWidget` (client) | UC-09 | Client script/plugin | — | Plugin Messenger/Zalo nổi mọi trang |
| `POST /api/lead` | UC-10 | Route Handler (BFF) | **no-store** | Thu PII (name/phone/dob) → forward PII endpoint |
| `SiteConfig.primaryContactChannel` | UC-08/09 | Flag động (ISR) | ISR | Đổi kênh/A-B test không redeploy (ADR-002) |

**Quy tắc cứng:** mọi response của `/api/lead` **không index, không cache**; mọi POST có **origin/CSRF check** (TB-1); **không PII** trong URL/query/log/analytics.

---

## 3. Mô hình dữ liệu phễu (D3 — ERD subset, PII tables)

```mermaid
erDiagram
    LEAD ||--|| CONSENT_RECORD : "ghi nguyên tử cùng TX (Art.11)"
    LEAD }o--o| PRODUCT : "quan tâm (productCode, nullable)"
    PRIVACY_POLICY_VERSION ||--o{ CONSENT_RECORD : "ghim version"

    LEAD {
        uuid     id PK
        string   name "PII — encrypted at rest"
        string   phone "PII enc — regex SĐT VN (ít nhất 1 trong phone/zalo)"
        string   zalo_id "PII enc (nullable)"
        date     dob "PII enc — dùng tư vấn phong thủy (US-C1 AC1)"
        string   product_code "FK mềm tới PRODUCT.code (nullable)"
        string   message "nhu cầu (nullable, scrub trước lưu)"
        string   source "lead_form | chatbot (nguồn sự thật duy nhất)"
        string   contact_channel "kênh gợi ý theo SiteConfig lúc tạo"
        uuid     consent_record_id FK "bắt buộc — không lead nào thiếu consent"
        string   status "new | contacted | qualified | closed (CRM — UC-19)"
        datetime created_at
    }
    CONSENT_RECORD {
        uuid     id PK
        string   policy_version "ghim version Privacy Policy đã hiển thị (US-H2 AC1)"
        string   purpose "lead_follow_up | fengshui_dob"
        string   method "lead_form | chatbot"
        boolean  granted "luôn = true khi tồn tại (consent≠true thì không ghi gì)"
        string   request_ctx_hash "hash ngữ cảnh request (không lưu IP thô)"
        datetime granted_at
    }
    PRIVACY_POLICY_VERSION {
        string   version PK "vd 2026-10-15.r1"
        string   content_ref
        datetime published_at
    }
    SITE_CONFIG {
        string key PK "primaryContactChannel | price_contact_threshold_vnd"
        string value "flag động — React-Admin, không redeploy (ADR-002)"
    }
```

> **Bất biến (HLD Flow 1 bước 4–7):** `LEAD` và `CONSENT_RECORD` **luôn ghi trong cùng một DB transaction** — không bao giờ có lead thiếu consent hoặc ngược lại (no partial commit). `consent_record_id` là **NOT NULL** ở tầng schema để ép bất biến này. Mọi cột PII **encrypted at rest**; bảng này thuộc **role PII** (RBAC), đường render public **không thể** đọc (ADR-001 bất biến 3 lớp). `SITE_CONFIG` tái dùng từ Epic A (D3) — thêm khóa `primaryContactChannel`.

### 3.1 Domain types + Zod schema dùng chung (D4)

```mermaid
classDiagram
    class LeadInput {
      +string name
      +string? phone
      +string? zaloId
      +string dobISO
      +string? productCode
      +string? message
      +ConsentAssertion consent
    }
    class ConsentAssertion {
      +string policyVersion
      +boolean granted
    }
    class LeadAccepted {
      +string leadId
      +"ok" status
    }
    class StoreUnavailable {
      +"store_unavailable" error
      +string fallbackContactUrl
    }
    class ContactChannel {
      <<enum>>
      messenger
      zalo_oa
      zalo_personal
    }
    class ResolvedContact {
      +ContactChannel channel
      +string url
      +string label
      +"prefill"|"ref"|"blank" tier
    }
    LeadInput "1" o-- "1" ConsentAssertion
    LeadInput ..> ContactChannel
```

```ts
// lib/schemas/lead.ts — Zod dùng chung CLIENT (validate trước submit) ↔ SERVER (validate khi nhận)
export const LeadInput = z.object({
  name: z.string().min(1).max(120),
  phone: z.string().regex(VN_PHONE).optional(),
  zaloId: z.string().min(3).max(64).optional(),
  dobISO: z.string().date(),                       // DOB → US-C1 AC1 (tư vấn phong thủy)
  productCode: z.string().max(40).optional(),
  message: z.string().max(2000).optional(),
  consent: z.object({
    policyVersion: z.string().min(1),              // ghim version → US-H2 AC1
    granted: z.literal(true),                      // literal(true): consent≠true ⇒ Zod fail, không xử lý PII
  }),
}).refine(v => !!(v.phone || v.zaloId), { message: "Cần ít nhất SĐT hoặc Zalo ID" }); // US-C1 AC1
```

> **1 schema, 2 nơi** (HLD B.1 fail-fast): `consent.granted` là `z.literal(true)` nên request thiếu/sai consent **bị từ chối ngay ở biên** — hiện thực quy tắc "consent≠true thì không ghi gì" (UC-10 ngoại lệ). `.refine()` ép "ít nhất 1 trong phone/zalo" (US-C1 AC1).

---

## 4. `ContactChannelResolver` — cây quyết định kênh + giảm cấp 3 tầng (D5)

Trái tim của ADR-002: kênh handoff **trừu tượng hoá** sau một resolver, do flag động quyết → đổi kênh/A-B test **không redeploy**. Widget (`ContactCTA`, `ContactFloatingWidget`, fallback của `LeadForm`) **không biết** kênh cụ thể — chỉ gọi `resolve(productCode)`.

```mermaid
flowchart TD
    start["resolve(productCode?)"] --> cfg{SiteConfig<br/>primaryContactChannel}
    cfg -->|"messenger (mặc định)"| mm["m.me/&lt;page&gt;?ref=&lt;productCode&gt;<br/>tier=ref (Meta có tài liệu, chắc chắn)"]:::ok
    cfg -->|"zalo_oa (bật sau)"| zq{Prefill Zalo<br/>đã kiểm chứng tài liệu?}
    cfg -->|"zalo_personal (stopgap)"| zp["zalo.me/&lt;sdt&gt; — chat trống<br/>⚠ nợ governance, tránh"]:::warn
    zq -->|"Yes (đã xác minh §8.1)"| z1["zalo.me/&lt;oa&gt;?...prefill=&lt;code&gt;<br/>tier=prefill"]:::ok
    zq -->|"No (chưa chắc — mặc định an toàn)"| z2["zalo.me/&lt;oa&gt; + ref nếu có<br/>tier=ref"]:::mid
    z2 -.->|"vẫn lỗi"| z3["zalo.me/&lt;oa&gt; chat trống<br/>tier=blank"]:::warn

    mm --> emit["bắn click_contact {channel, productId}"]:::ev
    z1 --> emit
    z2 --> emit
    z3 --> emit
    zp --> emit

    classDef ok fill:#e8f5e9,stroke:#2e7d32;
    classDef mid fill:#fff3e0,stroke:#ef6c00;
    classDef warn fill:#ffebee,stroke:#c62828;
    classDef ev fill:#e3f2fd,stroke:#1565c0;
```

> **Giảm cấp 3 tầng (ADR-002, US-B1 AC3):** `prefill → ref → blank`. **Messenger** đạt thẳng `tier=ref` vì `m.me?ref=` mang mã SP **chắc chắn** (Meta documented) → đây là lý do chọn Messenger-first cho P2. **Zalo prefill** còn là ẩn số (PRD §8.1 BLOCKING) nên resolver mặc định hạ về `ref`/`blank` an toàn cho tới khi tài liệu Zalo Developer được xác minh. Resolver luôn trả một URL hợp lệ — **không bao giờ dead-end**.

```ts
// components/funnel/ContactChannelResolver.ts — thuần hàm, test được, giấu kênh sau 1 interface
export function resolveContact(productCode: string | undefined, cfg: SiteConfig): ResolvedContact {
  switch (cfg.primaryContactChannel) {
    case "messenger":
      return { channel: "messenger", tier: "ref",
               url: `https://m.me/${cfg.messengerPage}${productCode ? `?ref=${encodeURIComponent(productCode)}` : ""}`,
               label: "Nhắn Messenger về sản phẩm này" };
    case "zalo_oa":
      return buildZaloOa(productCode, cfg);   // prefill→ref→blank theo cờ cfg.zaloPrefillVerified
    case "zalo_personal":
      return { channel: "zalo_personal", tier: "blank",
               url: `https://zalo.me/${cfg.zaloPersonalPhone}`, label: "Nhắn Zalo" };
  }
}
```

**AC → thiết kế:** US-B1 AC1 (một chạm mở kênh với ngữ cảnh SP) → `tier=ref` mang `productCode`; US-B1 AC3 (prefill Zalo chưa kiểm chứng) → nhánh `zalo_oa` giảm cấp + cờ `zaloPrefillVerified`; US-B2 AC1 (widget thường trực) → §6.

---

## 5. UC-08 — `ContactCTA`: build deep-link + `click_contact` (D6)

```mermaid
sequenceDiagram
    autonumber
    participant U as Khách (mobile/desktop)
    participant CTA as ContactCTA (client)
    participant R as resolveContact()
    participant Cfg as SiteConfig (prefetch ISR, props)
    participant AN as analytics dispatcher
    participant Ext as Messenger / Zalo

    Note over Cfg: cfg truyền từ server component (RSC) xuống props — không fetch runtime
    U->>CTA: chạm "Nhắn Messenger về SP này"
    CTA->>R: resolve(productCode)
    R->>Cfg: đọc primaryContactChannel + page/oa
    R-->>CTA: ResolvedContact { channel, url, tier }
    CTA->>AN: click_contact { channel, productId } (sau consent analytics)
    CTA->>Ext: window.open(url) — m.me?ref=code (app mobile / web desktop)
    Ext-->>U: Mở hội thoại, NV thấy ngữ cảnh SP (ref)
    Note over CTA,Ext: P2 — KHÔNG tạo lead ở đây; webhook auto-lead là P3/P4 (cần Meta app review)
```

**Chi tiết:**
- `SiteConfig` được **server component prefetch** (ISR, như catalog) và truyền xuống `ContactCTA` qua props → không có fetch runtime, không chặn INP, cache CDN được (nhánh UC-08 không PII).
- `click_contact` chỉ bắn **sau consent analytics** (ConsentGate analytics, HLD Flow 3) — không PII, chỉ `{channel, productId}`.
- Desktop vs mobile: `m.me` tự điều hướng app/web; không cần nhánh code riêng.

**AC → thiết kế:** US-B1 AC1 → `resolve()` + `window.open`; US-B1 AC2 (`click_zalo` + productID) → **`click_contact{channel,productId}`** (đã tổng quát hoá, ADR-002); US-B1 AC3 → §4 giảm cấp.

---

## 6. UC-09 — `ContactFloatingWidget`: widget chat nổi + CSP (D7)

```mermaid
flowchart TB
    layout["layout.tsx (RSC) — truyền siteConfig xuống"]:::rsc
    fw["ContactFloatingWidget ('use client', dynamic import)"]:::cli
    res["resolveContact() → chọn plugin theo channel"]:::fn
    mplug["Messenger Customer Chat plugin (mặc định)"]:::ext
    zplug["Zalo Social widget (bật sau — cùng flag)"]:::ext
    csp["middleware.ts — CSP allowlist origin kênh đang bật"]:::sec
    consent["Analytics/3rd-party gate: chỉ inject sau khi phù hợp"]:::cli

    layout --> fw --> res
    res -->|messenger| mplug
    res -.->|zalo_oa| zplug
    mplug & zplug -. origin phải nằm trong .-> csp
    fw --> consent

    classDef rsc fill:#e8f5e9,stroke:#2e7d32;
    classDef cli fill:#fff3e0,stroke:#ef6c00;
    classDef fn fill:#f3e5f5,stroke:#6a1b9a;
    classDef ext fill:#e1f5fe,stroke:#0277bd;
    classDef sec fill:#ffebee,stroke:#c62828;
```

**Quyết định kiến trúc:**
- **Dynamic import** (code-split) để script plugin bên thứ ba **không chặn LCP** của trang (NFR-1; như chatbot widget ở §3 customer-facing).
- Plugin kênh đổi **qua cùng flag** `primaryContactChannel` — một nguồn sự thật với UC-08.
- **CSP allowlist bắt buộc** (M-3): origin của plugin đang bật (`connect.facebook.net`, `*.messenger.com` cho Messenger; origin Zalo khi bật) phải nằm trong CSP; đổi kênh ⇒ cập nhật allowlist (checklist §12 customer-facing). Đây là điểm **cần đồng bộ CSP ↔ flag**: xem Rủi ro §12.
- Messenger Customer Chat plugin cần **Page ID**; lưu ở `SiteConfig` (publishable, không phải secret).

**AC → thiết kế:** US-B2 AC1 (widget chat nổi thường trực mọi trang) → `ContactFloatingWidget` mount trong `layout.tsx`.

---

## 7. UC-10 — `LeadForm` + `POST /api/lead` (Epic C, US-C1 + US-H2)

### 7.1 Cây component (D8)

```mermaid
flowchart TB
    page["/lien-he hoặc section LeadForm (vỏ SSG + form client)"]:::rsc
    form["LeadForm ('use client')"]:::cli
    fields["Fields: name · phone|zaloId · DOB · productCode · message"]:::cli
    notice["PrivacyNotice (DOB dùng cho tư vấn phong thủy — US-C1 AC3)"]:::cli
    gate["ConsentGate (checkbox minh thị, mặc định chưa tick, gắn policyVersion)"]:::consent
    client["leadClient.submit(LeadInput) — same-origin, CSRF"]:::fn
    api["POST /api/lead (TẦNG 2)"]:::api
    ok["ConfirmationState (xác nhận + submit_lead_form)"]:::cli
    fb["FallbackState (503) → ContactCTA (Messenger/Zalo)"]:::funnel

    page --> form --> fields & notice & gate
    form --> client --> api
    api -->|201 LeadAccepted| ok
    api -.->|503 store_unavailable| fb
    gate -. include .-> uc15["UC-15 ConsentRecord (server ghi)"]:::ucshared

    classDef rsc fill:#e8f5e9,stroke:#2e7d32;
    classDef cli fill:#fff3e0,stroke:#ef6c00;
    classDef consent fill:#e8f5e9,stroke:#2e7d32;
    classDef fn fill:#f3e5f5,stroke:#6a1b9a;
    classDef api fill:#e3f2fd,stroke:#1565c0;
    classDef funnel fill:#e8f5e9,stroke:#2e7d32;
    classDef ucshared fill:#e8f5e9,stroke:#2e7d32;
```

### 7.2 Sequence submit — consent+lead nguyên tử, 503 fallback (D9)

```mermaid
sequenceDiagram
    autonumber
    participant U as Khách
    participant LF as LeadForm (client)
    participant Z as Zod LeadInput (client)
    participant API as /api/lead (BFF, no-store)
    participant BE as omnigem-api /pii/lead
    participant TX as ConsentService + LeadService (1 transaction)
    participant AN as analytics

    U->>LF: điền + tick consent
    LF->>Z: validate (bắt buộc, regex SĐT, ít nhất phone|zalo)
    alt invalid
      Z-->>LF: lỗi field → báo tại chỗ (ARIA announce)
    else valid
      LF->>API: POST LeadInput (TLS, same-origin, CSRF header)
      API->>API: assertSameOrigin → rateLimit → Zod (consent.granted===true?)
      alt consent.granted !== true
        API-->>LF: 400 — KHÔNG ghi gì (UC-10 ngoại lệ)
      else ok
        API->>BE: forward (server-to-server, token PII)
        BE->>TX: BEGIN
        TX->>TX: ConsentService.record(policyVersion, purpose, method=lead_form)
        TX->>TX: LeadService.create(source="lead_form", PII enc, consent_record_id)
        alt commit OK
          TX-->>BE: COMMIT → { leadId }
          BE-->>API: 201
          API-->>LF: 201 LeadAccepted { leadId }
          LF->>AN: submit_lead_form (no PII)
          LF-->>U: ConfirmationState
        else DB lỗi
          TX-->>BE: ROLLBACK (no partial commit)
          BE-->>API: 503 store_unavailable
          API-->>LF: 503 { fallbackContactUrl }
          LF-->>U: FallbackState → CTA Messenger/Zalo (UC-10.4)
        end
      end
    end
```

> **Bất biến chốt (HLD Flow 1):** ConsentRecord + Lead ghi trong **cùng 1 transaction** → không bao giờ partial. `consent.granted!==true` ⇒ từ chối **trước khi** chạm DB (Zod `literal(true)`). DB lỗi ⇒ `503 + fallbackContactUrl` (resolver cấp), **không** commit một nửa → UC-10.4 "không ngõ cụt".

### 7.3 State machine của form (D10)

```mermaid
stateDiagram-v2
    [*] --> Editing: tải form (consent chưa tick)
    Editing --> Validating: submit
    Validating --> Editing: lỗi field (báo tại chỗ)
    Validating --> Blocked: consent chưa tick
    Blocked --> Editing: tick consent
    Validating --> Submitting: hợp lệ + consent=true
    Submitting --> Confirmed: 201 LeadAccepted
    Submitting --> RateLimited: 429 (thử lại sau)
    Submitting --> FallbackFunnel: 503 (CTA Messenger/Zalo)
    RateLimited --> Editing: sau cooldown
    FallbackFunnel --> [*]
    Confirmed --> [*]

    note right of Blocked
      UC-10 ngoại lệ: consent≠true
      → từ chối, KHÔNG ghi gì
    end note
    note right of FallbackFunnel
      UC-10.4: CRM lỗi → không dead-end,
      đổ về kênh chat (resolver)
    end note
```

**Hợp đồng lỗi (bám api-spec, §8.3 customer-facing):** `400` field/consent; `403` origin/CSRF; `429` rate-limit; `503 store_unavailable {fallbackContactUrl}`. Mọi trạng thái lỗi có **lối ra phễu** (D13).

**AC → thiết kế:** US-C1 AC1 (validate trường/định dạng trước khi nhận) → Zod client+server; US-C1 AC2 (ghi CRM built-in + xác nhận) → D9 bước commit + `ConfirmationState`; US-C1 AC3 (privacy notice cho DOB) → `PrivacyNotice`; US-C1 AC4 (fixture chỉ synthetic) → §13 test; US-H2 → §8.

---

## 8. UC-15 — Consent (Art.11) & ConsentGate (D11, D12)

### 8.1 Hai loại consent (D11)

```mermaid
flowchart TB
    subgraph CG["ConsentProvider (React context — trạng thái phiên)"]
      pii["Consent PII (Art.11, US-H2)<br/>bắt buộc TRƯỚC mọi submit PII<br/>checkbox minh thị, mặc định chưa tick<br/>gắn policyVersion hiện hành"]:::pii
      an["Consent Analytics (riêng)<br/>GA4/Pixel chỉ load SAU khi đồng ý<br/>AnalyticsLoader không inject tới khi có"]:::an
    end
    lf["LeadForm (UC-10) → cần Consent PII"]:::uc
    cta["ContactCTA/Widget (UC-08/09) → KHÔNG thu PII, không cần Consent PII"]:::uc
    ev["click_contact / submit_lead_form → cần Consent Analytics"]:::uc
    server["Bằng chứng tái lập được = SERVER ghi ConsentRecord nguyên tử<br/>(client chỉ KHẲNG ĐỊNH, không tự coi là đã đồng ý)"]:::srv

    pii --> lf
    an --> ev
    cta -. không chạm .- pii
    pii --> server

    classDef pii fill:#e8f5e9,stroke:#2e7d32;
    classDef an fill:#e3f2fd,stroke:#1565c0;
    classDef uc fill:#f3e5f5,stroke:#6a1b9a;
    classDef srv fill:#eceff1,stroke:#455a64;
```

> **Ranh giới quan trọng:** UC-08/09 (mở kênh chat) **không thu PII trên web** nên **không cần Consent PII** — đây là lý do Epic B rẻ và cache được. Chỉ UC-10 (và chatbot UC-11/13) mới «include» UC-15. **Consent Analytics** là cổng riêng, độc lập, gate cho cả `click_contact` lẫn `submit_lead_form`.

### 8.2 Ghi nguyên tử ConsentRecord + Lead (D12)

```mermaid
flowchart LR
    recv["/pii/lead nhận payload (đã qua origin/rate/Zod)"]:::api
    begin["BEGIN TX"]:::tx
    c["INSERT consent_record<br/>(policy_version, purpose, method, granted=true, ctx_hash, granted_at)"]:::tx
    l["INSERT lead<br/>(PII enc, source=lead_form, consent_record_id = ↑)"]:::tx
    commit["COMMIT → leadId"]:::ok
    rb["ROLLBACK → 503 store_unavailable"]:::err

    recv --> begin --> c --> l --> commit
    c -.->|lỗi bất kỳ| rb
    l -.->|lỗi bất kỳ| rb

    classDef api fill:#e3f2fd,stroke:#1565c0;
    classDef tx fill:#f3e5f5,stroke:#6a1b9a;
    classDef ok fill:#e8f5e9,stroke:#2e7d32;
    classDef err fill:#ffebee,stroke:#c62828;
```

> **Tái lập được (US-H2 AC1/AC2):** ConsentRecord ghim **đúng version Privacy Policy đã hiển thị** (`policyVersion` khách gửi phải khớp version hiện hành server — nếu lệch do publish giữa chừng, server dùng version thực nó hiển thị và có thể từ chối/nhắc tải lại). Bản ghi này export được làm bằng chứng Art.11 (UC-20.5, Boundary 2). `request_ctx_hash` thay cho lưu IP thô (tối thiểu hoá dữ liệu).

**AC → thiết kế:** US-H2 AC1 (bản ghi tái lập: điều gì + version + timestamp + method) → cột `CONSENT_RECORD`; AC2 (truy xuất làm bằng chứng) → export ở UC-20.5 (contract Boundary 2); AC3 (Art.11 áp cho MỌI PII gồm DOB) → `LeadInput.consent` bắt buộc khi có `dobISO`.

---

## 9. "Không ngõ cụt" — cây quyết định phễu Epic B/C (D13)

```mermaid
flowchart TD
    any["Khách muốn liên hệ"] --> q{Bối cảnh / trạng thái?}
    q -->|Quan tâm 1 SP| c1["ContactCTA(code) → resolver → Messenger ref=code (UC-08)"]
    q -->|Muốn chat bất kỳ lúc nào| c2["ContactFloatingWidget nổi mọi trang (UC-09)"]
    q -->|Chưa muốn chat, để lại info| c3["LeadForm → /api/lead (UC-10)"]
    c3 --> q2{Kết quả submit}
    q2 -->|201| done1["Xác nhận + submit_lead_form → NV follow up"]:::ok
    q2 -->|consent chưa tick| blk["Chặn tại UI, nhắc tick (không ghi gì)"]:::warn
    q2 -->|503 CRM lỗi| c4["FallbackState → ContactCTA Messenger/Zalo (UC-10.4)"]:::ok
    q2 -->|429| rl["Nhắc thử lại sau"]:::warn
    c1 & c2 & c4 --> enter["Vào kênh chat → lead (chăm sóc qua người)"]:::ok

    classDef ok fill:#e8f5e9,stroke:#2e7d32;
    classDef warn fill:#fff3e0,stroke:#ef6c00;
```

Mọi nhánh — kể cả lỗi CRM (503) và rate-limit — đều có **lối ra phễu**, nhất quán triết lý "không ngõ cụt" của Epic A (D17) và §14 customer-facing.

---

## 10. Analytics phễu (Epic G, UC-21; D14)

```mermaid
flowchart LR
    subgraph GATE["Consent Analytics gate (load tag SAU khi đồng ý)"]
      ga["GA4"]:::seo
      px["Meta Pixel"]:::seo
    end
    e1["click_contact { channel, productId }"]:::ev
    e2["submit_lead_form"]:::ev
    disp["lib/analytics/events.ts (chuẩn hoá, KHÔNG PII)"]:::fn

    cta["UC-08 ContactCTA"] --> e1
    fw["UC-09 Floating widget"] --> e1
    lf["UC-10 LeadForm 201"] --> e2
    e1 & e2 --> disp --> ga & px

    classDef seo fill:#e3f2fd,stroke:#1565c0;
    classDef ev fill:#ede7f6,stroke:#6a1b9a;
    classDef fn fill:#f3e5f5,stroke:#6a1b9a;
```

| Event | Nguồn | Khi | Payload (no PII) |
|---|---|---|---|
| `click_contact` | UC-08, UC-09 | Chạm CTA/widget chat | `{ channel: "messenger"\|"zalo_oa", productId? }` |
| `submit_lead_form` | UC-10 | Nhận `201 LeadAccepted` | `{}` (chỉ tên event) |

> **ADR-002:** `click_zalo` (PRD US-B1 AC2, US-G1) **tổng quát hoá → `click_contact{channel}`** để đo đúng kênh đang chạy (Messenger/Zalo) và phục vụ A-B test kênh (P4). **Không PII** (không name/phone/dob) — chỉ `productId` + `channel`. Tag chỉ load **sau consent analytics** (HLD Flow 3).

**AC → thiết kế:** US-B1 AC2 → `click_contact{channel,productId}`; US-G1 (`submit_lead_form` là event riêng có tên) → §10.

---

## 11. Cấu trúc thư mục lát cắt Epic B/C (D15)

```mermaid
flowchart TB
    subgraph APP["app/"]
      a1["(marketing)/lien-he/page.tsx — vỏ SSG + LeadForm (UC-10)"]
      a2["api/lead/route.ts — TẦNG 2 BFF (runtime=nodejs, no-store)"]
    end
    subgraph COMP["components/funnel/"]
      c1["ContactCTA.tsx (UC-08)"]
      c2["ContactFloatingWidget.tsx (UC-09)"]
      c3["ContactChannelResolver.ts (thuần hàm, ADR-002)"]
      c4["LeadForm.tsx + ConfirmationState + FallbackState (UC-10)"]
      c5["FunnelCTABlock.tsx (tái dùng blog/empty-state)"]
    end
    subgraph CONSENT["components/consent/"]
      g1["ConsentGate.tsx · ConsentProvider.tsx · PrivacyNotice.tsx (UC-15)"]
    end
    subgraph LIB["lib/"]
      l1["schemas/lead.ts (Zod dùng chung)"]
      l2["api-client/leadClient.ts (gọi /api/lead)"]
      l3["security/origin.ts · rateLimit.ts (middleware BFF)"]
      l4["analytics/events.ts (click_contact, submit_lead_form)"]
      l5["content/siteConfig.ts (đọc SiteConfig qua ContentRepository/ISR)"]
    end
    APP --> COMP --> CONSENT --> LIB
```

> Bám cấu trúc §5 customer-facing (`components/funnel/`, `components/consent/`, `lib/schemas`, `lib/security`). `ContactChannelResolver` là **thuần hàm** (dễ unit test mọi nhánh kênh/tier). `SiteConfig` đọc qua cùng `ContentRepository` của Epic A (prefetch ISR, truyền props).

---

## 12. Hợp đồng props & route (tóm tắt)

```ts
// components/funnel/ContactCTA.tsx ("use client")
interface ContactCTAProps {
  productCode?: string;            // UC-08: ref mang mã SP (Messenger tier=ref)
  siteConfig: SiteConfig;          // prefetch từ RSC (channel, page/oa) — không fetch runtime
  variant?: "bar" | "inline";      // CtaBar above-the-fold (US-A3 AC2) tái dùng
}

// components/funnel/ContactFloatingWidget.tsx ("use client", dynamic import)
interface ContactFloatingWidgetProps {
  siteConfig: SiteConfig;          // UC-09: chọn plugin Messenger/Zalo theo flag
}

// components/funnel/LeadForm.tsx ("use client")
interface LeadFormProps {
  defaultProductCode?: string;     // prefill khi mở từ Product Detail
  policyVersion: string;           // version hiện hành ghim vào consent (US-H2)
}

// app/api/lead/route.ts
// POST LeadInput → 201 LeadAccepted{leadId} | 400 | 403 | 429 | 503 StoreUnavailable{fallbackContactUrl}
// runtime = "nodejs"; headers: Cache-Control: no-store
```

---

## 13. Chiến lược test (lát cắt Epic B/C)

| Lớp | Phạm vi |
|---|---|
| **Unit** | `resolveContact()` — phủ mọi nhánh channel × tier (messenger/zalo_oa prefill/ref/blank/zalo_personal); Zod `LeadInput` (consent `literal(true)` reject; `.refine` phone\|zalo); `events.ts` không chứa PII. |
| **Integration** | `POST /api/lead`: (a) ghi **consent+lead nguyên tử**, (b) **reject khi consent≠true không ghi gì**, (c) **503 → rollback + fallbackContactUrl** (no partial commit), (d) origin/CSRF 403, (e) rate-limit 429. |
| **E2E (Playwright)** | Product → ContactCTA mở Messenger (assert URL `m.me?ref=code`); LeadForm submit 201 → confirmation; **consent gate chặn khi chưa tick**; 503 → fallback CTA hiển thị. |
| **PII convention** | **Chỉ dữ liệu synthetic** trong fixture/doc (US-C1 AC4) — không SĐT/DOB thật. |

---

## 14. Ma trận truy vết AC → thiết kế/sơ đồ

| US / AC | Thiết kế | Sơ đồ |
|---|---|---|
| US-B1 AC1 deep-link mang ngữ cảnh SP | `ContactCTA` + `resolveContact` tier=ref | D5, D6 |
| US-B1 AC2 event + productID | `click_contact{channel,productId}` (tổng quát `click_zalo`) | D6, D14 |
| US-B1 AC3 prefill Zalo chưa kiểm chứng | nhánh `zalo_oa` giảm cấp prefill→ref→blank | D5 |
| US-B2 AC1 widget chat nổi mọi trang | `ContactFloatingWidget` trong `layout.tsx` | D7 |
| US-C1 AC1 validate trường/định dạng | Zod `LeadInput` client+server (regex SĐT, phone\|zalo) | D4, D9, D10 |
| US-C1 AC2 ghi CRM + xác nhận | TX commit + `ConfirmationState` | D9, D12 |
| US-C1 AC3 privacy notice cho DOB | `PrivacyNotice` + `ConsentGate` | D8, D11 |
| US-C1 AC4 fixture synthetic | quy ước test | §13 |
| US-H2 AC1 bản ghi consent tái lập | `CONSENT_RECORD` (policy_version, method, timestamp) | D3, D12 |
| US-H2 AC2 truy xuất bằng chứng | export ở UC-20.5 (contract Boundary 2) | D12 |
| US-H2 AC3 Art.11 áp cho mọi PII (gồm DOB) | consent bắt buộc trong `LeadInput` | D4, D11 |
| US-G1 submit_lead_form là event riêng | `events.ts` | D14 |
| UC-10.4 CRM lỗi → fallback | 503 + `fallbackContactUrl` (resolver) | D9, D10, D13 |

---

## 15. Điểm cần chốt trước khi code Epic B/C

| # | Điểm | Trạng thái |
|---|---|---|
| 1 | **Cú pháp prefill deep-link Zalo** | **BLOCKING nhánh `zalo_oa`** (PRD §8.1, US-B1 AC3). P2 chạy **Messenger-first** nên **không chặn P2**; resolver giữ cờ `zaloPrefillVerified=false` + giảm cấp cho tới khi xác minh tài liệu Zalo Developer. |
| 2 | **Kênh handoff** | **CHỐT (ADR-002):** Messenger-first (`m.me?ref=code`) qua `ContactChannelResolver` + flag `SiteConfig.primaryContactChannel`; Zalo OA bật sau khi có doanh thu bằng đổi flag. |
| 3 | **Webhook auto-tạo lead từ Messenger/Zalo** | **Hoãn P3/P4** (cần Meta app review `pages_messaging`). P2: lead chỉ qua LeadForm (UC-10) + chatbot handoff (UC-13). |
| 4 | **ConsentRecord + Lead nguyên tử** | **CHỐT (HLD Flow 1):** cùng 1 transaction, `consent_record_id` NOT NULL, no partial commit; thực thi trong `omnigem-api` (ADR-001). |
| 5 | **CSP ↔ flag kênh** | Mở: khi đổi `primaryContactChannel`, phải cập nhật CSP allowlist origin plugin (Messenger ↔ Zalo). Đề xuất: sinh CSP từ `SiteConfig` ở `middleware.ts` để tránh lệch thủ công — chốt ở LLD. |
| 6 | **Nội dung privacy notice cho DOB** | Chờ compliance xác nhận câu chữ (PRD §8.2) — ảnh hưởng `PrivacyNotice`, không ảnh hưởng cấu trúc. |
| 7 | **Base URL + schema PII endpoint `omnigem-api`** | Topology CHỐT (ADR-001); schema `/pii/lead` cụ thể chốt ở Design; `leadClient`/BFF giữ hợp đồng ổn định + mock để không chặn P2. |

**Mô hình vận hành đã xác nhận (ADR-002):** thu lead (LeadForm P2 / chatbot P3 / webhook P3-P4) → chăm sóc + chốt qua người (Messenger) → ship / kéo về cửa hàng → *(tương lai)* bán online. NG1 (không checkout) giữ nguyên.
