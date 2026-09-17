# OmniGem Gallery — NFR Targets (P2 Design)

Status: DRAFT for Gate G1. Author: solution-architect. Approvers: Architect + Product Owner.
Fills PRD §6 TBDs (NFR-1…NFR-7) and resolves risk R-11. **Numbers are PROPOSALS for PO/Architect confirmation** unless marked FIRM.

G1 gate criterion "NFR targets defined (latency, throughput, availability)" maps to §2, §3, §4 below.

## 1. Posture

Marketing/lead-gen site, mobile-first, Vietnam 4G, high-resolution jade media, low cost ceiling. Not transactional/banking-grade — availability and latency targets are calibrated accordingly, but **PII-handling endpoints (`/api/lead`, `/api/chat`, `/api/dsr`) get firmer correctness/security targets** because they carry the compliance surface.

## 2. Performance — Core Web Vitals (NFR-1, NFR-2; R-08)

Field data (CrUX / RUM), **mobile 4G, 75th percentile**, "good" thresholds:

| Metric | Target (p75 mobile) | Status | Notes |
|---|---|---|---|
| LCP (Largest Contentful Paint) | ≤ 2.5 s | PROPOSED FIRM | Hero jade imagery is the LCP element — needs Cloudinary responsive + `next/image` priority + preload |
| INP (Interaction to Next Paint) | ≤ 200 ms | PROPOSED FIRM | Filter interactions (mệnh/category), chatbot open |
| CLS (Cumulative Layout Shift) | ≤ 0.1 | PROPOSED FIRM | Reserve image/video dimensions; no layout jump on CTA render |
| TTFB (edge) | ≤ 0.8 s | PROPOSED | SSG/ISR served from Vercel edge/CDN |
| Lighthouse Perf (lab, mobile) | ≥ 90 | PROPOSED (CI budget) | Gate in CI on key templates (Home, Collections, Product) |

### Image performance budget (R-08)
- Product Detail page total transferred image weight (initial viewport): **≤ 1.0 MB** on mobile; lazy-load below-the-fold gallery.
- Hero LCP image: **≤ 200 KB** (AVIF/WebP via Cloudinary `f_auto,q_auto`).
- Every `<img>` responsive (`srcset`/`sizes`) and explicitly sized. Light-transmission video: poster image + lazy, click-to-play, never autoplay-with-audio.

## 3. Availability / SLA (NFR-4)

| Item | Target | Status |
|---|---|---|
| Site (SSG/ISR pages) availability | **99.5% / month** (≈ 3h39m/mo budget) | PROPOSED |
| `/api/lead` + `/api/dsr` availability | 99.5% | PROPOSED |
| `/api/chat` availability (app route) | 99.0% (degrades gracefully if Claude API is down → fall back to Zalo handoff CTA) | PROPOSED |
| Graceful degradation | If CMS/ISR stale, serve last-good; if Claude API errors, chatbot must **offer Zalo handoff**, never a hard error (R-09/R-06) | FIRM (design rule) |

Rationale: inherits Vercel's platform SLA; 99.5% is appropriate for a non-transactional marketing site (PRD NFR-4). Lead capture must not silently fail — on write failure return a retriable error and surface the Zalo fallback CTA.

## 4. API latency & throughput (NFR-1)

Server-side, measured at the Vercel function, excluding third-party time where noted:

| Endpoint | p95 latency target | Throughput (design) | Status |
|---|---|---|---|
| `POST /api/lead` | ≤ 500 ms (excl. downstream write retries) | ~5 req/s sustained, burst 50 | PROPOSED |
| `POST /api/chat` (per turn, app overhead) | ≤ 800 ms app-side **excluding** Claude API model latency; end-to-end first-token ≤ 3 s | ~3 concurrent conversations sustained, burst 20 | PROPOSED |
| `POST /api/dsr` | ≤ 500 ms | Very low volume | PROPOSED |
| `POST /api/collect` (analytics proxy, if built) | ≤ 200 ms | Matches pageview volume | PROPOSED |

Rate limits (abuse/cost control, FIRM as design rule): `/api/lead` and `/api/dsr` — per-IP token bucket (e.g., 5/min, 30/hour); `/api/chat` — per-session cap on turns + per-IP cap to bound Claude API spend (ties to NFR-6 and R-04 data-egress volume).

## 5. Accessibility (NFR-5)

- **WCAG 2.1 AA** baseline — PROPOSED FIRM for all interactive elements: lead form (labels, error announcements), CTAs, chatbot widget (keyboard operable, focus trap, ARIA live region for bot messages), mệnh filter controls. Color contrast ≥ 4.5:1 on the dark/jade palette.

## 6. Cost posture (NFR-6)

- Intent: "promotion + SEO + speed + low cost." Monthly infra ceiling amount is PO-set (placeholder: Vercel Pro + Cloudinary free/plus + Claude API usage-based).
- FIRM design rules to protect the ceiling: aggressive ISR caching (few origin hits), Cloudinary `q_auto`, chatbot **server-enforced** per-session/turn caps + prompt-size minimization + prompt caching of the system prompt/RAG scaffold, GA4/Pixel client-side (no server cost).
- **Claude API spend circuit-breaker — PROMOTED TO FIRM (review H-1, denial-of-wallet).** A hard monthly Claude API spend cap is an **enforced control, not just an alert**: when tripped, `/api/chat` fails over to a Zalo-handoff-only mode (graceful, per NFR-4). Combined with a mandatory **bot challenge** (Turnstile/Vercel bot protection) and **HMAC-signed server-issued session ids** on `/api/chat`, this bounds the cost blast-radius of an unauthenticated public LLM endpoint. The PO sets the cap *amount*; the *enforcement mechanism* is firm regardless of the amount.

## 7. Data-protection & security NFRs (NFR-7) — carried from compliance

| NFR | Target | Status |
|---|---|---|
| Encryption in transit | TLS 1.2+ everywhere (HSTS on) | FIRM |
| Encryption at rest | Lead Store + ConsentRecord + ChatTranscript encrypted at rest (R-03) | FIRM |
| Consent reproducibility | 100% of PII submissions produce a retrievable ConsentRecord (Art. 11, US-H2) | FIRM |
| DSR handling SLA | Acknowledge ≤ 72h; fulfil within PDPD timeframe (exact SLA — PO to confirm, US-H3 AC2) | PROPOSED |
| Secrets | 0 secrets in source/client bundles (Vercel secret store only) | FIRM |
| Cross-border dossier readiness | Art. 25 + Art. 24 dossiers filed before go-live (R-04, R-12) | FIRM (compliance-owned) |

## 8. Summary — headline numbers for sign-off

LCP ≤ 2.5s / INP ≤ 200ms / CLS ≤ 0.1 (p75 mobile 4G) · Availability 99.5% · `/api/lead` p95 ≤ 500ms · `/api/chat` first-token ≤ 3s (app overhead ≤ 800ms) · WCAG 2.1 AA · TLS 1.2+ & at-rest encryption FIRM · cost ceiling TBD by PO.
