# OmniGem Gallery — Release Plan Sketch (P3 Planning)

Status: **PRELIMINARY SKETCH — NOT a Change Request.** Produced at P3 to catch release-blocking assumptions
early, in parallel with backlog refinement (`backlog.md`) and the test strategy. The full **Change Request**,
CAB approval package, and executed **Rollback Plan** are only produced at **P6 — Release (Gate G3)**, once
code exists to describe. Nothing here should be read as a deployment authorization or a tested rollback claim.

Author: release-manager. Inputs: `backlog.md` (Sprint 0–4 slicing, PO-1..PO-4 blocking decisions),
`ADR-0004-rendering-strategy.md` (SSG+ISR on Vercel), `nfr-targets.md` (99.5% availability, FIRM spend
circuit-breaker), `hld-lld.md` (deployment shape), `sdlc-state.json` (carried compliance obligations, open
PO items). No real PII/PAN referenced.

---

## 1. Release train / cadence

Sprints do **not** map 1:1 to releases. Proposed grouping, tied to the backlog's own MVP boundary:

| Release | Sprints | Scope | Nature |
|---|---|---|---|
| **R0 — Preview only** | Sprint 0 | Infra scaffold (Next.js/Sanity/Vercel, Postgres dev instance, CSP baseline) | Internal only — Vercel preview deployments per PR, never promoted to production traffic |
| **R1 — MVP soft-launch** | Sprint 0 + Sprint 1 | Gallery browsing, mệnh/category filter, Zalo CTAs, consent-gated lead form with atomic Lead+ConsentRecord write, published privacy policy | First production release. "Soft" because no chatbot, no DSR self-service yet — funnel is Zalo + form only |
| **R2 — Chatbot (feature-flagged)** | Sprint 2 | AI chatbot (feng-shui consult, FAQ, Zalo handoff) behind a feature flag/kill-switch, defaulting OFF until go-live prerequisites clear | Follow-on release; flag lets R1 stay live and stable while chatbot lands in prod code but stays dark |
| **R3 — DSR + compliance close-out** | Sprint 3 | DSR intake/OTP/fulfilment workflow, DPIA + cross-border TIA dossiers filed | Should land **before** real lead/chat volume scales meaningfully, per compliance carried-obligations |
| **R4+ — Growth** | Sprint 4+ | SEO blog, remarketing, CTA A/B, ZNS care | Incremental releases on the established train, standard cadence |

**Promotion model (Vercel):** every PR gets an ephemeral **preview** deployment (Vercel Preview Environments) for
review/QA; merges to `main` build a **production candidate**; promotion to **production** is a distinct, deliberate
step (Vercel's promote-to-production action or a protected `main`→prod pipeline), not automatic-on-merge, so that
the G3/CAB gate (P6) has a concrete artifact to approve before it serves real traffic.

## 2. Environments

| Environment | Purpose | Where data sits | PII rule |
|---|---|---|---|
| **Local / dev** | Developer machines | No shared DB; local/sandbox Postgres or mocked `CmsAdapter` | **No real PII** — synthetic data only (per `synthetic-test-data` guidance) |
| **Preview** (Vercel, per-PR) | Code review, QA, e2e | Points at a **sandbox** Postgres instance (not prod) and a Sanity **dev dataset**; Claude API calls (once chatbot lands) go through a sandboxed/rate-capped key | **No real PII/PAN** — synthetic leads/transcripts only, consistent with `backlog.md`'s write-guard note |
| **Staging** | Pre-prod integration/UAT, rollback rehearsal | Staging Postgres (same region/DPA posture as intended prod, once PO-2 resolves) + Sanity staging dataset | **No real PII** — synthetic data; staging is where rollback drills (§5) should actually be exercised before P6 |
| **Production** | Real visitor traffic | Production Postgres (region/vendor **pending PO-2**) with encryption at rest (M-6, `erd.md`); Sanity production dataset (public content only — CMS holds no PII, SoD per ADR-0001) | Real PII appears only here, only after go-live prerequisites (§4) are met |

Note: Sanity (catalog CMS) never holds PII in any environment (ADR-0001 SoD), so its per-environment split is
lower-risk than Postgres's. Postgres is the environment boundary that matters most for the no-real-PII rule.

## 3. Release gates that will apply later (reference only — not executed here)

| Gate | Phase | What it checks |
|---|---|---|
| **CI gate** | P4 — Development | Build, lint, tests, CSA/dependency scan (M-5) green before merge |
| **G2 — Quality/Security** | P5 — Testing | Test coverage, security review sign-off, NFR verification against `nfr-targets.md` |
| **G3 — Change approval (CAB)** | P6 — Release | Change Request, Release Notes, Rollback Plan reviewed and approved by CAB; SoD enforced (deployer ≠ developer) |
| **GO_LIVE smoke** | P7 — Deployment | Post-deploy smoke test + health check before traffic is declared stable |

This sketch exists to flag, ahead of time, what is likely to block or complicate G3/G0_LIVE so P4/P5 work can
de-risk it — it does not pre-approve or substitute for any of these gates.

## 4. Go-live prerequisites (blocking)

These are carried compliance/PO obligations (`sdlc-state.json`, `backlog.md` §0) that must resolve before the
sprint that depends on them can go live with real traffic/data. None are new — this section just ties each to
the sprint/release it gates so P6 doesn't discover them cold.

| Prerequisite | Blocks | Gates release |
|---|---|---|
| **Anthropic no-training / zero-retention DPA signed** (PO-4, ADR-0003 "BLOCKING PREREQUISITE") | Chatbot serving real traffic (sandbox build/test is fine without it) | **R2 (chatbot)** — must be signed before the feature flag is flipped ON in production |
| **DPIA (Art. 24) dossier filed, named owner** (PO-3) | US-H-COMP1 "done" | **R3** — should be filed before lead/chat volume scales past pilot levels |
| **Cross-border TIA (Art. 25) dossier filed, named owner** (PO-3) | US-H-COMP2 "done"; also feeds Postgres/Cloudinary/Anthropic/GA4 cross-border assessment | **R3**, informed by R1's Postgres choice and R2's Anthropic DPA |
| **Postgres region/vendor + DPA chosen** (PO-2) | US-INFRA2 production cutover (dev/sandbox instance can proceed without it) | **R1** — MVP soft-launch cannot promote its lead store to production until this is decided; also a dependency input to the Art. 25 dossier |
| **Chatbot cost ceiling / spend-cap amount set** (PO-1) | SpendCircuitBreaker needs a concrete VND figure; enforcement *mechanism* is already FIRM regardless of amount (`nfr-targets.md` §6) | **R2** — flag can ship OFF without this, but cannot be flipped ON without a live cap |
| **Privacy Policy published (versioned)** (US-H1) | Any consent-gated PII collection (lead form, chatbot) | **R1** — this is in-scope for Sprint 1 itself and is on the MVP critical path already (`backlog.md` §2) |

**Reading order:** R1 (MVP) is gated mainly by PO-2 + the privacy policy (already in-sprint). R2 (chatbot) is
gated by PO-1 + PO-4. R3 (DSR/compliance) is gated by PO-3. This lets the team plan owner follow-up per release
rather than treating all four PO decisions as one monolithic blocker.

## 5. Rollback sketch (preliminary — not the P6 Rollback Plan)

**Easy / near-instant:**
- **Static/ISR pages** (Home, Collections, PDP, Brand, Blog, Privacy Policy) — Vercel's instant rollback to the
  prior production deployment (atomic alias swap) covers all SSG/ISR content immediately; no data migration
  involved.
- **Chatbot feature (R2)** — a feature-flag kill-switch is the primary rollback lever, not a redeploy: flipping
  the flag OFF fails the experience over to the Zalo-handoff CTA gracefully (mirrors the NFR-4 degrade rule for
  Claude API outages), so chatbot issues don't require rolling back the whole release train.

**Needs care:**
- **DB schema changes** (Postgres — Lead, ConsentRecord, PrivacyPolicyVersion, ChatTranscript, DSRRequest) —
  posture is **expand/contract**: additive migrations first, old columns/tables dropped only in a later,
  separate release once nothing reads them. **No destructive migrations on PII-bearing tables**, ever, in the
  same release as the feature that depends on them — this keeps a code rollback safe even if a schema migration
  has already run forward.
- **ConsentRecord specifically** is DB-enforced **append-only** (M-6 — UPDATE/DELETE revoked for the app role).
  This is a feature, not a rollback obstacle: a bad release can't corrupt consent history, but it also means
  rollback can never "undo" a consent write — only a forward-compensating insert (e.g., a correction event) is
  possible.
- **Lead data written during a bad release** is real business data (once in production) — rollback of *code*
  does not retroactively fix or remove rows written under the buggy version; any such fix is a data-remediation
  task, not a deployment rollback.

**One-liner:** static/ISR pages and the chatbot flag roll back instantly; anything touching the Postgres schema
or consent/lead rows needs an expand/contract, no-destructive-migration posture instead of a hard rollback.

Rollback **testing** (drilling an actual rollback in staging, confirming smoke tests pass post-rollback) is a
P6/P7 activity, not claimed as done here — this section only sketches the posture the team should design toward
during P4 build.

## 6. Deployer ≠ developer (SoD) — note for later gates

Carried forward as a **binding note for P6/P7**, not enforced by this sketch: the person who deploys to
production must not be the same person who authored the code/PR being deployed (Separation of Duties). This
mirrors the SoD table already established in `hld-lld.md` B.6 for CMS/Lead Store/Deploy roles. At P6, the
Change Request and CAB approval package must name a deployer distinct from the developer(s) on the change, and
at P7 the release-manager will enforce this before executing (or assisting with) the deploy — production
deploys are human actions via PAM; this agent does not self-deploy or self-approve.

---

*End of preliminary sketch. Revisit and supersede with the full Change Request at P6, once R1's actual code and
diff exist to describe.*
