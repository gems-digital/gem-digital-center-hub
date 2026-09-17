# ADR-0002 — Lead Store: proper CRM/database vs Google Sheet

Status: PROPOSED (Gate G1). Deciders: Architect + Product Owner + Compliance. Resolves PRD Open Question 7. Addresses **R-03** head-on.

## Context
Leads (name, phone/Zalo, DOB, interest), ConsentRecords, ChatTranscripts, and DSRRequests are **PII under PDPD** (data-classification #5–10). The research doc treated "CRM/Google Sheet" as interchangeable. **R-03 (High):** a Google Sheet has materially weaker access control, audit trail, and encryption than a proper store — and PDPD Art. 24/26 expects appropriate security measures for a data controller. This is the single highest-leverage security decision in the design.

## Decision
Use a **proper managed store with encryption-at-rest, row-level access control, and audit logging** — **NOT** a Google Sheet — as the system of record for all PII entities. Recommended: a **managed PostgreSQL** (e.g., Supabase/Neon/RDS) fronted by the `LeadRepository` interface, OR a purpose-built CRM if the business already operates one.
- Encryption at rest (AES-256) at storage level + application-layer/column encryption for PII-enc fields (HLD B.5, erd §2).
- Least-privilege scoped DB credential for the app; separate read-only role for sales staff; DPO role for DSR/erasure; audit log on read/write (SoD, HLD B.6).
- If a Google Sheet is used as a **temporary stopgap only**, it is a documented exception requiring: restricted named-account sharing (no public/link sharing), no DOB/free-text stored in it, access logging, and a committed migration date — flagged as an accepted risk by Compliance, not a default.

## Consequences
- (+) Directly retires R-03; supports Art. 11 append-only consent, DSR erasure/anonymization (US-H3 AC3), retention purge, and audit evidence for the DPIA (Art. 24).
- (+) `LeadRepository` abstraction keeps the store swappable (managed Postgres ↔ CRM) without touching domain services.
- (−) More setup than a Sheet; a DB is another (likely cross-border) SaaS — must appear in the Art. 25 dossier (R-04). Prefer a region/provider with a DPA; evaluate VN/APAC region if available.
- (−) Small recurring cost vs a free Sheet — justified by the PII risk profile; fits within the NFR-6 ceiling.
