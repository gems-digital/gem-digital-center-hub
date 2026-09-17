# ADR-0003 — Chatbot: self-hosted Claude API vs no-code (Coze/Botpress)

Status: PROPOSED (Gate G1). Deciders: Architect + Product Owner. Resolves PRD Open Question 4.

## Context
The chatbot must: consult on ngũ-hành from DOB (US-D1), answer trust/FAQ consistently with Fanpage/web copy (US-D2, R-10), ground answers in live CMS catalog via RAG with **no hallucinated inventory** (US-D1 AC2, R-09), and perform a **mandatory human handoff** via a `handoff_to_zalo()` tool that writes a lead (US-D3). It also handles PII free-text that **crosses the border** (Art. 25, data-classification item 9), so we must control the system prompt, data-minimization, redaction, and where transcripts are stored.

## Decision
Build the chatbot on the **self-hosted Anthropic Claude API** via our own `/api/chat` route and `ChatbotService` orchestration (tool-calling loop with `search_products` + `handoff_to_zalo`).
- Full control of the **system prompt guardrails** (never solicit Art. 2(4) sensitive data; enforce minimization; mandatory handoff on purchase intent) — a compliance requirement we cannot guarantee on a no-code platform.
- We own the **RAG grounding** against live CMS data and the **transcript storage** (redaction + retention + encryption in our own store), rather than handing PII to an additional third-party bot vendor.
- Direct integration with our `LeadService` so chatbot handoffs land in the single lead source of truth (US-D3 AC2).
- Prompt caching + **server-enforced** per-session/turn caps + a **hard global spend circuit-breaker** to bound cost (NFR-6, review H-1).
- **Input-side sensitive-data scrubber (review C-1):** a `SensitiveInputScrubber` runs **before** every cross-border `ClaudeApiAdapter.createMessage` call, redacting Art. 2(4) sensitive categories and stripping Luhn-valid card-number strings so nothing sensitive or card-shaped leaves Vietnam (preserves the no-card-data invariant). Storage-time redaction alone was rejected as it sanitizes only the local DB, not the transfer.
- **BLOCKING PREREQUISITE (review C-1):** a signed **Anthropic Data Processing Addendum with no-training / zero-retention** terms must be in place before go-live; it is referenced by the Art. 24 DPIA and Art. 25 cross-border dossiers. No production chat traffic until confirmed.
- **Server-authoritative conversation state (review H-2):** the client submits only the new user message + an HMAC-signed session token; the server holds prior turns and re-injects the guardrail system prompt each turn, so a client cannot forge assistant history to bypass guardrails or force spurious handoffs.

## Consequences
- (+) Meets the hard compliance/guardrail requirements; minimizes the number of cross-border processors (only Anthropic, already listed in the Art. 25 dossier — avoids adding Coze/Botpress as extra recipients).
- (+) Consistent copy via CMS-sourced FAQ (retires R-10); grounded RAG mitigates R-09.
- (−) More engineering than a no-code builder (tool loop, RAG, transcript store) — accepted for control.
- (−) Availability depends on Claude API; mitigated by graceful degradation to a Zalo handoff CTA (NFR-4, R-06).
- Claude API key in Vercel secret store only (HLD B.5). Confirm Anthropic data-retention/no-training terms in the DPA for the Art. 25 dossier.
