# OmniGem Gallery — STRIDE Threat Model (P2 Design, Gate G1)

Status: DRAFT for Gate G1. Author: **threat-modeler** (Maker). Approvers: **Security + Architect** (Checker).
Scope: single Next.js app on Vercel + Sanity CMS + Anthropic Claude API + Zalo funnel + managed-Postgres lead store. **No on-web payments / no card data / no PCI scope** (data-classification #15). PII regime: Vietnam **PDPD (Decree 13/2023)** — carried obligations Art. 11 (consent record), Art. 24 (DPIA), Art. 25 (cross-border).
Inputs: `system-context.md` (TB-1…TB-5), `hld-lld.md` (§A.3 flows, §B.5 encryption, §B.6 SoD), `api-spec.yaml`, `erd.md`, `class-diagram.md`, `adr/ADR-0001..0005`, `discover/data-classification.md`, `discover/risk-register.md`.
All examples are **SYNTHETIC**. This file is analysis-only; the architect's artifacts are unmodified.

## Legend

- **Severity**: CRITICAL / HIGH / MEDIUM / LOW.
- **Status**: `mitigated` (control present in design) · `partial` (control present but incomplete/defective) · `gap` (no control) · `accepted` (residual, documented).
- Trust boundaries per `system-context.md §3`: **TB-1** browser→API, **TB-2** API→Lead Store, **TB-3** app↔CMS, **TB-4** app/browser→cross-border vendors, **TB-5** staff→Lead Store.

---

## 1. Sequence diagrams (the 3 sensitive flows) with threat/boundary annotations

### Flow 1 — Lead-form submission (US-C1, US-H2; R-01, R-03)

```mermaid
sequenceDiagram
    autonumber
    actor V as Visitor Browser (VN)
    participant P as Next.js /contact (SSG)
    participant L as /api/lead (server, no-store)
    participant CS as ConsentService
    participant LS as LeadService
    participant DB as Lead Store (Postgres, enc-at-rest)
    participant AN as Analytics (server event)

    Note over V,P: TB-1 crossing — untrusted input (name/phone/DOB PII)
    V->>P: load contact page + current PrivacyPolicyVersion
    P-->>V: consent gate (unticked by default)
    V->>L: POST {name,phone/zalo,dob,interest,consent:{version,granted:true}} (TLS)
    Note right of L: [S] no auth by design · subject_ref UNVERIFIED (R-18)<br/>[T/D] assertSameOrigin + rateLimit — origin header spoofable by non-browser (F-08)<br/>[T] Zod schema validation (fail-fast); reject if granted!=true
    L->>CS: db.tx begin — record(policyVersion,purpose,method,ctx-hash)
    Note over CS,DB: TB-2 crossing — PII write
    CS->>DB: INSERT ConsentRecord (append-only, Art.11)
    L->>LS: create(lead, source=lead_form, consentId)
    LS->>DB: INSERT Lead (PII-enc AES-256)
    Note right of DB: [R] atomic tx → no lead w/o consent (GOOD, mitigated)<br/>[I] PII-enc columns + KMS keys (R-03 mitigated)<br/>[E] app credential blast radius if over-scoped (F-06)
    DB-->>L: commit {leadId}
    L->>AN: serverEvent(submit_lead_form,{leadId}) — no PII
    L-->>V: 201 {accepted, leadId} (no-store)
    Note over V,L: on DB fail → 503 + Zalo CTA (never partial-commit)
```

### Flow 2 — Chatbot conversation + handoff (US-D1/D2/D3; R-04, R-09) — highest-risk flow

```mermaid
sequenceDiagram
    autonumber
    actor V as Visitor Browser (VN)
    participant C as /api/chat (server, no-store)
    participant CB as ChatbotService (tool loop)
    participant CL as ClaudeApiAdapter
    participant AP as ⚠️ Anthropic Claude API (OUTSIDE VN)
    participant PQ as ProductQueryService
    participant CMS as Sanity CMS (Public catalog/FAQ)
    participant Z as ZaloLinkBuilder
    participant LS as LeadService/ConsentService
    participant DB as Lead Store (enc-at-rest)

    Note over V,C: TB-1 crossing — free-text may carry volunteered PII + incidental Art.2(4) sensitive data (item 9)
    V->>C: POST {sessionId, messages[], consent?} (TLS)
    Note right of C: [S] no auth; sessionId client-supplied & rotatable (R-13)<br/>[D] rateLimit keyed on spoofable IP/sessionId → cost-exhaustion (R-13)<br/>[T] client supplies FULL messages[] incl. prior 'assistant' turns → forgeable history (R-14)<br/>[compliance] consent is OPTIONAL in ChatInput → DOB may arrive w/o consent (R-17)
    C->>CB: turn({sessionId,messages,consent})
    CB->>CL: createMessage(system-guardrail, messages, TOOLS)
    Note over CL,AP: TB-4 CROSS-BORDER (Art.25) — raw turn text (incl. any volunteered sensitive data)<br/>⚠️ [I] data reaches Anthropic BEFORE any redaction → redaction-before-STORAGE cannot protect this hop (R-16 CRITICAL)
    CL->>AP: messages + system prompt
    AP-->>CL: stop_reason=tool_use | end_turn
    alt tool_use = search_products
        CB->>PQ: search(menh,loai,giaMax)
        PQ->>CMS: fetch live catalog (TB-3, Public only)
        CMS-->>PQ: products (no hallucination — US-D1 AC2)
        PQ-->>CB: tool_result
        CB->>CL: loop (bounded maxIterations)
    else tool_use = handoff_to_zalo
        CB->>Z: build(productCode) deep-link
        CB->>LS: create(source=chatbot) + ConsentService IF consent captured
        Note right of LS: [compliance] consent recorded only HERE → non-atomic vs turn-1 DOB (R-17)<br/>[D/integrity] injection-forced handoff spam → fake leads (R-14)
        LS->>DB: INSERT Lead(+ConsentRecord) — TB-2
    end
    CB->>CB: redactSensitive(messages) — best-effort
    CB->>DB: persist ChatTranscript (PII-enc, sensitive_flagged) — TB-2
    Note right of DB: [I] redaction here protects local store only, NOT the Anthropic hop (R-16)
    CB-->>V: reply / handoff / {fallback:true, zaloUrl} on API error
```

### Flow 3 — Analytics event capture (US-G1; data-classification 11/12)

```mermaid
sequenceDiagram
    autonumber
    actor V as Visitor Browser (VN)
    participant CGate as Consent gate (client JS)
    participant Tags as GA4 / Meta Pixel (browser tags)
    participant COL as /api/collect (optional proxy, server)
    participant G as ⚠️ GA4 / Meta (OUTSIDE VN)

    Note over V,CGate: consent gate is CLIENT-side → bypassable, but only affects the user's own tracking (F-11 LOW)
    V->>CGate: interact (view_product, click_zalo, submit_lead_form, chatbot_handoff)
    alt analytics consent granted
        CGate->>Tags: load tags + dispatch event (event name + productId only, NO name/phone/DOB)
        Note over Tags,G: TB-4 CROSS-BORDER — online-identifier PII (cookie/clientId, IP) → third-party recipients (Art.25, Privacy Policy disclosed)
        Tags->>G: hit (online identifier)
    else no consent
        CGate-->>V: tags NOT loaded (consent-gated — US-H2)
    end
    opt hardening: /api/collect first-party relay
        CGate->>COL: POST {event, productId, consentAnalytics:true}
        Note right of COL: [T] event spoofable → metric skew (F-10 LOW); drop if consentAnalytics!=true
        COL->>G: server-side relay (reduces client identifier leakage, server consent-mode)
    end
```

---

## 2. STRIDE threat tables (per element / flow)

### 2.1 TB-1 — Visitor browser → API routes (`/api/lead`, `/api/chat`, `/api/dsr`, `/api/collect`)

| STRIDE | Threat (concrete) | Severity | Existing mitigation / status |
|---|---|---|---|
| **S** | No authentication on any public POST (by design — lead-gen, no accounts NG3). `subject_ref` (phone/zalo) is never verified → a lead/consent record can be created attributing consent to a person who never gave it (F-05/R-18). | MEDIUM | `partial` — TLS + schema validation only; identity of the data subject is unverified. Acceptable for lead intake, but weakens consent-evidence quality and enables impersonation. |
| **T** | `/api/chat` trusts the **entire client-supplied `messages[]` array including prior `assistant` turns**; attacker rewrites history / injects instructions to override the guardrail system prompt (F-02/R-14). | HIGH | `partial` — guardrail system prompt + bounded iterations exist, but server does not hold authoritative conversation state. |
| **R** | Weak non-repudiation on unverified submissions — a subject can deny a lead was theirs; only `context_hash` (ip/ua) ties it. | LOW | `partial` — ConsentRecord append-only + hashed context (good for what it is). |
| **I** | Free-text `messages.content` (≤4000×50) may carry volunteered PII and **incidental Art. 2(4) sensitive data** that then crosses TB-4 (F-01/R-16). | CRITICAL | `partial` — see §3(a); system prompt never solicits sensitive data, but ingestion is unavoidable. |
| **D** | `/api/chat` unauthenticated + rate-limit keyed on **client-supplied `sessionId`** and rotatable IP; each call fans out to multiple Claude calls (tool loop) → **LLM billing exhaustion / financial DoS** (F-03/R-13). Also junk lead/DSR flooding. | HIGH | `partial` — `rateLimit(req,"chat")` + prompt caching + per-session cap exist, but keying is spoofable and there is no global spend circuit-breaker / bot challenge. |
| **E** | Prompt-injection can force `handoff_to_zalo` (model-driven side effect creating a Lead) → privilege abuse via the model (F-02/R-14). | HIGH | `partial` — tool is bounded, but tool invocation is not authorization-gated beyond the model's judgement. |
| **CSRF/origin** | `assertSameOrigin` relies on Origin/Referer header — bypassable by any non-browser client; the design conflates CSRF with abuse control (F-08). | LOW | `partial` — adequate as CSRF control for a **session-less** endpoint (no cookie to ride), but NOT an anti-automation control; abuse control must come from R-13 measures. |

### 2.2 TB-2 — API → Lead Store (Lead, ConsentRecord, ChatTranscript, DSRRequest)

| STRIDE | Threat | Severity | Existing mitigation / status |
|---|---|---|---|
| **S** | App impersonation to DB if the scoped credential leaks (in bundle/log). | LOW | `mitigated` — server-only env, secrets in Vercel store, no `NEXT_PUBLIC_` (B.5). |
| **T** | Direct SQL from route handlers bypassing repository invariants. | LOW | `mitigated` — writes go through `LeadRepository`; "no raw SQL from handlers" invariant (class-diagram §2). |
| **R** | Denial of who read/changed lead data. | LOW | `mitigated` — audit log on read/write (TB-2, B.6). |
| **I** | PII at rest exposure on store compromise. | MEDIUM | `mitigated` — AES-256 storage + application-layer/column enc for PII-enc fields + KMS (B.5, retires R-03). |
| **D** | Junk-lead / junk-DSR flooding fills store and **DSR flood weaponizes the ≤72h ack SLA** against the DPO (F-04/R-18). | MEDIUM | `gap` — rate limit is the only control; no dedupe / anomaly detection described. |
| **E** | Single over-privileged app DB credential = full-PII blast radius if the serverless runtime is compromised (dep/SSRF). Intake only needs INSERT (+ narrow `findForContact` SELECT), not bulk SELECT (F-06/R-19). | MEDIUM | `partial` — "scoped credential" stated but scope not decomposed; least-privilege per-operation not specified. |

### 2.3 TB-4 — Cross-border egress (Anthropic, Vercel, Cloudinary, GA4, Meta)

| STRIDE | Threat | Severity | Existing mitigation / status |
|---|---|---|---|
| **I** | **Chat transcript + incidental Art. 2(4) sensitive data transferred to Anthropic outside VN** — the transfer *is* the processing and happens before redaction (F-01/R-16). | CRITICAL | `partial` — see §3(a). Minimization + no-solicitation prompt; DPA/no-training/zero-retention **to be confirmed** (ADR-0003). |
| **I** | Online-identifier PII (clientId/IP) to GA4/Meta abroad. | MEDIUM | `mitigated` — consent-gated tags + Privacy-Policy disclosure; `/api/collect` optional hardening. |
| **I** | PII in Vercel serverless **function logs** (e.g. accidental `console.log` of request body) leaves VN in logs. | MEDIUM | `partial` — "no PII in URLs/query" + ip/ua hashing stated; body-logging discipline not asserted. |
| **R** | Every egress must be enumerable for the Art. 25 dossier. | HIGH→tracked | `mitigated` (as design input) — `system-context.md §4` enumerates all egress (R-04/R-12 dossiers owned by Compliance). |
| **I** | Cloudinary receives media only, no customer data. | LOW | `mitigated` — "no customer data ever sent to Cloudinary." |

### 2.4 TB-3 / TB-5 — CMS and staff access (SoD)

| STRIDE | Threat | Severity | Existing mitigation / status |
|---|---|---|---|
| **E** | Content editor reaches PII / lead store. | LOW | `mitigated` — SoD B.6: editors have `none` on Lead Store; CMS holds no PII (ADR-0001). |
| **E** | Deployer reads lead PII. | LOW | `mitigated` — SoD B.6: deployer manages secrets but `none` direct data; no single human holds edit+export+deploy. |
| **S** | CMS editor account takeover → poison public catalog/FAQ → feeds RAG false info (links R-09/R-10). | MEDIUM | `partial` — editor MFA + read-only app token stated; RAG trusts CMS as source of truth (blast radius = content integrity). |
| **I** | CMS read token leak → only Public content exposed. | LOW | `mitigated` — read-only token; no PII in CMS. |

### 2.5 Consent subsystem (Art. 11) — cross-cutting

| STRIDE | Threat | Severity | Existing mitigation / status |
|---|---|---|---|
| **T/R** | Consent tampering / retroactive alteration of what a subject agreed to. | LOW | `mitigated` — append-only ConsentRecord; withdrawal = new row; `policyVersion` pins immutable hashed notice (ADR-0005). |
| **compliance** | **Chatbot path processes DOB/PII (persist + cross-border) before any ConsentRecord exists**; consent recorded only at handoff and only "if captured"; `ChatInput.consent` is optional → Art. 11 atomicity broken for the chat channel (F-07/R-17). | HIGH | `partial` — see §3(b). Lead-form path IS atomic (good); chatbot path is not. |

### 2.6 DSR subsystem (Art. 9/10)

| STRIDE | Threat | Severity | Existing mitigation / status |
|---|---|---|---|
| **S/I/D** | `/api/dsr` accepts erasure/access/rectify for **any** phone/zalo/email with **no requestor identity verification** → spoofed erasure (destructive integrity/DoS on a real subject) or access-disclosure (F-09/R-15). | HIGH | `partial` — see §3(d-bis). Fulfilment is a "controlled back-office action" with an owner (mitigates auto-disclosure), but no verification step is specified before fulfilment. |
| **R** | Denial that a DSR was actioned. | LOW | `mitigated` — DSRRequest logged, owner + `ackBy` on intake. |

---

## 3. Stress-test of the architect's flagged design risks

### (a) Cross-border chat transcripts carrying incidental Art. 2(4) sensitive data — is redaction-before-storage actually feasible? → **F-01 / R-16 (CRITICAL)**

**Verdict: the stated control is mis-ordered and cannot achieve its compliance goal.**

- The design says: "Redact/flag incidentally-volunteered sensitive text **before storage** where feasible" (`hld-lld.md` A.3 Flow-2 step 6, B.2 step 4; `erd.md` §3). But the sequence is: visitor free-text → **sent to Anthropic to generate the reply** → *then* stored. The **cross-border transfer (TB-4 / Art. 25) happens on the inbound Claude call, before any storage-time redaction runs.** Redaction-before-storage therefore only sanitizes the *local* `ChatTranscript`; it does **nothing** for the Anthropic hop, which is the actual Art. 25 / Art. 2(4) exposure.
- **Feasibility of redaction itself:** unreliable for this data class. Health condition, religious/belief statements, and financial/bank-account details in Vietnamese free text are **semantic, not pattern-based** — regex/Luhn/NER catch structured tokens (phone, card-like numbers) but miss "tôi bị bệnh tim" (health) or belief/financial context. Any redactor will have material false-negatives, so "flag" ≠ "prevented."
- **What breaks:** OmniGem transfers special-category personal data out of Vietnam without a lawful, minimized basis — the exact scenario data-classification item 9 and R-04 warn about, now concretely un-mitigated on the transfer hop.
- **Fix direction (for the architect):**
  1. Move redaction/classification **before** the `ClaudeApiAdapter.createMessage` call (input-side), not before storage — mask or block on detected sensitive spans, or refuse the turn with a UI nudge.
  2. Bind an Anthropic **DPA with no-training + zero/short retention** as a hard prerequisite (ADR-0003 lists this as "confirm" — promote to blocking).
  3. Hard UI copy + system-prompt steering that actively deflects sensitive disclosures (not just "never solicit").
  4. Explicitly document the **residual** (best-effort redaction cannot be perfect) as an accepted risk in the Art. 25 dossier (R-04) and DPIA (R-12). Status: `partial → gap on the transfer hop`.

### (b) Consent atomicity / reproducibility for Art. 11, incl. the chatbot DOB path → **F-07 / R-17 (HIGH)**

- **Lead-form path: PASS.** Flow-1 writes ConsentRecord + Lead in one `db.tx` (B.1), `granted:true` enforced, `policyVersion` pinned to an immutable hashed notice, append-only (ADR-0005). This is a solid Art. 11 artifact.
- **Chatbot path: FAIL on atomicity.** In Flow-2 a visitor can volunteer DOB (and more) at **turn 1** (the `api-spec.yaml` `/chat` synthetic example literally shows `"I was born 1990-01-01…"`). That DOB is (i) transmitted cross-border and (ii) persisted in `ChatTranscript` — but a **ConsentRecord is only created at `handoff_to_zalo`, and only "if consent captured."** `ChatInput.consent` is **optional** (`api-spec.yaml`: `required:[sessionId, messages]`). So PII/DOB can be processed with **no consent record at all** if the session never reaches handoff — and even when it does, consent is recorded *after* processing, not atomically with it.
- **What breaks:** Art. 11 (consent required for ALL personal data incl. DOB — data-classification #7 ruling) is not reproducibly satisfied for the chat channel; DPIA lawful-basis evidence has a hole.
- **Fix direction:** make `consent` **required** in `ChatInput`; write a session-scoped ConsentRecord (`method:"chatbot"`) at the **first PII-bearing turn / session start**, before the transcript is persisted or DOB is processed — mirror the lead-form atomicity. Gate any DOB use on that record (the micro-notice described in A.3 Flow-2 step 1 must produce a *record*, not just a UI notice).

### (c) `/api/chat` cost-exhaustion + prompt-injection abuse on an unauthenticated public endpoint → **F-03 / R-13 (HIGH)** and **F-02 / R-14 (HIGH)**

- **Cost-exhaustion (R-13):** endpoint is public/no-auth; `rateLimit(req,"chat")` is keyed on a **client-supplied `sessionId`** (attacker rotates it freely) and IP (rotatable via proxies). Each accepted request drives a **multi-call Claude tool loop** with up to **50 messages × 4000 chars** of context — a high per-request cost multiplier. There is no global spend circuit-breaker described. An attacker (or a botnet) converts free requests into unbounded Anthropic billing → financial DoS + potential service suspension.
  - **Fix direction:** bot challenge (Cloudflare Turnstile / proof-of-work) before the first Claude call; server-authoritative session tokens (HMAC-signed, not raw client string); cap tokens & message count/history length server-side; a **hard monthly spend circuit-breaker** that fails to the Zalo CTA; per-IP + fingerprint limiting independent of client sessionId.
- **Prompt-injection / forged history (R-14):** the server trusts the **entire client `messages[]` including prior `assistant` turns**. An attacker can (i) forge assistant turns to "put words in the bot's mouth" and jailbreak the guardrail prompt, (ii) extract/override the system prompt, (iii) coerce false authenticity/return-policy/price statements (ties R-05, R-09, R-10), or (iv) force `handoff_to_zalo` to spam the lead store with fake leads (integrity/DoS).
  - **Fix direction:** keep **server-side authoritative conversation state** keyed by the signed session (never trust client `assistant` content); re-inject guardrails each turn; validate/whitelist tool outputs; rate-limit and (optionally) confirm handoff lead creation; add injection heuristics + output checks. Note the guardrail system prompt alone is a **defense-in-depth**, not a boundary — LLM guardrails are bypassable.

### (d) CSRF / origin adequacy on public POST routes with no login → **F-08 (LOW)** + DSR gap **F-09 / R-15 (HIGH)**

- **CSRF adequacy:** these endpoints are **session-less** (no cookies, no login — NG3). Classic CSRF (riding an authenticated ambient session) has **little impact** here because there is no session to abuse. `assertSameOrigin` is therefore *adequate for CSRF* but is trivially bypassed by non-browser clients (curl) — so it must **not** be treated as an abuse/automation control. The design lightly conflates the two; the real anti-abuse controls are the R-13 measures. Rated **LOW** as a CSRF matter, but flagged so the revision does not lean on origin-check for bot defense.
- **DSR identity verification (R-15, HIGH):** the more serious sibling. `/api/dsr` (and `DsrService.fulfil`) has **no requestor-identity verification**. Anyone can POST `requestType:"erasure"` with a victim's phone/zalo/email. The human back-office gate (owner + legal-retention override, US-H3 AC3) reduces auto-disclosure risk on *access*, but **unverified erasure/rectify is a destructive-integrity attack** on a real subject's data, and DSR flooding also weaponizes the ≤72h ack SLA (R-18).
  - **Fix direction:** add an identity-verification step **before fulfilment** — e.g., OTP challenge to the phone/Zalo of record, or a match-challenge against stored fields — and document it in the DSR SOP. Intake may stay open; verification gates the action.

---

## 4. Risk mapping and new risks

### 4.1 Findings mapped to existing risks (R-01…R-12)

| Finding | Title | Maps to | Note |
|---|---|---|---|
| F-01 | Cross-border sensitive-data exposure to Anthropic (redaction mis-ordered) | **R-04**, R-12 | Concretizes R-04 as a specific ordering defect → new **R-16**. |
| F-07 | Chatbot consent non-atomic / DOB before consent | **R-01**, R-02 | New **R-17**. |
| F-02 | Prompt injection / forged assistant history → false claims, handoff spam | **R-09**, R-05, R-10 | Security vector beyond hallucination → new **R-14**. |
| F-06 | App DB credential blast radius | **R-03** | Extends R-03 (store security) → new **R-19**. |
| Consent/store controls | Atomic consent, enc-at-rest, SoD | R-01, R-03 mitigated | Design already addresses (lead-form path). |
| Egress enumeration | All cross-border hops listed | R-04, R-12 | Design input satisfied. |

### 4.2 New risks raised (R-13+)

| ID | Risk | Category | Likelihood | Impact | Priority | Fix owner |
|---|---|---|---|---|---|---|
| **R-13** | `/api/chat` unauthenticated LLM cost-exhaustion / financial DoS (spoofable session/IP keying, multi-call tool loop, no spend circuit-breaker) | Security/Cost | High | High | **HIGH** | Architect |
| **R-14** | Chatbot prompt-injection + client-forged assistant turns → guardrail bypass, false authenticity/price claims, handoff-spam fake leads | Security/Business | High | High | **HIGH** | Architect |
| **R-15** | DSR endpoint has no requestor identity verification before fulfilment → spoofed erasure (destructive) / access-disclosure (Art. 9/10) | Compliance/Security | Medium | High | **HIGH** | Architect + Compliance |
| **R-16** | Cross-border redaction mis-ordering — Art. 2(4) sensitive data reaches Anthropic before redaction; "redact-before-storage" cannot protect the transfer hop (Art. 25) | Compliance | High | High | **CRITICAL** | Architect + Compliance |
| **R-17** | Chatbot consent non-atomic — DOB/PII processed + persisted + sent cross-border before/without a ConsentRecord; `ChatInput.consent` optional (Art. 11) | Compliance | High | High | **HIGH** | Architect + Compliance |
| **R-18** | Unverified `subject_ref` → weak/false consent evidence + lead-store pollution; DSR flooding weaponizes ≤72h ack SLA | Security/Ops | Medium | Medium | **MEDIUM** | Architect |
| **R-19** | Single over-privileged app DB credential = full-PII blast radius on serverless compromise (intake needs INSERT + narrow SELECT, not bulk SELECT) | Security | Medium | Medium | **MEDIUM** | Architect |

### 4.3 MEDIUM / LOW findings (summary)

- **MEDIUM (5):** unverified data-subject identity on lead intake (F-05/R-18); junk-lead & DSR-flood DoS on store/SLA (F-04/R-18); PII in Vercel function logs (body-logging discipline unasserted); over-privileged app DB credential (F-06/R-19); CMS editor takeover poisoning RAG content (links R-09/R-10).
- **LOW (5):** weak non-repudiation on unverified submissions; origin-header CSRF check bypassable / conflated with abuse control (F-08); client-side analytics consent gate bypassable (affects only the user's own tracking, F-11); spoofable `/api/collect` events skew metrics (F-10); duplicate/no-dedupe leads on repeated handoff.

---

## 5. Convergence note (Gate G1)

The lead-form + consent + SoD + encryption + egress-enumeration parts of the design are **sound**. The concentration of un/under-mitigated risk is the **chatbot flow** (Flow 2) and the **DSR flow**: one CRITICAL (R-16) and four HIGH (R-13, R-14, R-15, R-17) findings there must go back to the architect before G1 sign-off. All are concrete and actionable per §3. No card/PCI findings (out of scope by design). No real PII used anywhere in this document.
