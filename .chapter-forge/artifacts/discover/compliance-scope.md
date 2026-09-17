# OmniGem Gallery — Proposed Compliance Scope (P1 Discovery & Requirements)

Status: **PROPOSED** by requirements-analyst for compliance-checker confirmation. This is not a final legal determination — it is the input the compliance-checker agent should verify or override before Gate G0 closes.

## Proposed regime-by-regime scope

| Regime | Proposed scope | Justification (one-liner) |
|---|---|---|
| **Decree 13/2023/NĐ-CP — Personal Data Protection Decree (PDPD), Vietnam** | **IN SCOPE** | Site collects name, phone/Zalo ID, date of birth, product interest, and chat transcripts from visitors — this is personal data under PDPD's broad definition, regardless of transaction size. |
| **PCI-DSS** | **PROPOSED OUT OF SCOPE** | No cardholder data (PAN, CVV, expiry) is ever collected, transmitted, or stored by the website; all payment/closing happens off-platform on Zalo between customer and staff (research doc §1, §6c). PCI-DSS is triggered specifically by capture of an actual payment card (PAN) — a future bank-transfer/deposit feature does not by itself trigger PCI-DSS, but does escalate PDPD sensitive-data scope (see `data-classification.md` item 15). |
| **SBV Circulars (banking/payment regulator, Vietnam)** | **PROPOSED N/A** | OmniGem is a retail jewelry business operating a marketing website, not a bank, payment service provider, e-wallet, or card acquirer/processor. No indication in the research doc of any licensed financial activity on the site. |
| **ISO 27001** | **OPTIONAL / ASPIRATIONAL** | Good-practice information security management is beneficial (protects lead PII, CMS credentials, vendor API keys) but not indicated as a contractual or regulatory requirement for this business type; propose as a forward-looking goal, not a Gate G0 blocker. |

## Firm compliance obligations (confirmed by compliance-checker — tracked, not optional)

### Obligation A — Art. 24 Data Protection Impact Assessment (DPIA) dossier
OmniGem, as **data controller**, must prepare and maintain a **DPIA dossier (Hồ sơ đánh giá tác động xử lý dữ liệu cá nhân)** per **Decree 13/2023 Art. 24**, covering all personal-data processing on the website (lead form, chatbot, analytics). The dossier must be available to / filed with the **Ministry of Public Security (A05)** within **60 days of the start of processing**.
- **Target:** dossier drafted and filed **before go-live** (i.e., before the site begins collecting real lead data in production).
- **Owner:** Compliance (placeholder — Product Owner to name a specific accountable person/role before Design phase).
- Tracked as `risk-register.md` R-12.

### Obligation B — Art. 25 Cross-Border Transfer Impact Assessment dossier
Personal data processed by this website leaves Vietnam via multiple vendors. Per **Decree 13/2023 Art. 25**, OmniGem must prepare and maintain a **Cross-Border Transfer Impact Assessment dossier (Hồ sơ đánh giá tác động chuyển dữ liệu cá nhân ra nước ngoài)**, maintained and filed with the **Ministry of Public Security (A05) within 60 days of the transfer** taking place. This is a firm obligation, not a contingent question.

Cross-border processors and what leaves Vietnam:
| Processor | Data that crosses the border | Sensitivity |
|---|---|---|
| **Vercel** | Hosting infrastructure, request/server logs (IP addresses, standard web logs) | PII (online identifier), low sensitivity |
| **Cloudinary** | Product images/video | Low-PII — product imagery only, not customer data |
| **Anthropic (Claude API)** | Chatbot conversation transcripts, which may contain name, DOB, phone, product interest, and — per `data-classification.md` item 9 — potentially incidental Art. 2(4) sensitive data volunteered in free text | PII, possibly sensitive PII |

- **Target:** dossier drafted and filed **before go-live**, and re-assessed if a new cross-border vendor is added.
- **Owner:** Compliance (placeholder — Product Owner to name a specific accountable person/role before Design phase).
- Tracked as `risk-register.md` R-04 (revised) below.

## Top items compliance-checker must rule on

1. **Consent mechanism (Art. 11)** — Confirm the specific reproducible-record consent UX (see PRD US-C1/Epic H) satisfies Art. 11's requirement for express, recorded consent — not merely a checkbox with no retrievable record.
2. **DPIA dossier ownership and filing timeline** — Confirm the named accountable owner and exact go-live date to anchor the 60-day Art. 24 filing clock (Obligation A above).
3. **Cross-border dossier ownership and filing timeline** — Confirm the named accountable owner and exact first-transfer date to anchor the 60-day Art. 25 filing clock (Obligation B above).
4. **Lead storage medium (Google Sheet vs. CRM)** — Confirm whether a Google Sheet is an acceptable PII store under PDPD's security-measure expectations, or whether Gate G0/G1 should require a proper CRM before go-live. (See `risk-register.md` R-03.)
5. **PCI/SBV boundary confirmation** — Formally confirm the "no card data, not a financial institution" position stated above, and flag that any future on-web deposit/payment feature must re-trigger this compliance review — noting (per `data-classification.md` item 15) that a bank-transfer/deposit feature escalates PDPD *sensitive* data scope (Art. 2(4)) independently of whether it also triggers PCI-DSS (which requires actual card/PAN capture).

## Explicit non-claims

- This document does **not** assert final regulatory conclusions. All "proposed" scope decisions above are inputs for the compliance-checker agent, whose sign-off (or revision request) is required to close Gate G0 per `sdlc-graph.yaml` (`loop: "requirements-analyst drafts PRD/classification → compliance-checker confirms it → any disagreement sends requirements-analyst back to revise → repeat until confirmed → stop at Gate G0"`).
- No real PII or card data appears anywhere in this document or its siblings — all data elements are described by field name/type only.
