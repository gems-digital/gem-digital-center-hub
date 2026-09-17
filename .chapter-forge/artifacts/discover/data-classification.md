# OmniGem Gallery — Data Classification (P1 Discovery & Requirements)

Status: DRAFT for Gate G0 — proposed classification, pending compliance-checker confirmation.
Note: all values below are field names / synthetic placeholders only. No real customer data is recorded in this or any project document.

## Classification legend
- **Public** — safe to expose to anyone (marketing/catalog content).
- **Internal** — business-operational, not customer-identifying, not for public release.
- **PII** — Personal Identifiable Information (identifies or can identify a natural person).
- **Sensitive PII** — PII with elevated handling requirements (e.g., special-category-like data).
- **Card data** — cardholder/payment data (PCI scope).

## Data element table

| # | Data element | Classification | Where it lives | Retention (proposed) | Notes |
|---|---|---|---|---|---|
| 1 | Product catalog (name, images, specs, origin, certification text) | Public | CMS (Sanity/Payload), CDN | Indefinite (business content) | No customer data involved. |
| 2 | Prices / "Contact for price" flags | Public | CMS | Indefinite | Business decision content, not customer data. |
| 3 | Blog/SEO articles | Public | CMS | Indefinite | — |
| 4 | Organization/brand info (address, hours, hotline) | Public | CMS / site footer | Indefinite | Same as published on Fanpage per `create-facebook-fanpage.md` §1. |
| 5 | Lead: full name | PII | Lead store (CRM/Google Sheet — TBD, PRD Open Question 7) | Proposed 24 months from last contact, pending Product Owner/legal input | Directly identifying. |
| 6 | Lead: phone number / Zalo ID | PII | Lead store | Same as #5 | Directly identifying, also a communication channel — handle as sensitive contact data. |
| 7 | Lead: **date of birth** | **Basic personal data (Decree 13/2023 Art. 2(3))** — confirmed, not sensitive | Lead store, chatbot conversation logs | Same as #5 | Collected for feng-shui/ngũ-hành matching, not identity verification. **Ruling (compliance-checker confirmed):** date of birth is explicitly enumerated in the Art. 2(3) basic-personal-data list — it does not become "sensitive personal data" merely by being combined with name+phone (that raises a security-controls concern, not a classification change). Consent is still required regardless of this classification, because PDPD requires consent for the processing of ALL personal data — there is no legitimate-interest carve-out. |
| 8 | Lead: product interest / stated need | PII (contextual) | Lead store | Same as #5 | Low sensitivity alone, but part of the PII record. |
| 9 | Chatbot conversation transcripts | PII — **potential Art. 2(4) sensitive-data exposure via free text** | Chat backend / lead store / Claude API request logs (vendor-side, cross-border — see `compliance-scope.md`) | Proposed same as #5, plus vendor retention (see Claude API terms — verify at Design phase) | May contain name, DOB, phone if volunteered in free text — treat whole transcript as PII. **Additional handling note:** because the chat field is free text, a visitor could incidentally volunteer Art. 2(4) sensitive personal data (e.g., health condition, religion/belief, financial/bank-account detail) even though the bot never solicits it — and this transcript crosses the border to Anthropic (Claude API). Chatbot system prompt and UI copy must (a) never ask for sensitive-category data, (b) apply data-minimization (only request what's needed for feng-shui matching: DOB + optionally birth time), and (c) flag/redact sensitive volunteered content before storage where feasible. |
| 10 | Contact form submissions (name, phone/Zalo, DOB, interest) | PII | Lead store, form backend/API logs | Same as #5 | Same fields as #5–8, submitted via a different channel. |
| 11 | GA4 client ID / cookies | PII (online identifier) | Google Analytics (GA4) | Per GA4 default retention settings (to be configured) | Treated as personal data under most modern privacy regimes even though not directly identifying by itself. |
| 12 | Meta Pixel identifiers / cookies | PII (online identifier) | Meta | Per Meta default retention | Same reasoning as #11. |
| 13 | IP address (server/request logs) | PII (online identifier) | Vercel/hosting logs | Per hosting provider default | Standard web server log data; still personal data under PDPD's broad definition. |
| 14 | Zalo OA conversation data | PII | Zalo platform (vendor-side) | Governed by Zalo's own retention/terms | Outside our direct storage but our funnel routes leads here — note as a data flow, not our data store. |
| 15 | **Card / payment data (PAN, CVV, expiry, cardholder name)** | **Card data — NOT IN SCOPE** | N/A | N/A | **Explicitly out of scope for this website.** All payment negotiation and settlement (deposit, COD) happens off-platform via Zalo with a human, per research doc §1 and §6c. **If a future phase adds an on-web deposit/bank-transfer flow (e.g., collecting a bank account number for a deposit), that escalates PDPD *sensitive* personal data scope (Art. 2(4) — financial/banking-account information), and re-triggers Gate G0/G1 review on that basis. PCI-DSS is triggered only if an actual payment card (PAN) is captured — these are two distinct escalation paths and should not be conflated.** |
| 16 | Staff/admin CMS login credentials | Internal / Sensitive (credential) | CMS auth provider | Per CMS provider policy | Standard operational credential, not customer PII, but still requires secret-management discipline (see security rules). |

## Explicit statements for Gate G0

1. **Card/payment data: NOT in scope.** The website never collects, transmits, or stores cardholder data. All closing/payment happens on Zalo between the customer and a human staff member (research doc §1: "không chốt đơn trên web"; §6c: "Chốt đơn qua Zalo... cọc, ship COD"). This is the requirements-analyst's proposed position — compliance-checker must confirm no future-phase creep (e.g., an "online deposit" feature) reintroduces card scope.
2. **PII in scope: yes.** Name, phone/Zalo ID, date of birth, product interest, and chat transcripts are collected via the lead form and chatbot. This is the primary compliance surface for this project.
3. **Governing regime (proposed):** Vietnam's **Decree 13/2023/NĐ-CP (Personal Data Protection Decree — PDPD)** governs all PII listed above. This is a proposed position for compliance-checker to confirm, not a final legal determination.
4. **DOB sensitivity — RESOLVED:** date of birth (#7) is **basic personal data under Decree 13/2023 Art. 2(3)**, not sensitive personal data, and does not become sensitive by combination with name/phone. Consent is still mandatory for its processing (PDPD requires consent for all personal data, basic or sensitive). See item 9 for the separate, narrower risk that free-text chatbot fields could incidentally capture actual Art. 2(4) sensitive data.
5. **SBV / banking-grade PCI controls:** proposed as **out of scope** for this website — it is not a bank, payment service provider, or card acquirer/processor. This is a proposed position for compliance-checker to confirm (see `compliance-scope.md`).
6. **Cross-border transfer flag:** Vercel hosting, Cloudinary image storage, and the Claude API (Anthropic) may process data on infrastructure located outside Vietnam. PDPD has cross-border data transfer provisions; this needs explicit compliance-checker review before Design-phase implementation locks in vendor choices (see `compliance-scope.md` and `risk-register.md` R-04).
