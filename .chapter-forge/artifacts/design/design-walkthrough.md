# OmniGem Gallery — Design Walkthrough Note (Gate G1)

Status: DRAFT for Gate G1 sign-off. Author: solution-architect (Maker). Approvers (Checker): **Architect + Security**.
Design loop: solution-architect → threat-modeler (STRIDE) + security-reviewer (OWASP/PCI), in parallel → revise → **CONVERGED (no unresolved Critical/High)**.
Companion artifacts: `system-context.md`, `nfr-targets.md`, `hld-lld.md`, `api-spec.yaml`, `erd.md`, `class-diagram.md`, `adr/ADR-0001..0005`, `threat-model.md`, `security-review.md`, `gate-g1-checklist.md`.

## 1. Design summary

OmniGem Gallery is a **single Next.js (App Router) marketing/lead-gen site on Vercel** for a Vietnamese natural-jade brand — a digital showroom and funnel into Zalo, **not** an e-commerce checkout (no cart, no payment, **no card data, no PCI scope**). Public catalog/blog content lives in a headless CMS (Sanity — ADR-0001) and is served **SSG/ISR** for SEO and Core Web Vitals; product imagery is delivered via Cloudinary. Three server API routes handle personal data and are always dynamic and uncached: `/api/lead` (consultation form), `/api/chat` (a self-hosted Anthropic Claude chatbot with `search_products` RAG over live CMS data and a mandatory `handoff_to_zalo` tool — ADR-0003), and `/api/dsr` (data-subject-rights intake). All PII (Lead, ConsentRecord, ChatTranscript, DSRRequest) is stored in a **proper encrypted managed store, not a Google Sheet** (ADR-0002), with encryption in transit (TLS 1.2+) and at rest, keyed-HMAC pseudonymization, DB-enforced append-only consent (Art. 11), and Separation of Duties across content-editor / lead-accessor / DPO / deployer roles. Cross-border PII egress (Vercel, Cloudinary, Anthropic) is enumerated for the PDPD Art. 24 DPIA and Art. 25 dossiers.

## 2. Findings-resolution table

| ID | Severity | Title | Raised by | Resolution | Re-verified by |
|---|---|---|---|---|---|
| C-1 / R-16 | CRITICAL | Cross-border redaction mis-ordered (scrub ran after the Claude call) | threat-modeler + security-reviewer | **FIXED** — `SensitiveInputScrubber` moved input-side, **before** `ClaudeApiAdapter.createMessage`; redacts Art. 2(4) sensitive categories + strips Luhn-valid card strings; Anthropic no-training/zero-retention DPA made a blocking go-live prerequisite (ADR-0003, `hld-lld` Flow 2/B.2) | both, re-verified |
| H-1 / R-13 | HIGH | `/api/chat` denial-of-wallet (unauth endpoint, client-rotatable sessionId, cost only alerted) | security-reviewer | **FIXED** — bot challenge (Turnstile/Vercel) + server-issued **HMAC-signed sessionId** + server-enforced per-session caps + **FIRM global spend circuit-breaker** → Zalo fallback (`hld-lld` B.2, `nfr-targets §6`) | both, re-verified |
| H-2 / R-14 | HIGH | Prompt-injection + client-forged assistant history → guardrail bypass / handoff spam | threat-modeler | **FIXED** — server-authoritative state; client sends only the new user message + signed session (no assistant turns); guardrail prompt re-injected each turn; tool-output validation; per-session handoff-lead rate-limit (`api-spec` ChatInput, `hld-lld` B.2) | both, re-verified |
| H-3 / R-17 | HIGH | Chatbot consent non-atomic (PII processed/sent before any ConsentRecord) | security-reviewer | **FIXED** — `consent` now **required** in ChatInput; session-scoped ConsentRecord written on first PII-bearing turn **before** persistence/DOB use/cross-border call (ADR-0005, `erd` consent_id FK) | both, re-verified |
| H-4 / R-15 | HIGH | `/api/dsr` no requester identity verification → impersonation exfiltration/destruction | threat-modeler | **FIXED** — `verifyIdentifier` (OTP/match-challenge to on-record channel) required before any disclosive/mutating fulfilment; `received→pending_verification→verified`; fulfilment also gated by P5 maker-checker (`hld-lld` B.4, `api-spec` /dsr) | both, re-verified |
| M-1 | MEDIUM | Low-entropy identifier hashes brute-forceable | security-reviewer | **RESOLVED** — keyed **HMAC-SHA256** for subject/context/identifier hashes; key in secret store (`hld-lld` B.5, `erd`) | re-verified |
| M-3 | MEDIUM | Missing security-header baseline | security-reviewer | **RESOLVED** — strict CSP allowlist + HSTS + nosniff + Referrer-Policy + Permissions-Policy in middleware (`hld-lld` B.5) | re-verified |
| M-4 / R-19 | MEDIUM | Over-broad DB credentials | security-reviewer | **RESOLVED** — least-privilege app role (no DDL, no consent DELETE); separate audited DPO role for erasure (`hld-lld` B.5/B.6) | re-verified |
| M-6 | MEDIUM | Consent immutability only in app code | security-reviewer | **RESOLVED** — DB-enforced append-only (revoke UPDATE/DELETE on ConsentRecord for app role); withdrawal = new row (ADR-0005) | re-verified |
| M-7 | MEDIUM | Card-shaped strings could reach the model | threat-modeler | **RESOLVED** — folded into C-1 scrubber (Luhn strip before the cross-border call); preserves no-card-data invariant | re-verified |
| — | MEDIUM | Raw PII in server logs | security-reviewer | **RESOLVED** — PII-scrubbed logs (hashed/redacted request context only); FIRM operational rule feeding the DPIA (`hld-lld` B.1/B.5) | re-verified |

## 3. Accepted with rationale (consciously not fully closed at design)

| # | Item | Disposition | Rationale / backstop | Owner · Target |
|---|---|---|---|---|
| a | Free-text sensitive-data scrubbing is heuristic — cannot guarantee 100% recall of incidentally volunteered Art. 2(4) data | **ACCEPTED-RISK** | Residual leakage is backstopped by the **blocking Anthropic no-training / zero-retention DPA** (ADR-0003) so any residue is not retained/trained on; input-side scrub + no-solicitation system prompt minimize incidence; documented in the DPIA (Art. 24) | Compliance/DPO · DPIA before go-live |
| b | M-5 supply-chain / SCA dependency-scanning gate | **DEFERRED** | Belongs to the CI/build gate, not design; the P4 CI gate already mandates 0 Critical/High SAST/SCA + 0 leaked secrets (`sdlc-graph` CI gate) — not a G1 blocker | dev-executor · P4 CI |
| c | DPO/staff authentication channel (IdP + MFA) not fully specified | **ACCEPTED (spec at build)** | The **authorization model / SoD is sound and complete** (B.6); the concrete IdP + MFA enrollment mechanism is a build-time choice, not an architectural gap | Architect · P4 build |
| d | LOWs carried forward: ISR staleness window (≤30–60 min), Zalo platform lock-in (R-06), organic-SEO ramp lag (R-07) | **ACCEPTED** | Business-acceptable for a non-transactional marketing site; on-publish webhooks reduce staleness; phone/hotline + widget fallback for Zalo; KPI expectations set for months 1–3 | Product Owner · monitor |

## 4. FIRM / blocking commitments implementation MUST honor

1. **Scrubber-before-Claude ordering** — `SensitiveInputScrubber` runs input-side, before every `ClaudeApiAdapter.createMessage`; never storage-time-only.
2. **Signed Anthropic DPA (no-training / zero-retention) before go-live** — no production chat traffic until in place; referenced by DPIA + Art. 25 dossiers.
3. **HMAC-signed, server-issued session ids** on `/api/chat`; client-supplied session values never trusted; no client assistant history accepted.
4. **Spend circuit-breaker** — enforced hard Claude spend cap → Zalo-handoff fallback (not merely alerted).
5. **DB-enforced append-only ConsentRecord** — grant-level immutability; consent required (form + chat) and atomic with the data it authorizes.
6. **Least-privilege DB roles** — scoped app role; separate audited DPO erasure role; DSR fulfilment behind identity verification + maker-checker.
7. **CSP + security-header baseline** and **PII-scrubbed logs** across the app; TLS 1.2+ everywhere; encryption at rest for all PII entities; secrets only in the Vercel secret store.

## 5. Self-assessment vs Gate G1 criteria

| G1 criterion | Self-assessment | Evidence |
|---|---|---|
| NFR targets defined (latency, throughput, availability) | **SATISFIED** (numbers proposed, PO/Architect to confirm amounts) | `nfr-targets.md` §2–§4, §8 |
| Threat model (STRIDE) complete | **SATISFIED** | `threat-model.md` (converged, no open Critical/High) |
| Sequence diagram(s) cover every sensitive flow | **SATISFIED** — all 3 flows (lead, chatbot+handoff, analytics) modeled | `threat-model.md` + `hld-lld.md` §A.3 |
| ERD complete, PII/card-data fields flagged | **SATISFIED** — PII flagged; card data confirmed none | `erd.md` §1–§2 |
| Architecture review passed | **SATISFIED pending human sign-off** — reviewers re-verified; this walkthrough is the self-check | this doc + `security-review.md` |
| SoD design & encryption at-rest/in-transit met | **SATISFIED** | `hld-lld.md` B.5 (encryption) + B.6 (SoD matrix) |
| ADR approved | **DRAFTED — awaits human approval** (status PROPOSED) | `adr/ADR-0001..0005` |
| All findings resolved or accepted with rationale | **SATISFIED** — §2 (all FIXED/RESOLVED) + §3 (accepted/deferred) | this doc |

**Overall:** all G1 criteria are satisfied by the artifacts; the remaining step is the **human Architect + Security sign-off** (ADR approval + architecture-review acceptance) — the Maker cannot self-approve (P4 SoD / P5 four-eyes).
