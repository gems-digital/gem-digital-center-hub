# ADR-0004 — Rendering strategy (SSG / ISR / SSR per page type)

Status: PROPOSED (Gate G1). Deciders: Architect. Supports NFR-1 (CWV), NFR-3 (SEO), NFR-6 (cost).

## Context
Vietnam mobile-4G audience, heavy jade imagery (R-08), SEO is a core growth channel (US-E1/E2, NFR-3), staff must publish without redeploys (US-F1), and API routes handle PII that must never be cached (compliance). We must pick a per-page-type rendering strategy on Next.js App Router.

## Decision
Use **SSG + ISR** for all public content, **client interactivity** for filtering/widgets, and **dynamic uncached** handlers for PII APIs.

| Page type | Strategy | Revalidate |
|---|---|---|
| Home, Brand, Certification, Blog | SSG + ISR | 60 min + on-publish webhook |
| Collections | SSG + ISR; mệnh/category filtering client-side over prefetched set | 30 min |
| Product Detail | SSG + ISR via `generateStaticParams` (Product JSON-LD for SEO) | 30 min + webhook |
| Privacy Policy | SSG, **versioned** (version pinned by ConsentRecord) | on publish |
| DSR intake | SSG shell + client form → `/api/dsr` | — |
| `/api/*` | Dynamic, `runtime=nodejs`, `Cache-Control: no-store` | never |

## Consequences
- (+) Crawlers get full static HTML (NFR-3 SEO); top CWV from edge-served static + `next/image`/Cloudinary (NFR-1, R-08); few origin hits (NFR-6 cost).
- (+) Staff edits go live within the revalidate window / on webhook without redeploy (US-F1).
- (+) **FIRM invariant:** no PII-bearing response is ever edge/CDN cached — API routes are dynamic + `no-store`.
- (−) ISR staleness window (max 30–60 min) for content; acceptable for a catalog, tunable via on-publish webhooks.
- (−) Client-side filtering requires shipping the collection set; bounded by pagination if the catalog grows large.
