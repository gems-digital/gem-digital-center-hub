# OmniGem Gallery — Security Design Review (P2 / Gate G1)

Status: DRAFT for Gate G1. Reviewer: security-reviewer (independent, READ-ONLY). Author under review: solution-architect.
Method: OWASP Top 10 (2021) + ASVS-style control check + PCI boundary check, against the DESIGN only (no code exists yet).
Artifacts reviewed: `system-context.md`, `hld-lld.md` (§A.3, §B.5, §B.6), `api-spec.yaml`, `erd.md`, `class-diagram.md`, `nfr-targets.md`, `adr/ADR-0001..0005`, plus discover context (`data-classification.md`, `compliance-scope.md`, `risk-register.md`).
No real PII/PAN appears in this file. All identifiers are synthetic/field-level.

Severity: CRITICAL / HIGH block Gate G1 (mandatory revision loop). MEDIUM / LOW are advisory and should be tracked.

---

## 0. Summary verdict

The design is fundamentally sound on the hardest items: encryption in transit/at rest (§B.5), secrets in Vercel env (not source), atomic + append-only versioned consent (§B.3, ADR-0005), Zod validation at every boundary, `no-store` on all PII responses, a documented SoD matrix (§B.6), and an explicit, correct **no-card-data / no-PCI** posture.

Two design-level gaps rise to **HIGH** and should trigger a revision back to the architect:
- **H-1** — the unauthenticated `/api/chat` → paid Claude API path has only IP/session rate limits (trivially bypassed) with no bot mitigation or hard spend circuit-breaker: a denial-of-wallet **and** amplified cross-border PII-egress vector.
- **H-2** — the DSR flow (`/api/dsr`) has no requester **identity-verification** control specified anywhere, so fulfilling an `access` request discloses a third party's PII, and an `erasure`/`rectify` request lets an attacker destroy/alter another subject's record.

No CRITICAL findings. No card data anywhere in the design (see §PCI).

---

## A01 — Broken Access Control

**Adequate:**
- No lead-read API exists (no `GET /leads`, no id-addressable PII endpoint) → no classic IDOR on the lead store via the public API.
- SoD matrix (§B.6) is clearly articulated: editors ↔ CMS only, sales = read-only audited, DPO = DSR/erasure + consent export, deployer manages secrets but not PII, app uses a scoped DB credential + read-only CMS token.

### [HIGH] H-2 — DSR fulfilment has no requester identity verification (R-16, relates R-01)
- **Artifact:** `hld-lld.md` §B.4; `api-spec.yaml` `/dsr`; `erd.md` `DSR_REQUEST`.
- **Weakness (CWE-284 / CWE-639):** `/api/dsr` accepts a `requestType` of `access|erasure|rectify|...` keyed only on a self-asserted `identifier` (phone/zaloId/email). The design defers fulfilment to a "controlled back-office action" but specifies **no identity-proofing step** before that action executes. Anyone can submit an `access` request for a victim's phone number, or an `erasure`/`rectify` for it.
- **Impact:** Confidentiality — `access` fulfilment returns a data subject's PII to an impersonator (a reportable PDPD breach). Integrity/Availability — malicious `erasure`/`rectify` destroys or corrupts another person's lead/consent evidence.
- **Fix direction:** Add an explicit design control: verify requester control of the claimed identifier before fulfilling any confidentiality- or integrity-affecting DSR (e.g., OTP/challenge to the phone/Zalo/email on record, or match against an already-authenticated channel). Document it as a firm step in the DsrService fulfilment path and in the SoD/maker-checker gate (P5).

### [MEDIUM] M-4 — No defined authenticated access path for staff/DPO to lead-store PII (R-18)
- **Artifact:** `system-context.md` staff flows; `hld-lld.md` §B.6.
- **Weakness:** §B.6 assigns roles but the *mechanism* by which sales/DPO authenticate and reach PII is unspecified (direct DB console? an admin UI? shared vs named accounts?). SoD is only enforceable if backed by named individual identities + row/role-level grants + MFA. "least-privilege, audited" is currently aspirational.
- **Fix direction:** Specify the access channel and IdP for PII access, individual (non-shared) accounts, MFA, and how sales-read vs DPO-write grants are enforced at the store (role/RLS), so §B.6 is technically enforced, not just documented.

---

## A02 — Cryptographic Failures

**Adequate:** TLS 1.2+ + HSTS in transit; AES-256 at rest + application/column-layer encryption for PII-enc fields; KMS-held keys, never in code; backups under same key policy; secrets server-only (no `NEXT_PUBLIC_`). This directly retires R-03 alongside ADR-0002. Publishable keys (GA4/Pixel/Cloudinary cloud name) correctly classified non-secret.

### [MEDIUM] M-1 — Low-entropy identifiers pseudonymised with a plain hash (R-17)
- **Artifact:** `erd.md` `CONSENT_RECORD.subject_ref_hash` / `context_hash`, `DSR_REQUEST.identifier_hash`; `hld-lld.md` §B.3.
- **Weakness (CWE-916 / CWE-759):** A VN phone number is a ~10-digit space; an unsalted/unkeyed hash of it is trivially brute-forced (full rainbow enumeration in seconds). As a "lookup key" it provides little confidentiality if the store is exposed.
- **Fix direction:** Use a **keyed HMAC** with a KMS-managed pepper (not a bare SHA) for all subject/identifier lookup hashes, so the mapping is not reversible without the key. Keep the raw identifier only in the encrypted column.

---

## A03 — Injection

**Adequate:** Zod schemas with `additionalProperties:false`, length/regex/enum bounds on every field of every endpoint (`api-spec.yaml`). SQL/ORM: §C invariant "writes go through the repository, never raw SQL from route handlers" implies parameterization — good; keep it FIRM.

### [MEDIUM] M-2 — LLM prompt-injection / tool-abuse and GROQ-injection not addressed
- **Artifact:** `hld-lld.md` §B.2; `class-diagram.md` §3; ADR-0003.
- **Weakness:**
  - **Prompt injection (direct + indirect via CMS FAQ RAG content):** the design covers *policy* guardrails (never solicit sensitive data, mandatory handoff) but not *adversarial* manipulation — a user (or poisoned CMS content) steering the model to exfiltrate the system prompt or to invoke `handoff_to_zalo` with attacker-chosen `needSummary`/`productCode`, creating spurious/poisoned leads.
  - **GROQ/CMS-query injection:** `search_products` args (esp. free-string `loai`) originate from model output influenced by the user; if interpolated into a GROQ/CMS query they must be parameterized/escaped.
- **Impact:** Bounded (tools are read-only search + link-builder; no lead-store read tool), but spurious lead writes and prompt disclosure are realistic.
- **Fix direction:** State the trust boundary that tool outputs are untrusted; validate `search_products` args against the declared enums server-side (reject free-form `loai` beyond an allow-list); parameterize CMS queries; keep the LLM out of any privileged write beyond the single, schema-validated handoff lead; treat CMS RAG content as untrusted input to the prompt.

---

## A04 — Insecure Design

**Adequate:** Consent is atomic (same DB tx), reproducible, versioned, append-only, server-rejects `granted!==true` (ADR-0005, §B.3) — a strong Art. 11 design. Rate limits defined (NFR §4). Graceful degradation to Zalo is a FIRM rule.

### [HIGH] H-1 — Unauthenticated `/api/chat` → paid LLM: denial-of-wallet + amplified cross-border PII egress (R-13, relates R-04)
- **Artifact:** `hld-lld.md` §B.2; `nfr-targets.md` §4 (rate limits), §6 (cost "TBD by PO", alert-only).
- **Weakness (CWE-770 / CWE-799):** the only controls on a fully public, unauthenticated endpoint that calls a metered third-party API are a per-IP token bucket and a "per-session cap." `sessionId` is client-supplied and **not server-bound**, so an attacker rotates it freely; per-IP limits fall to distributed/rotating sources. There is no bot/CAPTCHA challenge, no proof-of-work, and — critically — cost is only **alerted**, not **enforced** ("cost ceiling TBD by PO"). Each abusive turn both burns Claude spend and pushes free-text **across the border to Anthropic** (R-04), so abuse is simultaneously a financial-DoS and a PII-egress amplifier.
- **Impact:** Unbounded API spend ("denial of wallet"); inflated cross-border transfer volume that undermines the Art. 25 data-minimization posture.
- **Fix direction:** Add bot mitigation on `/api/chat` (e.g., Turnstile/hCaptcha or Vercel bot protection), issue+bind `sessionId` server-side (signed token) so per-session caps are real, and add a **hard global spend/turn circuit-breaker** that fails over to the Zalo handoff CTA when a budget threshold is hit (not just an alert). Promote the cost ceiling from "TBD/alert" to an enforced FIRM control.

---

## A05 — Security Misconfiguration

**Adequate:** `Cache-Control: no-store` on all API routes is a FIRM invariant; API routes are dynamic `runtime=nodejs`, never statically cached — the "no PII cached at edge" rule (§A.2, ADR-0004) is correctly stated. `assertSameOrigin` on all POST routes gives browser-side CSRF defense; with no login/session cookies the CSRF residual is low.

### [MEDIUM] M-3 — Only HSTS is specified; no CSP / security-header baseline
- **Artifact:** `hld-lld.md` §B.5 (HSTS only); `system-context.md` (Zalo widget, GA4, Meta Pixel, chatbot embeds).
- **Weakness (CWE-1021 / CWE-693):** the site loads multiple third-party scripts and renders chatbot/CMS-sourced text. No Content-Security-Policy, `X-Content-Type-Options`, `Referrer-Policy`, or `frame-ancestors` is specified. Absent a CSP, any XSS (incl. unescaped chatbot/product output if markdown/HTML rendering is later chosen) has full script capability, and there is no allow-list constraining where PII-bearing pages may exfiltrate to.
- **Fix direction:** Define a security-header baseline in the design: strict CSP (script/connect/img/frame allow-lists incl. Zalo/GA4/Pixel/Cloudinary/Anthropic-none-client-side), `Referrer-Policy: no-referrer` (also satisfies "no PII in referrers"), `X-Content-Type-Options: nosniff`, and confirm chatbot/CMS output is escaped (React default) or sanitised if rendered as HTML/markdown.

---

## A07 — Identification & Authentication Failures

**Adequate:** CMS editor accounts require MFA (ADR-0001, §B.6); app uses a read-only CMS token. No buyer accounts exist (NG3), removing a large auth surface.

See **M-4** (A01) — the staff/DPO PII-access authentication path is undefined; tracked there.

---

## A08 — Software & Data Integrity Failures

### [MEDIUM] M-5 — Supply-chain (SCA) controls absent from the design (R-14)
- **Artifact:** none — no design coverage of npm dependency integrity.
- **Weakness (CWE-1104 / CWE-829):** an AI-assisted build may introduce vulnerable or hallucinated/typosquatted packages. The design mandates no lockfile discipline, SCA gate, or Subresource Integrity for the third-party `<script>` embeds (Zalo/GA4/Pixel).
- **Fix direction:** Require committed lockfiles, an SCA gate in CI (fail build on known-vuln deps; every AI-suggested lib passes this gate), automated dependency updates, and SRI/pinning for externally hosted scripts where the vendor supports it. Add as a P4/CI control.

### [MEDIUM] M-6 — Consent "append-only" is app-convention, not enforced at the store
- **Artifact:** `erd.md` §3; ADR-0005; §B.3.
- **Weakness (CWE-284):** the same scoped app credential that INSERTs consent presumably also holds UPDATE/DELETE on the table. "Immutable / append-only" therefore relies on application discipline; a compromised app path or malicious insider could rewrite the Art. 11 evidence, defeating its legal defensibility.
- **Fix direction:** Enforce immutability at the store: revoke UPDATE/DELETE on `CONSENT_RECORD` from the app role (INSERT/SELECT only), or use append-only/WORM semantics / triggers, so reproducibility is technically guaranteed, not just intended.

---

## A09 — Security Logging & Monitoring Failures

**Adequate:** audit log on lead-store read/write (TB-2, §B.6); DSR intake logged with owner + SLA clock; logs avoid raw PII (ip-hash/ua-hash, "no PII in URLs/query strings"); analytics server events carry no PII. This is a good baseline.

### [LOW] L-1 — Audit trail is defined but not monitored, and its own integrity/access is unspecified
- **Artifact:** §B.6 ("audit-logged"), NFR §6 (only cost alerting).
- **Weakness:** there is no alerting on anomalous PII access/export, no retention/tamper-evidence for the audit log itself, and no defined reviewer. Detection of misuse of legitimate lead-read access (a key SoD residual) is therefore unlikely.
- **Fix direction:** Specify audit-log retention, restrict who can read/alter it (not the same role it audits), and add alerting on bulk PII read/export and on DSR fulfilment actions.

---

## A10 — Server-Side Request Forgery

**Adequate — no significant SSRF surface.** `search_products` takes structured params (`menh`/`loai`/`giaMax`) against a **fixed** CMS host, not a user-supplied URL; `handoff_to_zalo` builds a deep-link string with no server fetch. All server-side egress targets (CMS, Cloudinary, Claude) are fixed, configured hosts.

### [LOW] L-2 — Keep the no-user-controlled-fetch property as a FIRM constraint
- **Fix direction:** State explicitly that no endpoint/tool may fetch a user- or model-supplied URL/host; if a future tool needs outbound fetch, it must use an allow-list. (The GROQ-injection risk on `search_products` args is tracked under M-2, not SSRF.)

---

## PCI boundary check

**Confirmed: NO card data path exists anywhere in the design.** No PAN/CVV/expiry/cardholder entity, column, request field, or response field appears in `erd.md`, `api-spec.yaml`, `class-diagram.md`, or any flow. All closing/payment is explicitly human-to-human on Zalo, off-platform (`system-context.md` §5, data-classification #15). `x-pii-policy` in the OpenAPI spec correctly asserts no card data / no PCI scope. The design does not accidentally reintroduce card data.

### [MEDIUM] M-7 — Free-text fields could inadvertently capture a PAN and egress it cross-border (R-15)
- **Artifact:** `api-spec.yaml` `ChatInput.messages.content`, `LeadInput.productInterest`, `DsrInput.details`; `hld-lld.md` §B.2 step 4 redaction.
- **Weakness:** while no field *solicits* card data, a visitor could paste a card number into chat/interest/details free text. That would then be stored and, for chat, transmitted to Anthropic — an inadvertent (small) reintroduction of card data into a system designed to have none, and a needless sensitive-data cross-border transfer.
- **Fix direction:** Extend the existing pre-storage redaction/flag pipeline (`redactSensitive`, `sensitive_flagged`) to detect and scrub **PAN-like patterns (Luhn-valid digit runs)** before persistence and, for chat, before the Anthropic call. Keeps the no-card-data invariant true in practice.

---

## Cross-border (PDPD Art. 25) — security-relevant data-flow notes

The egress mapping (`system-context.md` §4, `compliance-scope.md` Obligation B) is thorough and correct: Vercel (IP/logs), Cloudinary (media only), Anthropic (transcripts, possibly incidental sensitive). Security-relevant residuals (dossier/compliance-owned, flagged here as data-flow concerns, not new blockers):

### [LOW] L-3 — Undecided Postgres region + soft redaction may add/inflate cross-border egress
- **Artifact:** ADR-0002 (store region "evaluate VN/APAC if available"), §A.3 Flow 2 step 6 / erd §3 ("redact… where feasible").
- **Note:** the managed-Postgres choice (ADR-0002) is itself a likely cross-border processor whose region is undecided — it must be pinned and added to the Art. 25 dossier (already noted in ADR-0002, reinforced here). Pre-Anthropic minimization is currently "where feasible" (soft); to hold the R-04 minimization posture, define concretely what turn context is sent (minimal window, stripped identifiers) rather than leaving it optional. Also confirm the Anthropic DPA / no-training + retention terms referenced in ADR-0003 before first transfer.

---

## Findings index

| ID | Sev | OWASP | Title | Artifact | Risk map |
|---|---|---|---|---|---|
| H-1 | HIGH | A04 | Unauthenticated `/api/chat` denial-of-wallet + PII-egress amplification | hld-lld §B.2; nfr §4/§6 | R-13 (new), R-04 |
| H-2 | HIGH | A01 | DSR fulfilment lacks requester identity verification | hld-lld §B.4; api-spec `/dsr` | R-16 (new), R-01 |
| M-1 | MED | A02 | Low-entropy identifiers hashed without keyed HMAC/pepper | erd §1; hld-lld §B.3 | R-17 (new) |
| M-2 | MED | A03 | Prompt-injection / tool-abuse / GROQ-injection unaddressed | hld-lld §B.2 | R-09 (rel.) |
| M-3 | MED | A05 | No CSP / security-header baseline (HSTS only) | hld-lld §B.5 | — |
| M-4 | MED | A01/A07 | Staff/DPO PII-access auth path undefined | hld-lld §B.6 | R-18 (new) |
| M-5 | MED | A08 | Supply-chain / SCA controls absent from design | — | R-14 (new) |
| M-6 | MED | A08 | Consent immutability is app-convention, not DB-enforced | erd §3; ADR-0005 | R-17 (rel.) |
| M-7 | MED | PCI | Free-text fields could capture/egress a PAN | api-spec; hld-lld §B.2 | R-15 (new) |
| L-1 | LOW | A09 | Audit log not monitored; integrity/access unspecified | hld-lld §B.6 | — |
| L-2 | LOW | A10 | Keep no-user-controlled-fetch as a FIRM constraint | class-diagram §3 | — |
| L-3 | LOW | A25 | Postgres region undecided; soft pre-Anthropic minimization | ADR-0002; erd §3 | R-04 |

Proposed new risk IDs for the register: **R-13** (chat cost/egress abuse), **R-14** (supply chain / SCA), **R-15** (PAN-in-free-text), **R-16** (DSR identity verification), **R-17** (identifier-hash + consent-immutability hardening), **R-18** (staff PII-access auth).

Counts: **2 HIGH, 0 CRITICAL, 7 MEDIUM, 3 LOW.**
