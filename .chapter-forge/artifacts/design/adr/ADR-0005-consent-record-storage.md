# ADR-0005 — Consent-record storage (Art. 11 reproducibility)

Status: PROPOSED (Gate G1). Deciders: Architect + Compliance. Implements US-H2; addresses R-01.

## Context
PDPD Art. 11 requires **express, recorded** consent for processing ALL personal data (basic or sensitive; no legitimate-interest carve-out — data-classification item 7). A UI checkbox whose state is not durably logged is **not** sufficient (US-H2 AC1). We need a record that (a) proves what the subject was shown, (b) when, (c) for what purpose, and (d) is retrievable as evidence (US-H2 AC2), including for DOB collected in the lead form and the chatbot.

## Decision
Persist an **append-only `ConsentRecord`** in the same encrypted lead store (ADR-0002), written in the **same DB transaction** as the Lead/handoff it authorizes.
- Each record pins `policyVersion` → a **versioned, immutable `PrivacyPolicyVersion`** (content + hash + published_at), so the exact notice text shown is reconstructable.
- Fields: `policyVersion, purpose, method (web_form|chatbot), subject_ref_hash, context_hash (ip/ua hashed), captured_at, event_type (granted|withdrawn)`.
- **Immutability:** consent is never updated in place. A withdrawal (US-H3) inserts a new `event_type=withdrawn` record, preserving the full history. **Enforced at the DB layer (review M-6):** the app role has no UPDATE/DELETE on `ConsentRecord` — append-only is a database grant, not just application convention.
- Client sends `consent:{policyVersion, granted:true}`; the server **rejects** any submission where `granted !== true` (no pre-ticked/implied consent).
- **Chatbot path is symmetric (review H-3):** the chatbot is not exempt. On the **first PII-bearing chat turn**, a **session-scoped `ConsentRecord`** (`method=chatbot`, `session_id` set, `lead_id` initially null, later linked on handoff) is written **before** any transcript persistence or DOB use — and before the cross-border call. `ChatInput.consent` is therefore **required**, not optional; a turn without granted consent processes and stores nothing.
- **Subject-reference hashing** in the record uses **keyed HMAC-SHA256 (review M-1)**, not bare SHA-256, so the low-entropy identifier space is not brute-forceable from a store leak.

## Consequences
- (+) Fully reproducible consent evidence for an auditor or data subject (Art. 11, US-H2 AC2); atomicity guarantees no lead exists without its consent record.
- (+) Versioned policy + hash means later policy edits don't retroactively alter what past subjects agreed to.
- (+) Feeds the Art. 24 DPIA dossier directly (demonstrable lawful basis).
- (−) Requires publishing the Privacy Policy as versioned content and storing hashes; small added complexity in the publish flow (ADR-0004 versioned Privacy page).
- (−) Append-only growth; bounded by the same retention/purge policy as leads (respecting legal-retention overrides, US-H3 AC3).
