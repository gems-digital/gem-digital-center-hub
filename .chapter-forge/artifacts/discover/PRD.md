# OmniGem Gallery — BRD/PRD (P1 Discovery & Requirements)

Status: DRAFT for Gate G0. Author: requirements-analyst. Approvers: Product Owner + Risk/Compliance.
Sources: `documents/omnigem-gallery-research.md` (all sections), `documents/create-facebook-fanpage.md` (§1–3).

---

## 1. Executive Summary & Business Case

OmniGem Gallery is a promotional website for a Vietnamese natural jade & gemstone jewelry brand. It is explicitly **not** an e-commerce checkout system. High-value pieces (20–200M VND) are never purchased on the web; the site's job is to build trust and emotional connection, then route the visitor into a human-assisted closing conversation on Zalo (research doc §1, §5).

**Business case (per research doc §1, §7):**
- High price points create purchase friction on a self-serve cart; a "click to buy" flow would suppress conversion, not help it.
- The market (Vietnam) closes high-trust, high-value purchases through personal chat (Zalo) more reliably than web checkout.
- Web's job = (a) prove authenticity ("real jade"), (b) prove seller credibility, (c) offer feng-shui/ngũ-hành matching, then (d) hand off to a human.
- SEO content on jade authenticity/feng-shui is a durable, near-zero-marginal-cost traffic channel (research doc §6b).

**Thesis:** OmniGem Gallery = digital showroom + trust engine + funnel into Zalo/Chatbot, not a marketplace.

## 2. Goals & Non-Goals

### Goals
- G1: Present the product catalog (Collections + Product Detail) with high-fidelity imagery/video that substantiates authenticity claims.
- G2: Offer feng-shui/ngũ-hành (five-element) based navigation/filtering to match visitors to products by birth element.
- G3: Convert visitor interest into a qualified lead via Zalo deep-link CTA, a contact/lead form, and an AI chatbot with mandatory human handoff.
- G4: Rank organically for jade-authenticity and feng-shui-jewelry search queries (SEO).
- G5: Instrument the full funnel (page view → chatbot engagement → Zalo click / lead form submit) in analytics.
- G6: Let non-technical shop staff manage catalog/blog content via a headless CMS.

### Non-Goals (explicit)
- NG1: **No on-site checkout, cart, or payment processing of any kind.** No card data, no card-present/card-not-present flow, no PCI scope on the website in this phase.
- NG2: No inventory/order-management system; fulfillment and payment happen entirely off-platform via Zalo (human-negotiated, COD/deposit per research doc §6c).
- NG3: No user account/login system for buyers (not specified as needed by the research; flagged as open question, see §8).
- NG4: No multi-currency/international shipping in this phase (Vietnam-market focus, per Zalo-centric funnel).

## 3. Target Users / Personas

1. **Feng-shui-matched buyer** — wants a bracelet/pendant that matches their ngũ hành (birth element); primary conversion path is the chatbot's DOB-based consultation (research doc §4, fanpage guide FAQ Q2).
2. **Gift buyer** — buying for a relative/partner; cares about presentation, certification, and warranty/return policy (fanpage guide FAQ Q3) more than personal feng-shui fit.
3. **Collector / high-value repeat buyer** — evaluates sculptures/one-of-a-kind pieces; needs deep product detail (origin, certification, high-res zoom) and is comfortable being routed straight to Zalo for negotiation (research doc §2 "Chi tiết tác phẩm").

*(Personas are inferred from the research doc's IA and marketing sections, not from primary user research. Flagged as an assumption — see §8.)*

## 4. Scope by Phase (aligned to research doc §7 roadmap)

| Phase | Scope | Primary output |
|---|---|---|
| P0 — Foundation | Next.js + CMS + design system scaffold, sitemap, Vercel deploy | Running skeleton site |
| P1 — Gallery | Home, Collections, Product Detail, optimized imagery/video, base SEO | Complete showroom |
| P2 — Navigation/Funnel | Zalo widget + per-product deep-link, lead form, GA4/Meta Pixel | Lead generation live |
| P3 — AI Chatbot | Chat widget + Claude API + RAG over catalog/FAQ + human handoff | 24/7 automated consultation |
| P4 — Growth | SEO blog content, remarketing, CTA A/B tests, Zalo ZNS care messages | Traffic & conversion growth |

This PRD's functional requirements below cover P1–P3 (the phases with net-new functional surface); P0 is infra setup and P4 is optimization work to be re-scoped later.

## 5. Functional Requirements — User Stories & Acceptance Criteria

### Epic A — Gallery Pages

**US-A1: Home page**
As a visitor, I want a cinematic home page highlighting featured pieces, so I trust the brand and want to explore.
- AC1: Given a visitor lands on `/`, when the page loads, then a hero section and a curated set of featured products render within Core Web Vitals budget (see NFR-1).
- AC2: Given the visitor scrolls, when they reach the featured section, then each product card links to its Product Detail page.

**US-A2: Collections browsing with feng-shui filter**
As a visitor, I want to browse by category (bangles, pendants, sculptures) and filter by ngũ hành/mệnh, so I find pieces that match my element.
- AC1: Given the Collections page, when a visitor selects a category, then only products in that category display.
- AC2: Given the Collections page, when a visitor applies a "Mệnh" (element) filter — e.g., Kim/Mộc/Thủy/Hỏa/Thổ — then only products tagged with a matching element display.
- AC3: Given no products match the combined filter, when the filter is applied, then an empty state with a "Ask the chatbot" or "Message Zalo" CTA displays (funnel fallback, not a dead end).

**US-A3: Product Detail page**
As a visitor, I want full detail on a piece (imagery, specs, provenance, certification, feng-shui fit, CTAs), so I can decide to pursue it.
- AC1: Given a Product Detail page, when it loads, then it displays: high-resolution image/video gallery (zoomable), product name/code, jade type, dimensions, origin, certification reference, price OR "Contact for consultation" (per pricing model in §6a), and a feng-shui fit block.
- AC2: Given the page, when rendered, then two primary CTAs are visible above the fold on mobile: "Message Zalo about this piece" (deep-linked, see US-B1) and "Ask the AI about this piece" (opens chatbot pre-seeded with product context).
- AC3: Given the product has customer feedback/proof media, when available, then it renders as a social-proof section.

**US-A4: Brand story / trust page**
As a visitor, I want to see provenance and certification claims, so I trust the products are genuine.
- AC1: Given `/brand-story` (or equivalent), when loaded, then it presents sourcing narrative, lab-certification explanation, and links to the Certification showcase page — reusing trust language from `documents/create-facebook-fanpage.md` §2 FAQ Q1 ("100% natural jade... certificate from a reputable gemology lab").

### Epic B — Zalo Funnel

**US-B1: Per-product Zalo deep-link CTA**
As a visitor interested in one piece, I want a single tap to open Zalo with the product context pre-filled, so staff know exactly what I'm asking about.
- AC1: Given a Product Detail page, when the visitor taps "Message Zalo about this piece," then Zalo opens (app on mobile, Zalo Web on desktop) with a prefilled message containing the product code — mechanism per Zalo's official deep-link/prefill docs (research doc §5.3 flags this as needing verification against current Zalo Developer docs before build).
- AC2: Given the tap event, when it fires, then a `click_zalo` analytics event is sent with product ID (research doc §5.5).
- AC3: **Open question**: exact Zalo prefill parameter syntax is unverified in the research doc — must be confirmed against live Zalo Developer docs before implementation (see §8).

**US-B2: General Zalo widget**
As any visitor, I want a persistent Zalo chat entry point, so I can reach a human anytime.
- AC1: Given any page, when it renders, then a floating Zalo chat widget (official Zalo script) is present, per research doc §5.2.

### Epic C — Lead Capture

**US-C1: Contact/consultation form**
As a visitor not ready to use Zalo/chat, I want to leave my contact info and interest, so staff can follow up.
- AC1: Given the Contact/Consultation page, when the visitor submits name, phone or Zalo ID, **date of birth**, and product interest, then the submission is validated (required fields, phone format) before accepting.
- AC2: Given a valid submission, when saved, then it is written to the lead store (CRM/Google Sheet per research doc §4) and a confirmation message displays.
- AC3: Given the form collects DOB, when the form is presented, then a privacy notice/consent statement is shown explaining DOB is used for feng-shui consultation, and the consent capture satisfies Art. 11 (see Epic H, US-H2) — **contingent on compliance sign-off**, see `compliance-scope.md`.
- AC4: No real PII may appear in test fixtures or documentation; synthetic data only (per test-engineer/data-classification conventions).

### Epic H — Privacy Policy & Data Subject Rights (Decree 13/2023)

Compliance-checker confirmed (see `compliance-scope.md`) that this is a firm functional deliverable for Gate G0/G1, not a footnote — promoted to its own epic.

**US-H1: Published privacy policy page**
As a visitor, I want to read how my personal data is collected, used, and protected, so I can make an informed decision before submitting any data.
- AC1: Given any page that collects personal data (lead form, chatbot), when the visitor looks for it, then a linked, published Privacy Policy page is reachable from that page and from the site footer.
- AC2: Given the Privacy Policy page, when it loads, then it discloses: what data is collected (name, phone/Zalo ID, DOB, product interest, chat transcripts, analytics identifiers), why (feng-shui consultation, lead follow-up, funnel analytics), who processes it (OmniGem as controller; Vercel, Cloudinary, Anthropic/Claude API, GA4, Meta Pixel, Zalo as processors/recipients — see `compliance-scope.md` Obligation B), and that some processing/storage occurs outside Vietnam.

**US-H2: Express, reproducible consent capture (Art. 11)**
As the business, I need a legally sufficient consent record, so PDPD Art. 11 obligations are met — not just a UI checkbox with no retrievable evidence.
- AC1: Given the lead form or chatbot DOB request, when a visitor submits personal data, then the system captures and durably stores a **reproducible consent record** (what was consented to, version of the privacy notice shown, timestamp) — not merely an unlogged checkbox state.
- AC2: Given a consent record exists, when requested (e.g., by the visitor or an auditor), then it can be retrieved and shown as evidence of that specific consent event.
- AC3: Given Art. 11 applies to ALL personal data with no legitimate-interest carve-out, when any personal data field is collected (including DOB, which is basic — not sensitive — personal data per `data-classification.md` item 7), then consent capture per AC1 still applies to it.

**US-H3: Data subject rights mechanism (Art. 9/10)**
As a visitor whose data OmniGem holds, I want to be informed, access my data, withdraw consent, request deletion, restrict, or object to processing, so my PDPD rights are honored.
- AC1: Given the Privacy Policy page, when a visitor wants to exercise a right, then it names a concrete channel (e.g., an email address or Zalo OA request) for: right to be informed, right of access, right to withdraw consent, right to erasure/deletion, right to restrict processing, and right to object.
- AC2: Given a data-subject-rights request arrives through that channel, when received, then it is logged and routed to an accountable owner (Product Owner/Compliance — placeholder pending a named role) with a defined internal SLA (exact SLA TBD, open question for Product Owner).
- AC3: Given a visitor withdraws consent or requests deletion, when actioned, then their record is removed/anonymized from the lead store and, where technically feasible, flagged for removal from chatbot transcript storage — subject to any legal retention obligation that overrides (e.g., an active DPIA/audit record).

### Epic D — AI Chatbot

**US-D1: Feng-shui consultation via chatbot**
As a visitor, I want to ask the chatbot for a stone/color recommendation based on my birth date, so I get a personalized suggestion.
- AC1: Given the chat widget, when a visitor provides a date of birth, then the bot returns an element (ngũ hành) mapping and recommends matching product categories (research doc §4.1).
- AC2: Given the bot recommends products, when it responds, then it uses the `search_products(mệnh, loại, giá)` tool against live CMS data — not hallucinated inventory (research doc §4, RAG requirement).

**US-D2: Trust FAQ via chatbot**
As a visitor, I want the chatbot to answer authenticity/certification/return-policy questions, so I don't need to wait for a human for basic questions.
- AC1: Given a visitor asks an FAQ-type question (e.g., "is this real jade," "what's your return policy"), when the bot responds, then it uses the same commitments as `create-facebook-fanpage.md` §2 (7-day defect exchange, lifetime string/polish service, lab certification) — content must stay consistent across Fanpage, chatbot, and web copy.

**US-D3: Mandatory human handoff**
As a visitor showing purchase intent, I want to be connected to a real person, so I can actually close a high-value purchase.
- AC1: Given the visitor expresses interest in a specific product or asks to buy, when detected, then the bot invokes `handoff_to_zalo()`, generates a Zalo link, and stores the lead (name, need, mệnh, product interest) — research doc §4 "Human handoff bắt buộc."
- AC2: Given a handoff occurs, when the lead is stored, then it lands in the same lead store as US-C1 (single source of truth for leads), tagged with source = "chatbot."
- AC3: Given the bot cannot confidently answer (e.g., price negotiation, custom orders), when detected, then it defaults to offering handoff rather than guessing.

### Epic E — SEO / Discoverability

**US-E1: Structured data & metadata**
As the business, I want product and organization schema markup, so search engines produce rich results.
- AC1: Given any Product Detail page, when crawled, then `Product` schema.org JSON-LD is present with name, image, description (research doc §6b).
- AC2: Given the site, when crawled, then `Organization` schema is present site-wide.

**US-E2: SEO content (blog)**
As a prospective buyer researching jade, I want educational content (e.g., "how to tell real jade"), so I find OmniGem via organic search.
- AC1: Given the CMS, when a content editor publishes a blog post, then it is reachable at a stable URL, indexable, and internally links to relevant collections/products.

### Epic F — Content Management

**US-F1: Non-technical catalog management**
As shop staff, I want to add/edit products and prices without a developer, so the catalog stays current.
- AC1: Given the CMS admin, when staff create/edit a product entry (images, specs, price or "contact for price," feng-shui tags), then the change reflects on the live site within the CMS's revalidation window (ISR — research doc §3) without a full redeploy.

### Epic G — Analytics

**US-G1: Funnel instrumentation**
As the business, I want to measure the funnel from page view to lead, so I can optimize spend and content.
- AC1: Given GA4 + Meta Pixel are installed, when a visitor views a product, clicks Zalo, submits the lead form, or gets a chatbot handoff, then each is fired as a distinct, named event (`view_product`, `click_zalo`, `submit_lead_form`, `chatbot_handoff`).

## 6. Non-Functional Requirements

- **NFR-1 Performance/Core Web Vitals:** Given jade imagery is large/high-res, pages must still meet "good" Core Web Vitals thresholds (LCP, INP, CLS) on mobile 4G — target values TBD with Product Owner/Architect (open question, §8). Requires CDN-served, lazy-loaded, responsive images (Cloudinary/`next/image`, per research doc §3).
- **NFR-2 Mobile-first:** Majority of VN traffic is mobile (research doc §6b) — all flows (filter, CTA, chatbot, form) must be fully usable at mobile viewport widths, CTAs above the fold.
- **NFR-3 SEO:** SSG/ISR rendering so crawlers receive full HTML (research doc §3); target rendering strategy confirmed at design phase.
- **NFR-4 Availability:** Marketing site — target uptime TBD (proposed 99.5% as a starting point for Product Owner confirmation; not a transactional system so lower than banking-grade is acceptable).
- **NFR-5 Accessibility:** No explicit target in research doc — proposed WCAG 2.1 AA baseline for form/CTA elements; open question for Product Owner (§8).
- **NFR-6 Cost:** Stack must stay within "quảng bá + SEO + tốc độ + chi phí thấp" (promotion + SEO + speed + low cost) intent (research doc §3) — Vercel/Cloudinary usage should be monitored against a budget ceiling (amount TBD, §8).
- **NFR-7 Data protection:** PII collected (name, phone, DOB, chat transcripts) must be handled per Vietnam's PDPD (Decree 13/2023/NĐ-CP) — see `data-classification.md` and `compliance-scope.md`.

## 7. Success Metrics / KPIs

- Lead volume: form submissions + chatbot handoffs per week/month.
- Zalo click-through rate: `click_zalo` events / product-page sessions.
- Chatbot handoff rate: conversations resulting in `handoff_to_zalo()` / total conversations.
- Organic traffic growth: sessions from organic search, month over month, plus keyword rankings for target jade/feng-shui terms.
- Core Web Vitals pass rate (field data) per NFR-1.
- (Downstream, not directly measurable on-site) Zalo-to-close rate — requires manual reporting from sales staff since closing happens off-platform.

## 8. Open Questions / Assumptions (must be resolved before/at Gate G0 or G1)

1. **Zalo deep-link prefill syntax** is explicitly flagged as unverified in the research doc (§5.3, §Nguồn tham khảo) — must be confirmed against current Zalo Developer docs before Design phase build-out.
2. **Consent/privacy notice content and mechanism for DOB collection** — research doc does not specify consent UX; this PRD assumes a checkbox/notice is required pending compliance-checker confirmation (PDPD).
3. **CMS choice (Sanity vs Payload)** — research doc offers both as options without a final decision; treated as an architecture-phase decision, not locked here.
4. **Chatbot AI vendor lock-in** — research doc proposes Claude API as the "self-hosted, higher-quality-control" option vs. Coze/Botpress/Manychat as a faster no-code path (§4). This PRD assumes Claude API per the primary recommendation; Product Owner should confirm budget/timeline tradeoff.
5. **Buyer accounts** — no requirement stated for login/accounts; assumed out of scope (NG3) — confirm with Product Owner.
6. **Pricing display policy** — research doc recommends a "hybrid" model (show price for mass-market items, "contact for price" for premium/unique items) but does not give a firm threshold (§6a) — needs a concrete VND cutoff from Product Owner/business.
7. **Lead store: Google Sheet vs. real CRM** — research doc mentions "CRM/Google Sheet" (§4) as if interchangeable; a Google Sheet has materially weaker access control — flagged as a risk (see `risk-register.md` R-03) and needs a decision before Design phase.
8. **Uptime/accessibility/budget targets** — no numeric targets given in the research doc; proposed placeholders above need Product Owner sign-off.
9. **Cross-border hosting** — Vercel, Cloudinary, and Claude API infrastructure may process/store Vietnamese personal data outside Vietnam; PDPD cross-border transfer rules need compliance-checker review (see `compliance-scope.md`).
10. **Personas (§3) are inferred, not sourced from user research** — flagged as an assumption; consider a lightweight validation pass (e.g., customer interviews) if time allows.

## 9. Dependencies

- Verified Zalo Official Account (OA) — required before deep-link/widget work (research doc §5.1).
- Claude API key/account (Anthropic) — required for chatbot (research doc §4).
- Headless CMS instance (Sanity or Payload — TBD, see Open Question 3).
- Cloudinary account (or equivalent CDN/image pipeline).
- Domain + Vercel project.
- GA4 property + Meta Pixel + Zalo conversion tracking setup.
- Brand assets (logo, cover imagery) — per `documents/create-facebook-fanpage.md` §3, reusable across Fanpage and website.
- Lead storage target (Google Sheet or CRM) — decision needed, see Open Question 7.

## 10. Proposed Gate G0 Checklist (per `sdlc-graph.yaml` G0 criteria)

- [ ] Business case approved — draft above, pending Product Owner sign-off.
- [ ] Risks ranked — see `risk-register.md`.
- [ ] Compliance scope (SBV/PCI/ISO) identified — see `compliance-scope.md` (proposed: PDPD yes, PCI/SBV out of scope, ISO optional) — pending compliance-checker confirmation.
- [ ] Data classified (any card data/PII?) — see `data-classification.md` (proposed: PII yes — name/phone/DOB/chat transcripts; card data — no).
- [ ] No legal blockers — pending compliance-checker review of PDPD consent/cross-border items (Open Questions 2, 9).
