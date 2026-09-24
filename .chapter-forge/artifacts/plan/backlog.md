# OmniGem Gallery — Refined Backlog (P3 Planning & Backlog)

Status: DRAFT for DoR convergence. Author: requirements-analyst. Consumed next by: test-engineer (test strategy, parallel), release-manager (release sketch, parallel), then a DoR convergence check.
Gates G0 and G1 are APPROVED — this backlog operationalizes the approved PRD epics (A–H) against the approved design (`hld-lld.md`, `api-spec.yaml`, `erd.md`, `class-diagram.md`, ADR-0001..0005).

Definition of Ready (DoR) applied to every story below:
1. Requirements clear & testable · 2. Data touched classified · 3. Security impact assessed · 4. Estimated · 5. `test_approach` hint present.

No real PII/PAN in this document — all examples synthetic, consistent with `api-spec.yaml`'s synthetic examples and the write-guard.

---

## 0. Blocking Product-Owner decisions (carried from PRD §8 / ADRs)

These four items are **not new work** — they are open decisions whose absence blocks specific stories from being marked DONE (not from being planned/estimated). Each is referenced by ID on the stories it blocks.

- **PO-1 — Chatbot cost ceiling (monthly Claude API spend cap, VND).** Blocks: US-D1, US-D2, US-D3, US-D-INFRA1 (SpendCircuitBreaker needs a concrete number; `nfr-targets.md §6` currently says "FIRM cap" with no figure).
- **PO-2 — Lead-store Postgres region + DPA.** Blocks: US-INFRA-LEAD1 (ADR-0002 recommends managed Postgres but region/vendor DPA undecided — affects the Art. 25 dossier).
- **PO-3 — DPIA (Art. 24) and cross-border TIA (Art. 25) named owners.** Blocks: US-H-COMP1, US-H-COMP2 (dossiers can be drafted, but "done" requires a named accountable owner per risk-register R-04/R-12).
- **PO-4 — Anthropic no-training / zero-retention DPA signed.** Blocks: US-D3 go-live (ADR-0003 "BLOCKING PREREQUISITE" — chatbot can be built and tested in a sandbox without it, but must not serve real traffic before it's signed).

Stories affected are marked **DEFERRED (go-live gate)** or carry an explicit blocking note in their Dependencies column — they are still estimated and DoR-compliant for *build*, just not closeable end-to-end without the PO decision.

---

## 1. Sprint slicing (maps to PRD §4 roadmap P0–P4)

| Sprint grouping | PRD phase | Theme | Stories |
|---|---|---|---|
| **Sprint 0 (infra, prerequisite)** | P0 — Foundation | Scaffold, CMS, lead store, consent plumbing | US-INFRA1..5 |
| **Sprint 1 (MVP — gallery + lead funnel)** | P1 + P2 (partial) | Showroom pages, Zalo funnel, lead form + consent, privacy policy | US-A1..A4, US-B1..B2, US-C1a/b, US-H1, US-H2, US-E1, US-F1, US-G1 |
| **Sprint 2 (chatbot)** | P3 | AI chatbot with guardrails, human handoff | US-D-INFRA1, US-D1, US-D2, US-D3a/b |
| **Sprint 3 (DSR automation + compliance dossiers)** | Cross-cutting (H) | Full DSR fulfilment workflow, DPIA/TIA dossiers | US-H3a/b/c, US-H-COMP1, US-H-COMP2 |
| **Sprint 4+ (growth, P4)** | P4 — Growth | SEO blog, remarketing, CTA A/B, ZNS care | US-E2, and future stories (out of this backlog's detailed scope — noted, not elaborated) |

**MVP definition (Sprint 1 exit):** a visitor can browse the gallery, filter by mệnh, view a product detail page, and convert via Zalo deep-link **or** a consent-gated lead form that writes an atomic Lead+ConsentRecord to a real (non-Sheet) store — with a published, versioned Privacy Policy backing the consent. Chatbot and DSR self-service are explicitly **not** in MVP.

---

## 2. Dependency map (critical path)

```
US-INFRA1 (Next.js/Sanity scaffold, Vercel project)
   └─→ US-INFRA2 (Postgres lead store provisioned — BLOCKED on PO-2 for prod, sandbox OK for dev)
          └─→ US-INFRA3 (ConsentRecord + PrivacyPolicyVersion schema, DB-enforced append-only — M-6)
                 └─→ US-H1 (Privacy Policy page, versioned, published)
                        └─→ US-H2 (Consent gate component + ConsentService)
                               └─→ US-C1a (Lead form UI) → US-C1b (POST /api/lead, atomic tx)
                                      └─→ US-G1 (submit_lead_form analytics event)
   └─→ US-INFRA4 (CmsAdapter + catalog schema in Sanity)
          └─→ US-A1/A2/A3/A4 (gallery pages) [parallel with US-INFRA2/3 chain]
                 └─→ US-B1 (Zalo deep-link CTA) — also needs US-INFRA5 (verified Zalo OA — EXTERNAL dependency)
   └─→ US-B2 (Zalo widget) [parallel, only needs US-INFRA5]
   └─→ US-F1 (CMS non-technical editing) [parallel with US-A*, same CmsAdapter]
   └─→ US-E1 (Product/Org JSON-LD) [depends on US-A3 markup existing]

--- Sprint 2 (chatbot), after Sprint 1's consent + lead infra exists ---
US-D-INFRA1 (SessionAuthority + bot-challenge + SpendCircuitBreaker + SensitiveInputScrubber)
   └─→ US-D1 (feng-shui consult) + US-D2 (FAQ) [parallel, share ChatbotService]
          └─→ US-D3a (handoff_to_zalo tool + lead write) → US-D3b (go-live gate: PO-1 spend cap + PO-4 Anthropic DPA)

--- Sprint 3 (DSR + dossiers), can start in parallel with Sprint 2 ---
US-H3a (DSR intake API) → US-H3b (identity verification OTP, H-4) → US-H3c (fulfilment + maker-checker)
US-H-COMP1 (DPIA dossier) + US-H-COMP2 (cross-border TIA dossier) — BLOCKED on PO-3 (owner)
```

**Critical path (one line):** `US-INFRA1 → US-INFRA2 (Postgres, PO-2) → US-INFRA3 (ConsentRecord schema) → US-H1 (Privacy Policy) → US-H2 (consent gate) → US-C1a/b (lead form, atomic write) → US-G1 (analytics)` — this chain gates the entire MVP lead-funnel; the chatbot (Sprint 2) and DSR automation (Sprint 3) both build on the same ConsentService/LeadRepository this chain establishes.

---

## 3. Backlog — Sprint 0 (Infrastructure, prerequisite to everything)

### US-INFRA1 — Project scaffold (Next.js App Router + Sanity + Vercel)
**As a** dev team, **I want** a running Next.js skeleton wired to Sanity CMS and deployed to Vercel, **so that** all feature stories have a base to build on.
**Acceptance criteria:**
- Given a fresh clone, when `npm run dev` runs, then the app boots with App Router, TypeScript, Zod, and a placeholder home route.
- Given a push to `main`, when Vercel builds, then it deploys successfully with `runtime=nodejs` configured for `/api/*` routes and `Cache-Control: no-store` verified on a placeholder API route (ADR-0004).
- Given the repo, when security headers are checked, then CSP/HSTS/`X-Content-Type-Options`/`Referrer-Policy`/`Permissions-Policy` baseline (M-3) is present in `next.config`/middleware from day one.
**Data touched:** None.
**Security impact:** `security-review: yes` — security-header baseline (M-3) is foundational; misconfiguration here affects every later story.
**Estimate:** 5
**Dependencies:** External: Vercel project + domain (PRD §9).
**test_approach:** integration — CI pipeline asserts build succeeds + a header-presence smoke test against the deployed preview URL.

### US-INFRA2 — Provision managed Postgres lead store
**As a** dev team, **I want** a managed Postgres instance with encryption at rest and least-privilege roles, **so that** PII (Lead, ConsentRecord, ChatTranscript, DSRRequest) has a real store instead of a Google Sheet (ADR-0002, retires R-03).
**Acceptance criteria:**
- Given the provider is selected, when the DB is created, then AES-256 encryption at rest is enabled and confirmed (HLD B.5).
- Given app/sales/DPO roles, when granted, then the app service identity has only INSERT/SELECT on operational tables, no DDL, no DELETE on `ConsentRecord`; DPO role is separate and audited (B.5 R-19, B.6 SoD table).
- Given a connection string, when stored, then it lives only in Vercel Environment Variables / secret store, never in code or docs.
**Data touched:** Infrastructure only at this stage (no rows written yet) — but the store itself will hold PII (name, phone, zaloId, dob, productInterest, transcripts) per `erd.md` §2.
**Security impact:** `security-review: yes` — encryption, credential scoping, secret management all directly gate R-03.
**Estimate:** 5
**Dependencies:** US-INFRA1. **BLOCKED for production use on PO-2 (Postgres region/vendor + DPA decision)** — a dev/sandbox instance may proceed without PO-2, but the prod cutover cannot close until PO-2 is resolved (feeds the Art. 25 dossier per ADR-0002).
**test_approach:** integration — connect with each scoped role and assert forbidden operations (DELETE on ConsentRecord, DDL) fail; encryption-at-rest verified via provider console/API, documented as evidence.

### US-INFRA3 — ConsentRecord + PrivacyPolicyVersion schema (DB-enforced append-only)
**As a** dev team, **I want** the `ConsentRecord`, `PrivacyPolicyVersion`, `Lead`, `ChatTranscript`, `DSRRequest` tables per `erd.md`, **so that** Art. 11 consent evidence and PII are structurally correct before any feature writes to them.
**Acceptance criteria:**
- Given the schema migration runs, when applied, then all fields/types match `erd.md` §1 exactly (including `subject_ref_hash`/`context_hash`/`identifier_hash` as HMAC-SHA256, keyed — M-1).
- Given the app DB role, when it attempts `UPDATE` or `DELETE` on `ConsentRecord`, then the database rejects it (M-6 — grant-level enforcement, not application-level).
- Given a withdrawal event, when inserted, then it is a new row with `event_type=withdrawn`, never an update to the original row.
**Data touched:** PII schema definition — name, phone, zalo_id, dob (PII-enc), product_interest (PII-enc); consent metadata (internal/evidence); no card data (confirmed, `erd.md` §2).
**Security impact:** `security-review: yes` — this schema is the encryption/append-only boundary for the whole system (M-6, B.5).
**Estimate:** 3
**Dependencies:** US-INFRA2.
**test_approach:** integration — migration test + a negative test asserting DB-level rejection of UPDATE/DELETE on ConsentRecord under the app role credential.

### US-INFRA4 — CmsAdapter + catalog schema in Sanity
**As a** dev team, **I want** `CmsAdapter` wired to a Sanity schema for Collection/Product/ElementTag (ADR-0001), **so that** gallery pages and later RAG have a live, editor-managed catalog.
**Acceptance criteria:**
- Given Sanity schemas for `collection`, `product`, `elementTag`, `productElement`, when deployed, then they match `erd.md` §1 Public-content entities.
- Given a read-only API token, when configured for the app, then it cannot write (SoD — editors write via Sanity Studio with MFA, app only reads, ADR-0001).
- Given a publish event in Sanity, when it fires, then an on-publish webhook triggers ISR revalidation for the affected route (ADR-0004).
**Data touched:** Public content only (no PII) — catalog fields per `erd.md` COLLECTION/PRODUCT/ELEMENT_TAG.
**Security impact:** `security-review: no` (Public data, no PII) — but flag token scoping (read-only) for a quick check.
**Estimate:** 5
**Dependencies:** US-INFRA1. External: Sanity project provisioned.
**test_approach:** integration — schema validation test + a webhook-triggers-revalidation smoke test against a staging deploy.

### US-INFRA5 — Verified Zalo Official Account (OA)
**As a** business, **I want** a verified Zalo OA, **so that** the Zalo widget and deep-link CTAs have a real destination.
**Acceptance criteria:**
- Given Zalo's verification process, when completed, then the OA is verified and its ID is available as a (non-secret, but still env-configured) app setting.
- Given the OA exists, when the deep-link prefill syntax is checked, then it is confirmed against **current** Zalo Developer docs (PRD Open Question 1 — resolve before US-B1 build, not assumed from the research doc).
**Data touched:** None (OA setup itself is not a data-processing story).
**Security impact:** `security-review: no`.
**Estimate:** 2
**Dependencies:** External only — business/ops task, not engineering-blocked.
**test_approach:** manual — verify deep-link opens Zalo app (mobile) and Zalo Web (desktop) with correct prefill in a manual QA pass before US-B1 is marked done.

---

## 4. Backlog — Sprint 1 (MVP: Gallery + Lead Funnel)

### US-A1 — Home page
**As a** visitor, **I want** a cinematic home page highlighting featured pieces, **so that** I trust the brand and want to explore.
**Acceptance criteria:**
- Given `/` loads, when measured in the field/lab, then LCP/INP/CLS meet the "good" CWV thresholds on mobile 4G simulation (NFR-1; numeric target TBD — use Lighthouse "good" defaults as the interim bar until Product Owner confirms, per PRD Open Question 8).
- Given the visitor scrolls to the featured section, when a product card is clicked, then it navigates to that product's Product Detail page.
- Given the page is crawled, when rendered, then it is served via SSG+ISR (60 min + on-publish webhook, ADR-0004) with full HTML present (no client-only render for above-the-fold content).
**Data touched:** None (Public catalog content only).
**Security impact:** `security-review: no`.
**Estimate:** 3
**Dependencies:** US-INFRA1, US-INFRA4.
**test_approach:** e2e — Lighthouse CI budget check + Playwright click-through from featured card to PDP.

### US-A2 — Collections browsing with feng-shui filter
**As a** visitor, **I want** to browse by category and filter by ngũ hành/mệnh, **so that** I find pieces that match my element.
**Acceptance criteria:**
- Given the Collections page, when a category is selected, then only matching products display (client-side filter over the prefetched SSG+ISR set, ADR-0004).
- Given a "Mệnh" filter (Kim/Mộc/Thủy/Hỏa/Thổ) is applied, when combined with category, then only products tagged with the matching `ElementTag` display (`PRODUCT_ELEMENT` join, `erd.md`).
- Given no products match, when the empty state renders, then it shows an "Ask the chatbot" or "Message Zalo" CTA (funnel fallback — this CTA may point to the Zalo widget in Sprint 1 even before the chatbot ships in Sprint 2; link target updates when US-D1 lands).
**Data touched:** None (Public catalog).
**Security impact:** `security-review: no`.
**Estimate:** 5
**Dependencies:** US-INFRA4, US-A1 (shares layout/filter component).
**test_approach:** unit (filter logic) + e2e (Playwright: select category+mệnh, assert result set; assert empty-state CTA).

### US-A3 — Product Detail page
**As a** visitor, **I want** full detail on a piece with CTAs, **so that** I can decide to pursue it.
**Acceptance criteria:**
- Given a Product Detail page loads, when rendered, then it shows: zoomable image/video gallery, name/code, jade type, dimensions, origin, certification reference, price OR "Contact for consultation" (per `contact_for_price` flag, `erd.md`), and a feng-shui fit block.
- Given mobile viewport, when the page renders, then both primary CTAs ("Message Zalo about this piece", "Ask the AI about this piece") are visible above the fold.
- Given the product has proof media, when available, then a social-proof section renders.
- Given `Product` JSON-LD is required (US-E1), when the page is crawled, then valid schema.org markup is present (this AC is split into its own story US-E1 for testability, but the markup hook lives here).
**Data touched:** None (Public catalog; the CTAs link out to Zalo/lead-form/chat but the PDP itself displays no PII).
**Security impact:** `security-review: no` for the PDP render itself; the CTAs it hosts are covered by US-B1/US-C1a/US-D1 security reviews.
**Estimate:** 8 — **split rationale:** kept as one story since content rendering is cohesive, but if velocity requires splitting, break into US-A3a (media gallery + specs) and US-A3b (CTA placement + social proof) at 5+3.
**Dependencies:** US-INFRA4, US-A1 (shared layout).
**test_approach:** e2e — Playwright asserts all required fields render + CTA above-the-fold at mobile breakpoints (viewport assertions) + Lighthouse CWV check.

### US-A4 — Brand story / trust page
**As a** visitor, **I want** to see provenance and certification claims, **so that** I trust the products are genuine.
**Acceptance criteria:**
- Given `/brand-story`, when loaded, then it presents sourcing narrative, lab-certification explanation, and a link to the Certification showcase — reusing the exact commitment language from `create-facebook-fanpage.md` §2 Q1 (single-source-of-truth for trust copy, mitigates R-10).
**Data touched:** None (Public content).
**Security impact:** `security-review: no`.
**Estimate:** 2
**Dependencies:** US-INFRA4.
**test_approach:** manual/content-review — a copy-diff check against the Fanpage guide source text; no functional logic to unit test.

### US-B1 — Per-product Zalo deep-link CTA
**As a** visitor interested in one piece, **I want** a single tap to open Zalo with product context pre-filled, **so that** staff know exactly what I'm asking about.
**Acceptance criteria:**
- Given a Product Detail page, when "Message Zalo about this piece" is tapped, then Zalo opens (app on mobile, Zalo Web on desktop) with a prefilled message containing the product code — **using the syntax confirmed in US-INFRA5**, not the unverified research-doc assumption.
- Given the tap fires, when it resolves, then a `click_zalo` analytics event sends with `productId` (no PII — `erd.md`/AnalyticsEvent schema).
- Given Zalo's parameter syntax changes in the future (R-06), when the link builder is implemented, then it is isolated in `ZaloLinkBuilder` (single point of change, `class-diagram.md`).
**Data touched:** None directly — `click_zalo` event carries only `productId` (not PII per `api-spec.yaml` AnalyticsEvent schema).
**Security impact:** `security-review: no` (no PII in this flow) — but note the deep-link URL construction should be checked for injection/open-redirect hygiene as a light security touch.
**Estimate:** 3
**Dependencies:** US-INFRA5 (verified OA + confirmed prefill syntax), US-A3, US-G1 (event dispatch mechanism).
**test_approach:** e2e — Playwright/manual device matrix test (mobile app-open intent vs desktop Zalo Web) + analytics event assertion via network-request capture.

### US-B2 — General Zalo widget
**As** any visitor, **I want** a persistent Zalo chat entry point, **so that** I can reach a human anytime.
**Acceptance criteria:**
- Given any page renders, when loaded, then the official Zalo chat widget script is present and functional (floating widget, all viewports).
**Data touched:** None directly (widget interactions happen on Zalo's platform — `erd.md` notes "Zalo OA conversation data" as a data flow, not our store, per `data-classification.md` #14).
**Security impact:** `security-review: yes` — third-party script must be allowlisted in the CSP (M-3, US-INFRA1) and reviewed for the data it can access on-page.
**Estimate:** 2
**Dependencies:** US-INFRA5, US-INFRA1 (CSP baseline must allowlist the Zalo widget origin).
**test_approach:** e2e — Playwright smoke test asserting widget renders on 3 representative pages + CSP header inspection confirms the Zalo origin is allowlisted and nothing else broadens.

### US-H1 — Published, versioned privacy policy page
**As a** visitor, **I want** to read how my personal data is collected, used, and protected, **so that** I can make an informed decision before submitting any data.
**Acceptance criteria:**
- Given any page that collects personal data (lead form, chatbot), when the visitor looks for it, then a linked Privacy Policy page is reachable from that page and the site footer.
- Given the page loads, when rendered, then it discloses: data collected (name, phone/Zalo ID, DOB, product interest, chat transcripts, analytics identifiers), purposes, controller/processor list (OmniGem + Vercel, Cloudinary, Anthropic/Claude API, GA4, Meta Pixel, Zalo — `compliance-scope.md` Obligation B), and that some processing occurs outside Vietnam.
- Given the policy is published, when a new version goes live, then it is stored as an immutable `PrivacyPolicyVersion` (content_hash, published_at) — SSG, versioned per ADR-0004/ADR-0005 — so `ConsentRecord.policyVersion` can pin the exact text shown.
**Data touched:** None displayed to the visitor, but this story creates the versioned artifact that `ConsentRecord` (PII-adjacent evidence) depends on.
**Security impact:** `security-review: no` for the page itself; `security-review: yes` for the versioning/hash mechanism since it's the Art. 11 evidentiary anchor (US-H2 depends on its correctness).
**Estimate:** 3
**Dependencies:** US-INFRA3 (PrivacyPolicyVersion table).
**test_approach:** integration — publish a new version, assert `content_hash`/`published_at` recorded immutably; e2e link-reachability test from footer + lead form + chat entry point.

### US-H2 — Express, reproducible consent capture (Art. 11)
**As** the business, **I need** a legally sufficient consent record, **so that** PDPD Art. 11 obligations are met — not just an unlogged UI checkbox.
**Acceptance criteria:**
- Given the lead form or chatbot DOB request, when a visitor submits personal data, then the system captures a `ConsentRecord` (policyVersion, purpose, method, timestamp, hashed subject/context) durably — never merely a client-side checkbox flag (`hld-lld.md` B.1 step 4, B.3).
- Given a `ConsentRecord` exists, when requested by the visitor or an auditor, then `ConsentService.getByLead(leadId)` retrieves the full history (append-only, `class-diagram.md`).
- Given the consent checkbox defaults, when the form renders, then it is unticked by default and submission with `granted !== true` is rejected server-side (`api-spec.yaml` `ConsentAssertion.granted: const true`).
- Given DOB is basic (not sensitive) personal data (Art. 2(3), `data-classification.md` #7), when collected, then it still requires the same consent capture as any other field (no carve-out).
**Data touched:** PII — `subject_ref_hash`/`context_hash` (HMAC-SHA256 keyed, M-1) derived from phone/zalo/ip/ua; `ConsentRecord` itself classified "Internal (evidence)" per `erd.md` §2, but the hashing inputs are PII.
**Security impact:** `security-review: yes` — HMAC keying (M-1), append-only DB enforcement (M-6, verified in US-INFRA3), atomicity with the Lead write (see US-C1b).
**Estimate:** 5
**Dependencies:** US-INFRA3, US-H1 (policy version to pin).
**test_approach:** unit (ConsentService.record/getByLead logic, HMAC hashing) + integration (reject `granted=false`, assert append-only via DB constraint from US-INFRA3).

### US-C1a — Contact/consultation lead form (UI)
**As a** visitor not ready to use Zalo/chat, **I want** to leave my contact info and interest, **so that** staff can follow up.
**Acceptance criteria:**
- Given the Contact/Consultation page, when rendered, then it collects name, phone or Zalo ID, DOB, product interest, and shows the consent gate (US-H2) with a link to the current Privacy Policy version (US-H1) — unticked by default.
- Given required-field validation, when the visitor submits with missing name or no phone/zaloId, then client-side validation blocks submit with a clear error (mirrors `LeadInput` Zod schema, `api-spec.yaml`).
- Given a valid submission, when the form posts, then a loading/success state displays; on `503`, the client shows a Zalo fallback CTA instead of a bare error (NFR-4, `hld-lld.md` B.1).
**Data touched:** PII (input collection only in this story; persistence is US-C1b) — name, phone/zaloId, dob, productInterest — all `x-pii: true` per `api-spec.yaml` LeadInput schema.
**Security impact:** `security-review: yes` — client-side handling of PII before transmission; confirm no PII logged to browser console/analytics/error trackers.
**Estimate:** 3
**Dependencies:** US-H2 (consent gate component), US-A3 (linked from PDP CTA).
**test_approach:** unit (form validation logic) + e2e (Playwright: submit invalid → error shown; submit valid → success state; consent unticked → submit blocked).

### US-C1b — `POST /api/lead` — atomic lead + consent write
**As** the system, **I want** to validate and persist a lead submission with its consent record atomically, **so that** no lead ever exists without evidence of consent (and vice versa).
**Acceptance criteria:**
- Given a POST to `/api/lead`, when the body is parsed, then it is validated against the `LeadInput` Zod schema (`hld-lld.md` B.1) — reject with `400` on failure.
- Given origin/CSRF and rate-limit checks, when they fail, then respond `403`/`429` respectively (TB-1, NFR-4 abuse control).
- Given `consent.granted !== true`, when checked, then the request is rejected before any DB write — no partial state.
- Given a valid request, when processed, then `ConsentService.record()` and `LeadService.create()` execute **in the same DB transaction** (`hld-lld.md` B.1 step 4-5) — a DB failure rolls back both, never a lead without consent or vice versa.
- Given success, when the transaction commits, then respond `201` with `{status: "accepted", leadId}`, headers include `Cache-Control: no-store`, and a `submit_lead_form` analytics event fires server-side with **no PII** in the event payload.
- Given a DB outage, when the transaction fails, then respond `503` with a `zaloUrl` fallback (`api-spec.yaml` StoreUnavailable response).
**Data touched:** PII — full `LeadInput` (name, phone/zaloId, dob, productInterest — all `x-pii: true`) written to the encrypted `LEAD` table; consent metadata to `CONSENT_RECORD`.
**Security impact:** `security-review: yes` — this is one of the four sensitive `/api/*` flows explicitly called out in the task brief. Covers CSRF/origin check, rate limiting, atomic transaction integrity, encryption-at-rest write path, no-PII-in-logs (B.5 PII-scrubbed logs rule).
**Estimate:** 8
**Dependencies:** US-INFRA2, US-INFRA3, US-H2, US-C1a (client contract).
**test_approach:** integration — contract test against `api-spec.yaml` (400/403/429/503/201 paths); a DB-transaction rollback test (force a failure mid-transaction, assert neither table has a row); a log-scrub assertion (no raw PII in captured log output).

### US-E1 — Structured data & metadata (Product + Organization JSON-LD)
**As** the business, **I want** product and organization schema markup, **so that** search engines produce rich results.
**Acceptance criteria:**
- Given any Product Detail page, when crawled, then valid `Product` JSON-LD (name, image, description) is present and passes Google's Rich Results structured-data test.
- Given the site, when crawled, then `Organization` schema is present site-wide (footer/layout-level).
**Data touched:** None (Public catalog content).
**Security impact:** `security-review: no`.
**Estimate:** 2
**Dependencies:** US-A3, US-A4.
**test_approach:** unit/integration — schema validator (e.g., structured-data-testing-tool or a JSON-LD schema assertion in CI) run against a built PDP page.

### US-F1 — Non-technical catalog management via CMS
**As** shop staff, **I want** to add/edit products and prices without a developer, **so that** the catalog stays current.
**Acceptance criteria:**
- Given the Sanity Studio admin, when staff create/edit a product entry (images, specs, price or "contact for price," feng-shui tags), then the change reflects on the live site within the ISR revalidation window (30 min) or immediately via the on-publish webhook — without a redeploy.
- Given staff-only access, when a non-editor attempts CMS access, then Sanity's own auth/MFA (per ADR-0001) blocks it — no app-side PII exposure risk since CMS holds no PII (SoD).
**Data touched:** None (Public catalog; explicitly no PII in CMS per ADR-0001/`data-classification.md` #1-4).
**Security impact:** `security-review: no` for data; light check that editor role scoping (MFA, no PII access) matches ADR-0001's SoD claim.
**Estimate:** 3
**Dependencies:** US-INFRA4.
**test_approach:** manual/e2e — staff edits a product in Studio, assert live-site reflects it within the revalidation window (timed test) or instantly via webhook trigger.

### US-G1 — Funnel instrumentation (GA4 + Meta Pixel)
**As** the business, **I want** to measure the funnel from page view to lead, **so that** I can optimize spend and content.
**Acceptance criteria:**
- Given GA4 + Meta Pixel are installed, when a visitor views a product / clicks Zalo / submits the lead form / gets a chatbot handoff, then each fires as a distinct named event: `view_product`, `click_zalo`, `submit_lead_form`, `chatbot_handoff`.
- Given analytics consent has not been granted, when a page loads, then GA4/Meta Pixel tags do not load (consent-mode gate, `hld-lld.md` Flow 3 step 2) — events never carry PII (name/phone/DOB), only product IDs and event names.
- Given the optional `/api/collect` proxy is used, when an event posts, then it is dropped unless `consentAnalytics: true` (`api-spec.yaml` AnalyticsEvent schema).
**Data touched:** Online identifiers (GA4 client ID, Meta Pixel cookie) classified PII per `data-classification.md` #11/#12, but **no direct PII** (name/phone/DOB) ever included in event payloads.
**Security impact:** `security-review: yes` — consent-mode gating is a compliance control (analytics identifiers are PII under PDPD's broad definition); verify no PII leaks into event payloads or URL parameters.
**Estimate:** 3
**Dependencies:** US-A1..A3 (view_product source), US-B1 (click_zalo source), US-C1b (submit_lead_form source); US-D3 later adds chatbot_handoff.
**test_approach:** e2e — Playwright + network-request interception asserting each event fires with the correct name/payload and zero PII fields; a consent-off test asserting tags don't load at all.

---

## 5. Backlog — Sprint 2 (AI Chatbot)

### US-D-INFRA1 — Chatbot session security primitives (SessionAuthority, bot challenge, spend breaker, input scrubber)
**As** the system, **I want** SessionAuthority (HMAC-signed sessions), a bot challenge, a SpendCircuitBreaker, and a SensitiveInputScrubber, **so that** the chatbot is not exploitable for denial-of-wallet, forged history, or cross-border sensitive-data leakage before any conversational feature is built on top (H-1, H-2, C-1).
**Acceptance criteria:**
- Given a new chat session, when it starts, then a bot challenge (Cloudflare Turnstile / Vercel bot protection) must pass before `SessionAuthority.issueSigned()` returns an HMAC-signed `sessionToken` bound to the challenge result (H-1).
- Given a `sessionToken` on a subsequent request, when `verifySigned()` runs, then a forged or client-rotated token is rejected (H-1).
- Given per-session/per-IP message and token caps, when exceeded, then the request is rate-limited (`429`) server-side, not merely advised client-side (H-1).
- Given `spendCircuitBreaker.ok()` returns false (monthly cap tripped), when a chat request arrives, then the system returns the `fallback: true` / `zaloUrl` response immediately without calling Claude (`api-spec.yaml` degraded example).
- Given any text destined for the Claude API, when `SensitiveInputScrubber.scrub()` runs, then it strips Luhn-valid card-number-shaped digit runs and redacts Art. 2(4) sensitive categories **before** the text reaches `ClaudeApiAdapter.createMessage` (C-1, M-7) — verified by a unit test with synthetic Luhn-valid numbers and synthetic sensitive-category phrases.
**Data touched:** None persisted by this story alone, but it processes the raw `userMessage` (PII, `x-pii: true`) in-memory before scrubbing.
**Security impact:** `security-review: yes` — this is the guardrail infrastructure for the entire `/api/chat` sensitive flow; H-1/H-2/C-1 findings from the design-phase threat model are implemented here.
**Estimate:** 8
**Dependencies:** US-INFRA1. **PO-1 (spend cap amount) required to set `monthlyCapVnd` to a real value** — build/test can proceed with a placeholder cap, but go-live needs PO-1.
**test_approach:** unit — extensive table-driven tests for the scrubber (Luhn strings, sensitive-category phrases, mixed content) + session forgery/replay tests + a spend-breaker trip test asserting zero Claude API calls when tripped.

### US-D1 — Feng-shui consultation via chatbot
**As a** visitor, **I want** to ask the chatbot for a stone/color recommendation based on my birth date, **so that** I get a personalized suggestion.
**Acceptance criteria:**
- Given the chat widget, when a visitor provides a DOB, then the bot returns an element (ngũ hành) mapping and recommends matching product categories.
- Given the bot recommends products, when it responds, then it calls `search_products(menh, loai, giaMax)` against `ProductQueryService` → live CMS data — **never** hallucinated inventory; tool output is shape-validated before being returned to the model (H-2).
- Given the consent-first invariant (H-3, US-H2), when the first PII-bearing turn (DOB) arrives, then a session-scoped `ConsentRecord` is written **before** the DOB is used or any transcript is persisted; `consent.granted !== true` rejects the turn with nothing processed or stored.
**Data touched:** PII — `userMessage` (may contain DOB, `x-pii: true`), persisted as scrubbed `ChatTranscript.messages` (PII-enc) linked to a session `ConsentRecord`.
**Security impact:** `security-review: yes` — one of the four sensitive flows named in the task brief; consent atomicity (H-3), RAG grounding (no hallucination, R-09), server-authoritative state (H-2).
**Estimate:** 8
**Dependencies:** US-D-INFRA1, US-H2 (ConsentService reused), US-INFRA4 (CmsAdapter/ProductQueryService), US-INFRA3 (ChatTranscript schema).
**test_approach:** integration — contract test against `ChatInput`/`ChatReply` schemas; a consent-gate test (turn rejected without consent, nothing persisted); a RAG-grounding test asserting `search_products` results only ever come from seeded CMS fixtures (never invented).

### US-D2 — Trust FAQ via chatbot
**As a** visitor, **I want** the chatbot to answer authenticity/certification/return-policy questions, **so that** I don't need to wait for a human for basic questions.
**Acceptance criteria:**
- Given an FAQ-type question ("is this real jade," "what's your return policy"), when the bot responds, then it uses RAG grounded in CMS FAQ content that is content-identical to `create-facebook-fanpage.md` §2 (7-day defect exchange, lifetime string/polish service, lab certification) — single source of truth, mitigates R-10.
- Given the guardrail system prompt, when re-injected every turn (H-2), then the bot never solicits Art. 2(4) sensitive data even when answering FAQ-adjacent questions.
**Data touched:** PII — same `userMessage`/`ChatTranscript` handling as US-D1 (shared `ChatbotService`).
**Security impact:** `security-review: yes` — shares the same `/api/chat` sensitive flow and guardrail-prompt-injection surface as US-D1 (H-2).
**Estimate:** 3
**Dependencies:** US-D-INFRA1, US-D1 (shares `ChatbotService.turn()` loop), US-INFRA4 (FAQ content in CMS as single source of truth).
**test_approach:** integration — seed CMS FAQ fixtures, assert bot answers match seeded content; a content-consistency check comparing chatbot copy to the Fanpage guide source text.

### US-D3a — `handoff_to_zalo` tool + chatbot-sourced lead write
**As a** visitor showing purchase intent, **I want** to be connected to a real person, **so that** I can actually close a high-value purchase.
**Acceptance criteria:**
- Given the visitor expresses interest in a specific product or asks to buy, when detected, then the bot invokes `handoff_to_zalo(productCode?, needSummary, menh?)`, which builds a Zalo deep-link via `ZaloLinkBuilder` and calls `LeadService.create({source:"chatbot", consentId})` linked to the already-written session `ConsentRecord` (US-D1's consent step).
- Given a handoff occurs, when the lead is stored, then it lands in the same `LEAD` table as US-C1b (single source of truth), tagged `source="chatbot"`.
- Given the bot cannot confidently answer (price negotiation, custom orders), when detected, then it defaults to offering handoff rather than guessing.
- Given repeated handoff attempts in one session, when detected, then handoff-driven lead creation is rate-limited/deduped per session (H-2 anti-spam — prevents prompt-injection-forced lead spam).
**Data touched:** PII — writes to `LEAD` (name/phone/zaloId if volunteered, product interest, menh) same as US-C1b, plus links to the session `ConsentRecord` from US-D1.
**Security impact:** `security-review: yes` — same sensitive-flow classification as US-C1b (lead write) plus tool-output validation and anti-spam rate limiting (H-2).
**Estimate:** 5
**Dependencies:** US-D1 (session + consent established), US-C1b (shared `LeadService`).
**test_approach:** integration — assert `handoff_to_zalo` produces a lead row identical in shape/encryption to a form-sourced lead; a rate-limit test asserting repeated handoff calls in one session are deduped; `submit chatbot_handoff` analytics event assertion (extends US-G1).

### US-D3b — Chatbot go-live gate (spend cap + Anthropic DPA)
**As** the business, **I need** the chatbot to not serve real production traffic until cost and data-processing terms are locked, **so that** we don't incur uncontrolled spend or breach Art. 25 cross-border obligations.
**Acceptance criteria:**
- Given `SpendCircuitBreaker.monthlyCapVnd`, when set, then it reflects the **actual PO-approved figure** (PO-1), not a placeholder.
- Given production Claude API traffic, when it begins, then a signed Anthropic DPA with no-training/zero-retention terms (PO-4) is on file and referenced in the Art. 24 DPIA / Art. 25 dossiers (ADR-0003 "BLOCKING PREREQUISITE").
**Data touched:** N/A (this is a go-live gate, not a data-processing story itself).
**Security impact:** `security-review: yes` — this is the compliance/cost gate for the entire chatbot cross-border flow.
**Estimate:** 2 (verification/sign-off effort; the technical wiring is done in US-D-INFRA1/US-D1)
**Dependencies:** US-D-INFRA1, US-D1, US-D3a. **DEFERRED — blocked on PO-1 and PO-4.** Can be built/tested end-to-end in a sandbox with a placeholder cap and no live Anthropic traffic; marking this story DONE (i.e., authorizing production chatbot traffic) requires both PO decisions resolved.
**test_approach:** manual — a go-live checklist review (DPA on file, spend cap confirmed) signed off by Product Owner + Compliance before flipping the production feature flag.

---

## 6. Backlog — Sprint 3 (DSR automation + compliance dossiers)

### US-H3a — DSR intake API (`POST /api/dsr`)
**As a** visitor whose data OmniGem holds, **I want** to submit a data-subject-rights request, **so that** I can exercise my PDPD rights.
**Acceptance criteria:**
- Given a POST to `/api/dsr` with `requestType` (access/withdraw_consent/erasure/restrict/object/rectify/inform) and a self-asserted `identifier` (phone/zaloId/email), when validated against the `DsrInput` Zod schema, then a `DSRRequest` row is created with `status=received`, an accountable owner assigned, and an `ackBy` SLA deadline set (US-H3 AC2; exact SLA hours TBD by Product Owner — use 72h as the interim default per `hld-lld.md` B.4 comment).
- Given the request is logged, when accepted, then respond `202` with `{status:"received", dsrId, ackBy, verificationRequired: true}` for any disclosive/mutating type.
- Given the Privacy Policy page, when a visitor looks for a DSR channel, then a concrete channel (email or Zalo OA) is named for all six rights (US-H3 AC1) — this is a content requirement on US-H1, cross-referenced here.
**Data touched:** PII — `identifier` (phone/zaloId/email, `x-pii: true`) and `details` (`x-pii: true`), written to `DSR_REQUEST.identifier_hash` (HMAC-SHA256 keyed, M-1) and `details` (PII-enc).
**Security impact:** `security-review: yes` — one of the four sensitive flows named in the task brief; identifier hashing (M-1), rate limiting.
**Estimate:** 5
**Dependencies:** US-INFRA3 (DSRRequest schema), US-H1 (channel named on the Privacy Policy page).
**test_approach:** integration — contract test against `DsrInput`/`DsrAccepted` schemas (400/429/202 paths); assert `identifier_hash` uses keyed HMAC not bare SHA-256.

### US-H3b — DSR identity verification (H-4)
**As** the system, **I want** to verify a DSR requester controls the on-record channel before any disclosive or mutating action, **so that** an impersonator cannot exfiltrate or destroy a victim's data using just their phone/Zalo/email.
**Acceptance criteria:**
- Given a DSR of type access/inform/erasure/rectify/restrict/withdraw_consent, when intake completes, then status moves to `pending_verification` and a one-time code is sent to the **on-record** channel (the phone/Zalo/email already stored for that subject) — **never** to a value the requester re-supplies in the request itself (H-4).
- Given the OTP is returned correctly within N minutes, when verified, then status advances to `verified`; given it expires or fails, then status remains `pending_verification` and no fulfilment proceeds.
- Given intake/acknowledgement, when a request arrives, then it is never blocked from *starting* even before verification — only fulfilment is gated (H-4 rationale, `hld-lld.md` B.4).
**Data touched:** PII — the on-record contact channel used for OTP delivery; no new PII collected beyond what's already in the matched `LEAD`/`DSR_REQUEST` records.
**Security impact:** `security-review: yes` — this is the core identity-spoofing mitigation (H-4) for the DSR flow; a design gap here would let an attacker impersonate a data subject.
**Estimate:** 5
**Dependencies:** US-H3a.
**test_approach:** integration — a negative test asserting OTP is sent only to the on-record channel (not an attacker-supplied one) + an expiry test + a "verified-before-fulfil" enforcement test.

### US-H3c — DSR fulfilment with maker-checker approval
**As a** DPO, **I want** disclosive/mutating DSR actions to require a distinct human approver, **so that** SoD (P5 maker-checker) is enforced and a single compromised/malicious actor cannot both request and approve a destructive action.
**Acceptance criteria:**
- Given a `verified` DSR request, when `DsrService.fulfil(dsrId, action, approverId)` is called, then `approverId` must be a distinct DPO-role identity from whoever performed intake/verification triage (P5 maker-checker, `class-diagram.md` DsrService invariant).
- Given `erasure`/`withdraw_consent` is fulfilled, when actioned, then the `LEAD` record is anonymized/status=`erased` and `CHAT_TRANSCRIPT` is flagged for removal — **subject to legal-retention override** (e.g., an active DPIA/audit record) per US-H3 AC3.
- Given `access`/`inform` is fulfilled, when actioned, then the subject receives their held data via the named channel (US-H3 AC1), logged as a disclosure event.
**Data touched:** PII — the full `LEAD`/`CHAT_TRANSCRIPT` records being disclosed, anonymized, or erased.
**Security impact:** `security-review: yes` — maker-checker enforcement is a P5 SoD control; erasure/anonymization logic touches encrypted PII directly.
**Estimate:** 8
**Dependencies:** US-H3b, US-INFRA2 (DPO-scoped DB role from US-INFRA2's role design).
**test_approach:** integration — assert `fulfil()` rejects when `approverId === requesterId`/triage identity; an erasure test asserting PII columns are nulled/crypto-shredded and status flips to `erased`; a retention-override test (active legal hold blocks erasure with a clear reason).

### US-H-COMP1 — DPIA dossier (Art. 24, R-12)
**As** the business (data controller), **I need** a DPIA dossier prepared and filed within 60 days of starting PII processing, **so that** Decree 13/2023 Art. 24 obligations are met (risk-register R-12).
**Acceptance criteria:**
- Given the HLD/LLD, ERD, and data flows (already produced at Design phase), when compiled, then the DPIA dossier enumerates every PII store and processing purpose per `hld-lld.md` §C traceability notes.
- Given the dossier is complete, when filed, then it is available to / filed with the Ministry of Public Security (A05) within the Art. 24 window.
**Data touched:** References all PII entities system-wide (Lead, ConsentRecord, ChatTranscript, DSRRequest) as a compliance artifact — this story does not itself process new data.
**Security impact:** `security-review: no` (this is a compliance-checker/Compliance-team artifact, not code) — flag for `compliance-checker` review, not `security-reviewer`.
**Estimate:** 8 (documentation/legal-coordination effort, not engineering)
**Dependencies:** All Sprint 1 PII-touching stories must be built (US-C1b, US-H2) to accurately describe processing; the chatbot flow (US-D1/D3a) should also be described before filing if chatbot ships before the dossier deadline.
**DEFERRED — blocked on PO-3 (named DPIA owner).** Content can be drafted by compliance-checker/requirements-analyst now; filing/ownership cannot close without PO-3.
**test_approach:** manual — Compliance sign-off checklist; not a code-testable story.

### US-H-COMP2 — Cross-border transfer impact assessment dossier (Art. 25, R-04)
**As** the business (data controller), **I need** a Cross-Border Transfer Impact Assessment dossier for Vercel, Cloudinary, and Anthropic, **so that** Decree 13/2023 Art. 25 obligations are met before those vendors process VN personal data.
**Acceptance criteria:**
- Given the per-vendor data-flow breakdown (`compliance-scope.md` Obligation B, `hld-lld.md` §C Art. 25 note), when compiled, then the dossier covers what data crosses the border to each vendor (Vercel logs, Cloudinary images — low-PII, Anthropic — scrubbed chat transcripts), the safeguards in place (SensitiveInputScrubber, no-training DPA), and is filed with A05 within 60 days of transfer start.
- Given the Anthropic DPA (PO-4), when referenced, then the dossier cites its no-training/zero-retention terms as the primary safeguard for the chatbot's cross-border leg.
**Data touched:** References PII in transit to cross-border vendors (compliance artifact, not new processing).
**Security impact:** `security-review: no` — Compliance-team artifact; `compliance-checker` review, not `security-reviewer`.
**Estimate:** 8
**Dependencies:** US-D-INFRA1 (scrubber must exist and be describable), US-D3b (Anthropic DPA referenced).
**DEFERRED — blocked on PO-3 (named cross-border TIA owner) and, for the Anthropic section specifically, PO-4 (DPA must be signed to describe its terms accurately).**
**test_approach:** manual — Compliance sign-off checklist; not a code-testable story.

---

## 7. Backlog — Sprint 4+ (Growth, P4 — noted, not fully elaborated)

### US-E2 — SEO content (blog)
**As a** prospective buyer researching jade, **I want** educational content, **so that** I find OmniGem via organic search.
**Acceptance criteria:**
- Given the CMS, when a content editor publishes a blog post, then it is reachable at a stable URL, indexable, and internally links to relevant collections/products.
**Data touched:** None (Public content).
**Security impact:** `security-review: no`.
**Estimate:** 5
**Dependencies:** US-INFRA4, US-F1 (shared CMS editing workflow).
**test_approach:** e2e — publish-and-crawl smoke test (sitemap inclusion, internal link presence).

*(Remarketing pixels, CTA A/B tests, and Zalo ZNS care messages from PRD §4 P4 are noted as future backlog items — not broken into stories here since they depend on Sprint 1–3 baseline data/infra that doesn't exist yet; re-scope at the start of that phase per PRD §4.)*

---

## 8. Summary table (quick reference)

| ID | Title | Sprint | Points | security-review | Status |
|---|---|---|---|---|---|
| US-INFRA1 | Project scaffold | 0 | 5 | yes | Ready |
| US-INFRA2 | Postgres lead store | 0 | 5 | yes | Ready (prod cutover blocked on PO-2) |
| US-INFRA3 | Consent/PII schema | 0 | 3 | yes | Ready |
| US-INFRA4 | CmsAdapter + Sanity schema | 0 | 5 | no | Ready |
| US-INFRA5 | Verified Zalo OA | 0 | 2 | no | Ready (external/ops) |
| US-A1 | Home page | 1 | 3 | no | Ready |
| US-A2 | Collections + filter | 1 | 5 | no | Ready |
| US-A3 | Product Detail page | 1 | 8 | no | Ready |
| US-A4 | Brand story page | 1 | 2 | no | Ready |
| US-B1 | Zalo deep-link CTA | 1 | 3 | no | Ready |
| US-B2 | Zalo widget | 1 | 2 | yes | Ready |
| US-H1 | Privacy Policy page | 1 | 3 | yes | Ready |
| US-H2 | Consent capture (Art. 11) | 1 | 5 | yes | Ready |
| US-C1a | Lead form UI | 1 | 3 | yes | Ready |
| US-C1b | `/api/lead` atomic write | 1 | 8 | yes | Ready |
| US-E1 | JSON-LD structured data | 1 | 2 | no | Ready |
| US-F1 | CMS catalog editing | 1 | 3 | no | Ready |
| US-G1 | Funnel analytics | 1 | 3 | yes | Ready |
| US-D-INFRA1 | Chatbot security primitives | 2 | 8 | yes | Ready (go-live needs PO-1) |
| US-D1 | Feng-shui consult | 2 | 8 | yes | Ready |
| US-D2 | Trust FAQ | 2 | 3 | yes | Ready |
| US-D3a | Handoff + lead write | 2 | 5 | yes | Ready |
| US-D3b | Chatbot go-live gate | 2 | 2 | yes | **DEFERRED** — PO-1, PO-4 |
| US-H3a | DSR intake API | 3 | 5 | yes | Ready |
| US-H3b | DSR identity verification | 3 | 5 | yes | Ready |
| US-H3c | DSR fulfilment + maker-checker | 3 | 8 | yes | Ready |
| US-H-COMP1 | DPIA dossier | 3 | 8 | no (compliance) | **DEFERRED** — PO-3 |
| US-H-COMP2 | Cross-border TIA dossier | 3 | 8 | no (compliance) | **DEFERRED** — PO-3, PO-4 |
| US-E2 | SEO blog | 4 | 5 | no | Ready |

**Totals:** 29 stories. Sprint-0+1-ready (infra + MVP): 18 (US-INFRA1..5 + US-A1..A4 + US-B1..B2 + US-H1..H2 + US-C1a..C1b + US-E1 + US-F1 + US-G1) = **70 points** (20 infra + 50 MVP-feature). Deferred: 3 (US-D3b, US-H-COMP1, US-H-COMP2), each deferred on named PO decisions, not on unclear requirements.
