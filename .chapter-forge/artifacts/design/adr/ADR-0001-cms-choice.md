# ADR-0001 — Headless CMS: Sanity vs Payload

Status: PROPOSED (Gate G1). Deciders: Architect + Product Owner. Resolves PRD Open Question 3.

## Context
Need a headless CMS for Public catalog/blog content that non-technical shop staff manage (US-F1) and that the app reads at build/ISR time (US-A*, US-E2) and the chatbot RAG reads live (US-D1/D2). No PII lives in the CMS (data-classification #1–4). Priorities: editor UX in Vietnamese context, ISR/webhook support, low cost (NFR-6), fast time-to-market, and it must **not** become a lead/PII store (SoD, R-03).

## Decision
Adopt **Sanity** as the CMS.
- Managed SaaS → zero DB/ops overhead, fits the low-cost/low-maintenance intent and a small team.
- Strong structured content + GROQ querying suits catalog + ngũ-hành tags + FAQ-as-content (single source of truth for chatbot copy, R-10).
- First-class ISR/on-publish webhooks for Next.js revalidation (US-F1 revalidation window).
- Real-time editor experience with good localization support.
- Payload remains the fallback if data residency or self-hosting becomes a hard requirement (it is self-hostable on our own Postgres) — revisit if compliance later requires VN-resident content hosting.

## Consequences
- (+) Fastest path to a staff-manageable catalog; minimal infra; generous free tier.
- (+) Clean SoD: editors touch only Sanity; it holds no PII, so a CMS breach does not expose leads.
- (−) Another cross-border SaaS (content only, low-PII) → note in vendor list, but no customer PII, so **not** an Art. 25 concern for personal data.
- (−) Vendor lock-in to Sanity's schema/GROQ; mitigated by the `CmsAdapter` interface isolating the app from CMS specifics.
- Read-only API token for the app; editor accounts use MFA (SoD, secret management per HLD B.5/B.6).
