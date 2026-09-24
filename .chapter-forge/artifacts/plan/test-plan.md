# OmniGem Gallery — Test Strategy & Test Data Plan (P3 Planning, Group 3)

Status: DRAFT for DoR convergence. Author: **test-engineer**. Consumes: `plan/backlog.md` (29 stories), `design/hld-lld.md`, `design/api-spec.yaml`, `discover/data-classification.md`, `design/threat-model.md`, `design/security-review.md`, `design/nfr-targets.md`. Read-only on all of those; this file is the only artifact written here.

No real PII/PAN in this document. Every fixture described below is **generated synthetically at build/test-run time** — no sensitive-shaped literal values are embedded in this file or in any committed test-data file (write-guard + PDPD/PCI discipline).

---

## 1. Test strategy by level

### 1.1 Unit (target ≥80% line coverage — CI gate; domain/service layer)

Scope: pure logic in isolation, no network/DB.

| Service | Key unit cases |
|---|---|
| `LeadService` | valid `LeadInput` → domain object mapping; `contact` requires phone OR zaloId (`anyOf`); rejects when both absent; `dob` format validation; length/pattern boundaries (name 1–120, phone regex, productInterest ≤500). |
| `ConsentService` | `record()` builds `ConsentRecord` with `policyVersion`, `purpose`, `method`, hashed `subject_ref`/`context`; `granted!==true` throws before any side effect; `getByLead()` returns full append-only history in order; HMAC-SHA256 keyed hashing verified against a known test key + known synthetic input → expected digest (never a bare SHA256 — regression-locks M-1 fix). |
| `ChatbotService` + tools | turn loop bounded by `maxIterations`; `search_products` args validated against enum allow-list (`menh`/`loai`) before hitting `ProductQueryService` (regression-locks M-2 GROQ-injection fix); `handoff_to_zalo` output shape-validated; guardrail system prompt re-injected every turn (assert prompt string present in each Claude call payload in a mocked adapter). |
| `DsrService` | `fulfil()` rejects when `approverId === requesterId`/triage identity (maker-checker invariant); state machine transitions `received → pending_verification → verified → fulfilled` only in that order, rejects skips; erasure marks `status=erased` and nulls/crypto-shreds PII columns; retention-override path blocks erasure with a documented reason. |
| `SensitiveInputScrubber` | table-driven: synthetic Luhn-valid digit runs (13–19 digits, generated at test-run time — see §3) are stripped/masked; Luhn-**invalid** digit runs of the same length are left alone (no false-positive over-redaction); synthetic Art. 2(4)-shaped sensitive-category phrases (health/religion/financial-account, in Vietnamese and English synthetic sentences) are flagged/redacted; mixed content (normal text + embedded sensitive span) redacts only the span. |
| `SessionAuthority` | `issueSigned()` returns a token that `verifySigned()` accepts; tampering one byte of a valid token → rejected; a token signed with a different (test-only) HMAC key → rejected; expired token → rejected; token not bound to a passed bot-challenge → rejected. |
| `SpendCircuitBreaker` | `ok()` returns `true` under cap, `false` at/over cap; a false result causes **zero** downstream Claude-adapter calls when composed with `ChatbotService` (mock adapter call-count assertion); cap value is injectable (test uses a placeholder, never a hardcoded prod figure — PO-1 pending). |
| `ZaloLinkBuilder` | prefill syntax matches the confirmed-in-US-INFRA5 format; special characters in `productCode`/`needSummary` are URL-encoded (no injection/open-redirect into the deep-link). |

CI gate: unit suite runs on every PR; coverage report enforced at ≥80% for `src/domain/**` and `src/services/**` (services above); a story cannot merge below the threshold.

### 1.2 Integration / SIT (API routes end-to-end against a test DB)

Run against an ephemeral/test Postgres instance (migrated schema from `erd.md`, never a shared or prod DB) and a mocked Claude API adapter (no real Anthropic calls in CI).

| Flow | Key integration cases |
|---|---|
| `POST /api/lead` | contract test against every response in `api-spec.yaml` (`201/400/403/429/503`); **atomic Lead+ConsentRecord write** — force a mid-transaction failure (e.g., fault-inject after ConsentRecord insert, before Lead insert) and assert **neither** row persists; `consent.granted !== true` → rejected before any DB write (assert zero rows); DB-level negative test: attempt `UPDATE`/`DELETE` on `ConsentRecord` under the scoped app-role credential → DB rejects (grant-level, not app-level — locks M-6 fix); log-scrub assertion: capture log output for a request, assert no raw PII (name/phone/dob) appears, only hashed/redacted forms. |
| `POST /api/chat` | contract test against `ChatInput`/`ChatReply`; **HMAC session** — a request with a forged/tampered `sessionToken` is rejected (`401`/`403` per implementation) before any Claude call; **consent-required path** — `consent` is a required field per the revised schema; a request with `granted=false` or missing consent is rejected with nothing persisted or sent to Claude; **input-scrub path** — a synthetic Luhn-valid card-shaped string in `userMessage` is verified (via mock adapter capture) to reach the Claude-adapter call already redacted; **spend-breaker trip** — with the breaker forced to `ok()=false`, assert the mock Claude adapter receives zero calls and the response is the `fallback:true`/`zaloUrl` shape; **RAG grounding** — `search_products` results returned to the model only ever come from seeded CMS fixtures, never invented (assert against a fixed fixture set). |
| `POST /api/dsr` | contract test (`202/400/429`); `identifier_hash` uses keyed HMAC (assert against known test key/input, not bare SHA-256); **identity-verification state machine** — OTP is sent only to the on-record channel matched from the existing `LEAD`/prior `DSR_REQUEST` record, never to a requester-supplied alternate channel (negative test: submit an `identifier` matching a seeded lead but assert OTP delivery target is the seeded on-record value, not any value echoed from the request); OTP expiry test (expired code leaves status at `pending_verification`); "verified-before-fulfil" enforcement (fulfil call against a non-`verified` status is rejected). |
| `POST /api/collect` | contract test; event dropped (no persistence/relay) when `consentAnalytics !== true`. |
| `US-INFRA2/3` DB roles | connect as each scoped role (app, sales-read, DPO) and assert forbidden operations fail for that role (e.g., app role cannot `DELETE` on `ConsentRecord`, cannot run DDL); encryption-at-rest confirmed via provider API/console, captured as evidence artifact (not a runtime assertion). |

### 1.3 E2E (critical user journeys, Playwright or equivalent)

| Journey | Assertions |
|---|---|
| Gallery → Product Detail → Zalo deep-link | category+mệnh filter narrows result set; PDP renders required fields + both CTAs above-the-fold at mobile breakpoints; Zalo tap fires `click_zalo` analytics event with `productId` only (network-request capture, assert no PII in payload); device-matrix check (mobile app-open intent vs. desktop Zalo Web). |
| Lead-form submit with consent | consent checkbox unticked by default; submit blocked client-side when unticked; invalid input shows inline errors; valid submit shows success state; simulated `503` shows Zalo fallback CTA instead of a bare error. |
| Chatbot consult → handoff | DOB-bearing first turn triggers session-scoped consent write before any reply; bot recommendation cites only seeded-fixture products; purchase-intent phrasing triggers `handoff_to_zalo` and a `chatbot_handoff` analytics event; repeated handoff attempts in one session are deduped (assert only one lead row per session for repeated triggers). |
| DSR request → verify | intake accepted (`202`) even before verification; OTP-gated fulfilment cannot be triggered pre-verification; full happy path (intake → verify → maker-checker fulfil by a **different** identity → erasure reflected) run against seeded synthetic subject data only. |

### 1.4 Performance

| Target (from `nfr-targets.md`) | Test | Tooling |
|---|---|---|
| LCP ≤2.5s / INP ≤200ms / CLS ≤0.1, p75 mobile | Lighthouse CI budget on Home/Collections/Product templates, mobile-4G throttling profile, gated in CI (fails build on regression). | Lighthouse CI |
| `/api/lead` p95 ≤500ms | Load test against the test-DB-backed route, sustained ~5 req/s with burst-50 profile per `nfr-targets.md` §4. | k6 (or similar) |
| `/api/chat` first-token ≤3s (app overhead ≤800ms excl. Claude latency) | Load test with the mocked Claude adapter injecting a representative latency distribution; assert app-side overhead component separately from mocked model latency. | k6 + mock adapter harness |
| `/api/dsr`, `/api/collect` | Smoke-level latency assertions (low volume; p95 ≤500ms / ≤200ms respectively). | k6 |
| Image weight budget (PDP ≤1.0MB initial viewport, hero ≤200KB) | Automated asset-weight check in CI against a built PDP page. | Lighthouse CI / custom script |

### 1.5 Security (maps every threat-model control to a test)

| Threat-model control | Test |
|---|---|
| Prompt-injection resistance (R-14, M-2) | Integration test feeds adversarial synthetic user turns attempting to override the guardrail system prompt / exfiltrate it / coerce false claims; assert the reply never abandons the guardrail framing and never fabricates authenticity/price claims outside seeded fixtures. |
| Forged-assistant-history rejection (R-14) | Because the revised `ChatInput` schema no longer accepts client-supplied prior `assistant` turns (server holds authoritative state), assert any request attempting to smuggle a `messages[]`/assistant-turn field is rejected by schema validation (`400`) — regression-locks the fix. |
| Spend-breaker trip (R-13) | Integration test in §1.2 — breaker tripped ⇒ zero Claude-adapter calls, `fallback:true` response. |
| Session-token forgery rejection (H-1, R-13) | Unit test in §1.1 (`SessionAuthority`) + integration test: a request with a tampered/self-issued `sessionToken` is rejected before reaching the Claude adapter or any rate-limit bypass. |
| Bot-challenge binding (H-1) | Integration test: `issueSigned()` only succeeds after a passed bot-challenge result is supplied; a session token requested without a challenge pass is not issued. |
| DSR impersonation rejection (H-2/H-4, R-15/R-16) | Integration test in §1.2 — OTP goes only to the on-record channel; fulfilment blocked pre-verification; maker-checker distinct-identity enforcement. |
| Luhn card-string scrub (M-7) | Unit test in §1.1 (`SensitiveInputScrubber`) using synthetic Luhn-valid strings generated at test-run time (§3); integration test confirms the scrub runs **before** the Claude-adapter call (input-side), not only before storage — regression-locks the F-01/R-16 mis-ordering fix from the threat model. |
| PII-not-in-logs | Integration log-scrub assertion (§1.2, `/api/lead`) extended to `/api/chat` and `/api/dsr`: capture log output, assert only hashed/redacted identifiers appear, never raw name/phone/dob/free-text PII. |
| Consent atomicity — chat path (R-17) | Integration test: a chat session's first PII-bearing turn (`consent.granted:true`) writes a `ConsentRecord` before any DOB use or transcript persistence; a request with `consent` missing/`granted:false` is rejected by schema (`ChatInput.consent` is now required) with nothing processed — regression-locks the F-07/R-17 fix. |
| Consent append-only enforcement (M-6) | Integration DB-role negative test (§1.2, US-INFRA3) — `UPDATE`/`DELETE` on `ConsentRecord` rejected at the grant level. |
| Keyed-HMAC identifier hashing (M-1) | Unit test (`ConsentService`, `DsrService`) — hash output verified against a known test HMAC key + synthetic input, distinguishing it from a bare SHA-256 of the same input. |
| CSP / security-header baseline present (M-3) | Integration/smoke test against a deployed preview: assert CSP, HSTS, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy` headers present; assert the CSP allow-list includes only the expected third-party origins (Zalo widget, GA4, Meta Pixel, Cloudinary) and nothing broader. |
| CMS-query injection guard (M-2) | Unit test — `search_products` rejects/normalizes a `loai` value outside the declared enum allow-list before it reaches the CMS query layer. |
| DB credential least-privilege (R-19) | Integration test (§1.2) — each scoped role (app/sales/DPO) is exercised and forbidden operations for that role fail. |
| No-user-controlled-fetch (L-2) | Static/code-review check (not a runtime test per se) — CI lint rule or code-reviewer checklist item asserting no endpoint/tool accepts a user- or model-supplied URL for server-side fetch; flagged here so it's tracked, executed by `security-reviewer` at P4/P5 rather than test-engineer. |

Security tests that require an actual Anthropic call (e.g., true end-to-end prompt-injection against the live model) are **out of scope for the deterministic CI suite** — those run against the mocked adapter for determinism; a separate, manually-triggered sandbox run against the real Claude API (using only synthetic, non-production data, and only after the PO-4 DPA gate for any live traffic) is coordinated with `security-reviewer` for periodic DAST/pentest-style validation, not part of the CI regression gate.

---

## 2. Per-story test approach (DoR criterion 5)

Every story in `backlog.md` already carries a `test_approach` hint from the requirements-analyst; this table restates it with concrete deterministic cases (levels + key positive/negative/abuse cases) to close DoR criterion 5. All 29 stories have a deterministic approach below — **none require a DoR revision loop on this criterion.**

### Sprint 0 — Infrastructure

| Story | Levels | Deterministic cases |
|---|---|---|
| US-INFRA1 — Project scaffold | Integration | CI asserts `npm run dev`/build succeeds; header-presence smoke test (CSP/HSTS/X-Content-Type-Options/Referrer-Policy/Permissions-Policy) against the deployed preview URL; `Cache-Control: no-store` verified on a placeholder `/api/*` route. |
| US-INFRA2 — Postgres lead store | Integration | Connect as each scoped role (app/sales/DPO), assert forbidden ops (DELETE on ConsentRecord, DDL) fail per role; encryption-at-rest confirmed via provider console/API, captured as evidence, not a live assertion (infra fact, checked once per environment). |
| US-INFRA3 — ConsentRecord/PrivacyPolicyVersion schema | Integration | Migration-apply test asserts field/type parity with `erd.md` §1; negative test — app-role `UPDATE`/`DELETE` on `ConsentRecord` rejected by the DB; withdrawal-as-new-row test (never an update). |
| US-INFRA4 — CmsAdapter + Sanity schema | Integration | Schema-validation test (Collection/Product/ElementTag/ProductElement match `erd.md`); webhook-triggers-revalidation smoke test against a staging deploy; read-only token cannot perform a write (negative test). |
| US-INFRA5 — Verified Zalo OA | Manual | Manual QA pass: deep-link opens Zalo app (mobile) and Zalo Web (desktop) with correct prefill, executed once before US-B1 is marked done. Deterministic in the sense of a fixed, repeatable manual script with a pass/fail checklist — appropriate for an external/ops story with no code under test. |

### Sprint 1 — MVP (Gallery + Lead Funnel)

| Story | Levels | Deterministic cases |
|---|---|---|
| US-A1 — Home page | E2E | Lighthouse CI budget check (LCP/INP/CLS against interim "good" thresholds); Playwright click-through from featured card → PDP. |
| US-A2 — Collections + feng-shui filter | Unit + E2E | Unit: filter-logic function given a fixed synthetic product/tag fixture set returns the expected subset for each category×mệnh combination. E2E: Playwright selects category+mệnh, asserts rendered result set matches fixture expectation; empty-state fixture (no matches) asserts the fallback CTA renders. |
| US-A3 — Product Detail page | E2E | Playwright asserts all required fields render from a seeded synthetic product fixture; viewport assertions confirm both CTAs are above-the-fold at defined mobile breakpoints; Lighthouse CWV check. |
| US-A4 — Brand story page | Manual/content-review | Deterministic copy-diff check: brand-story page text vs. the source-of-truth `create-facebook-fanpage.md` §2 text, run as a scripted diff (not subjective judgment) — pass/fail on exact-match of the reused commitment language. |
| US-B1 — Zalo deep-link CTA | E2E | Device-matrix Playwright/manual test (mobile app-open intent vs. desktop Zalo Web) against the confirmed prefill syntax from US-INFRA5; `click_zalo` analytics event assertion via network-request capture (payload = `productId` only, no PII). |
| US-B2 — General Zalo widget | E2E | Playwright smoke test on 3 representative pages asserts widget renders; CSP header inspection confirms the Zalo origin is allowlisted and no broader origin was added. |
| US-H1 — Privacy Policy page | Integration + E2E | Integration: publish a new `PrivacyPolicyVersion`, assert `content_hash`/`published_at` recorded immutably (re-publish does not mutate the prior row). E2E: link-reachability test from footer, lead form, and chat entry point. |
| US-H2 — Consent capture (Art. 11) | Unit + Integration | Unit: `ConsentService.record()`/`getByLead()` logic, HMAC hashing (known-key/known-input digest check). Integration: `granted=false` rejected server-side before any write; append-only enforcement verified via the DB constraint from US-INFRA3 (shared test). |
| US-C1a — Lead form UI | Unit + E2E | Unit: client-side Zod-mirrored validation logic (missing name/phone-and-zaloId → blocked). E2E: Playwright — invalid submit shows error; valid submit shows success state; consent left unticked blocks submit; simulated `503` shows Zalo fallback CTA. |
| US-C1b — `POST /api/lead` atomic write | Integration | Contract test against all `api-spec.yaml` response codes (400/403/429/503/201); transaction-rollback test (forced mid-tx failure — assert neither ConsentRecord nor Lead row persists); log-scrub assertion (no raw PII in captured logs). |
| US-E1 — JSON-LD structured data | Unit/Integration | Schema-validator assertion (structured-data test) run in CI against a built PDP page for `Product` JSON-LD; separate assertion for site-wide `Organization` schema presence. |
| US-F1 — CMS catalog editing | Manual/E2E | Staff edits a product in Sanity Studio; timed test asserts live-site reflects the change within the ISR window, or instantly via the on-publish webhook trigger (reuses US-INFRA4's webhook test harness). |
| US-G1 — Funnel analytics | E2E | Playwright + network-request interception: each of `view_product`/`click_zalo`/`submit_lead_form`/`chatbot_handoff` fires with the correct name and zero-PII payload; a consent-off run asserts GA4/Meta Pixel tags do not load at all. |

### Sprint 2 — AI Chatbot

| Story | Levels | Deterministic cases |
|---|---|---|
| US-D-INFRA1 — Chatbot security primitives | Unit | Table-driven scrubber tests (synthetic Luhn-valid/invalid strings, synthetic sensitive-category phrases, mixed content); session forgery/replay/expiry tests (`SessionAuthority`); spend-breaker trip test asserting zero Claude-adapter calls when tripped; per-session/per-IP rate-limit-exceeded → `429` test. |
| US-D1 — Feng-shui consultation | Integration | Contract test against `ChatInput`/`ChatReply`; consent-gate test (`consent` missing/`granted:false` → turn rejected, nothing persisted); RAG-grounding test (`search_products` results only ever match a fixed seeded CMS fixture set — assert no product appears in a reply that isn't in the fixture). |
| US-D2 — Trust FAQ | Integration | Seed CMS FAQ fixtures matching `create-facebook-fanpage.md` §2 content; assert bot answers for representative FAQ-shaped questions match seeded content (content-consistency check, exact or near-exact string match against the fixture, not subjective grading); assert the guardrail prompt blocks sensitive-data solicitation even on FAQ-adjacent adversarial synthetic prompts. |
| US-D3a — `handoff_to_zalo` + lead write | Integration | Assert a chatbot-sourced lead row matches the shape/encryption of a form-sourced lead (US-C1b); rate-limit/dedup test — repeated handoff triggers within one session produce exactly one lead row; `chatbot_handoff` analytics event assertion (extends US-G1's harness). |
| US-D3b — Chatbot go-live gate | Manual | Fixed checklist: DPA-on-file confirmation (PO-4), spend-cap-set-to-real-figure confirmation (PO-1), signed off by Product Owner + Compliance before the production feature flag flips. Deterministic as a pass/fail checklist against named, auditable artifacts (not subjective). This story stays **DEFERRED** per the backlog — build/sandbox testing (covered by US-D-INFRA1/D1/D3a above) can proceed now; this gate itself cannot close until PO-1/PO-4 land. |

### Sprint 3 — DSR automation + compliance dossiers

| Story | Levels | Deterministic cases |
|---|---|---|
| US-H3a — DSR intake API | Integration | Contract test against `DsrInput`/`DsrAccepted` (400/429/202); assert `identifier_hash` uses keyed HMAC (known-key/known-input digest check, not bare SHA-256). |
| US-H3b — DSR identity verification (H-4) | Integration | Negative test — OTP delivered only to the on-record channel (from a seeded lead/prior DSR record), never to a requester-supplied alternate; expiry test (code expires → status remains `pending_verification`); enforcement test — fulfilment attempted pre-`verified` status is rejected. |
| US-H3c — DSR fulfilment + maker-checker | Integration | `fulfil()` rejects when `approverId` equals the intake/triage identity; erasure test asserts PII columns nulled/crypto-shredded and `status=erased`; retention-override test — an active legal-hold flag blocks erasure with a documented reason in the response. |
| US-H-COMP1 — DPIA dossier | Manual | Compliance sign-off checklist against the enumerated PII stores/purposes from `hld-lld.md` §C — not code-testable; owned by `compliance-checker`, tracked here as **DEFERRED** on PO-3 per the backlog (drafting can proceed now). |
| US-H-COMP2 — Cross-border TIA dossier | Manual | Compliance sign-off checklist per-vendor (Vercel/Cloudinary/Anthropic) against `compliance-scope.md` Obligation B — not code-testable; **DEFERRED** on PO-3/PO-4 per the backlog. |

### Sprint 4+ (noted, not fully elaborated per backlog scope)

| Story | Levels | Deterministic cases |
|---|---|---|
| US-E2 — SEO blog | E2E | Publish-and-crawl smoke test: sitemap inclusion check, internal-link presence assertion from a published synthetic blog fixture. |

**DoR criterion 5 convergence result: 29/29 stories have a deterministic test approach.** No story is flagged for a DoR revision loop on this criterion. (US-D3b, US-H-COMP1, US-H-COMP2 remain DEFERRED for *closure*, per the backlog's own PO-1/PO-3/PO-4 blocking notes — their *build-time* test approach is still deterministic, which is the DoR-5 requirement; DoR-5 is about testability of the story as scoped, not about whether the PO decision has landed.)

---

## 3. Test data plan (masked/synthetic ONLY)

**Rule: no real PII, no real PAN/card data, ever — in any test file, fixture, log, or committed artifact.** All test data is synthetic, generated deterministically, and never resembles a real person or a real payment credential closely enough to be mistaken for one.

### 3.1 Generation approach

- **Faker-style, seeded, deterministic:** use a faker library (e.g., `@faker-js/faker`) seeded with a fixed integer per test suite run, so fixtures are reproducible across CI runs and across developer machines without being hardcoded literals in the repo. Seeds are documented per fixture module (e.g., `seed=42` for lead fixtures) so a failing test is reproducible.
- **Naming convention:** all synthetic names use an obviously-fake domain (e.g., faker's generated Vietnamese-locale names, or a fixed prefix like `"Test Subject <n>"`) — never real public figures, never names resembling real customers/staff.
- **Phone/Zalo ID:** synthetic phone numbers generated in a clearly non-issuable range (e.g., a fixed test-reserved prefix combined with a faker-generated suffix) that matches the `LeadInput.contact.phone` regex (`^\+?[0-9]{9,15}$`) but is not a real dialable Vietnamese number. Zalo IDs are faker-generated opaque strings, not real Zalo account identifiers.
- **DOB / feng-shui (ngũ hành) mapping:** synthetic dates of birth are generated across a fixed calendar-year range (e.g., 1960–2005) with a seeded generator so the DOB→mệnh mapping table is exercised deterministically — one synthetic DOB fixture per mệnh (Kim/Mộc/Thủy/Hỏa/Thổ) so `US-D1`'s recommendation logic is tested against all five branches, plus one boundary-date fixture per lunar/solar-calendar edge case the mapping logic defines (e.g., year boundaries), all synthetic and never a real person's birth date.
- **Chat transcripts:** synthetic multi-turn conversations authored as fixture files, covering: a clean feng-shui consult, an FAQ-only conversation, a purchase-intent conversation that should trigger handoff, an adversarial prompt-injection attempt, and a conversation that incidentally includes a synthetic sensitive-category phrase (to exercise the scrubber) — all authored text, never captured from a real user.
- **Consent records:** synthetic `ConsentRecord` fixtures reference a fixture `PrivacyPolicyVersion` (synthetic `content_hash`, fixed `published_at`), with `subject_ref_hash`/`context_hash` computed at test-run time via the real HMAC function against synthetic inputs and a **test-only** HMAC key (never the production key, sourced from a test-environment secret, not committed).

### 3.2 Card-string (Luhn) scrubber testing without committing PAN-shaped literals

- The scrubber unit tests **generate Luhn-valid digit strings programmatically at test-run time** using a standard Luhn-checksum-append algorithm seeded with a fixed test seed — e.g., a helper `makeLuhnValidDigits(seed, length)` that builds a random base digit run and appends the correct Luhn check digit. This produces a valid-Luhn 13–19 digit string for the test assertion without ever writing a literal digit-run that looks like a real card number into a source or fixture file (which would itself trip the same write-guard class of control and risks an accidental real-PAN collision).
- A companion **Luhn-invalid** digit string (same length, checksum deliberately broken) is generated the same way, to assert the scrubber does not over-redact non-card numeric content (e.g., order IDs, phone numbers of similar length).
- These generator helpers live in the test-utilities module, not in any data/fixture file, and are documented as "generates synthetic test-only digit strings — never a real or plausible customer card number."

### 3.3 Fixture inventory (summary)

| Fixture set | Contents | Used by |
|---|---|---|
| `synthetic-leads` | Seeded faker name/phone/zaloId/dob/productInterest combinations | US-C1a/b, US-D3a, US-H3a/b/c |
| `synthetic-consent-records` | Seeded ConsentRecord rows tied to a fixture PrivacyPolicyVersion | US-H2, US-H3b/c |
| `synthetic-chat-transcripts` | Authored multi-turn fixtures (clean / FAQ / handoff / injection / sensitive-phrase) | US-D1/D2/D3a, security tests |
| `synthetic-catalog` | Seeded Product/Collection/ElementTag fixtures (Kim/Mộc/Thủy/Hỏa/Thổ coverage) | US-A1–A4, US-D1, RAG-grounding tests |
| `luhn-generators` | Programmatic Luhn-valid/invalid digit-string generators (no literals committed) | US-D-INFRA1 scrubber tests |
| `dsr-fixtures` | Seeded DSRRequest + matching on-record-channel fixtures for OTP-target tests | US-H3a/b/c |

---

## 4. Regression approach

- **CI regression suite composition:** the full unit suite (§1.1) + the integration/SIT suite (§1.2) run on every PR against the test DB and mocked Claude adapter; the E2E suite (§1.3) runs on every PR against a preview deployment; performance budgets (§1.4, Lighthouse CI) run on every PR for the three key templates and on a schedule (nightly) for the full site; k6 load tests run on a schedule (not every PR, to control CI cost) plus on-demand before a release candidate.
- **Security regression:** every test case in §1.5 is part of the standard integration/unit suite (not a separate manual pass) so a regression in any threat-model control fails the same PR gate as a functional bug — this is deliberate: these are correctness tests, not optional hardening, per the "maker not checker" boundary (test-engineer proposes the gate; `security-reviewer`/`compliance-checker` independently verify before G2).
- **Flakiness control:**
  - No test depends on wall-clock time without an injectable clock (e.g., OTP expiry, ISR revalidation-window, ack-SLA deadline tests use a fake/advance-able clock, not real `sleep`).
  - No test depends on the real Claude API, real Zalo platform, or real third-party network calls in the CI gate — all are mocked/stubbed (a separate, explicitly-labeled manual/sandbox suite exists for periodic real-integration smoke checks, run outside the PR gate).
  - E2E tests use fixed seeded fixture data (§3) so assertions are exact-match, not fuzzy/timing-dependent.
  - Any test that fails intermittently twice in a rolling 2-week window is quarantined (marked skip with a tracked defect, not deleted) and reviewed by `test-engineer` before re-enabling — consistent with "report a defect, don't fix the test to make it pass" when the underlying code is at fault, versus genuine test flakiness which does get hardened.
  - Load/performance tests (k6, Lighthouse) run against a dedicated preview/staging deployment, never against production, to avoid noisy-neighbor flakiness from shared infra.
