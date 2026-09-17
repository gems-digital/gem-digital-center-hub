# OmniGem Gallery — Proposed Gate G1 Checklist (P2 Design)

Status: DRAFT for Gate G1. Author: solution-architect (Maker). Approvers (Checker): **Architect + Security** — sign-off is theirs, not the author's (SoD / four-eyes).
Criteria source: `sdlc-graph.yaml` gate `G1` ("Design / Security Gate").

| # | G1 criterion | Evidence artifact | Maker status |
|---|---|---|---|
| 1 | NFR targets defined (latency, throughput, availability) | `nfr-targets.md` §2–§4, §8 | ✅ drafted (numbers pending PO/Architect confirm) |
| 2 | Threat model (STRIDE) complete | `threat-model.md` — **OWNED BY threat-modeler** (input: `system-context.md` §3 boundaries, 3 sensitive flows in `hld-lld.md` A.3) | ⏳ handed off |
| 3 | Sequence diagram(s) cover every sensitive flow | 3 flows documented in `hld-lld.md` A.3 for threat-modeler to render as sequences: (1) lead-form, (2) chatbot+handoff, (3) analytics | ⏳ handed off |
| 4 | ERD complete, PII/card-data fields flagged | `erd.md` §1 diagram + §2 register (card data = none) | ✅ drafted |
| 5 | Architecture review passed | `hld-lld.md` + `class-diagram.md` for design walkthrough | ⏳ walkthrough pending |
| 6 | SoD design & encryption at-rest/in-transit met | `hld-lld.md` B.5 (encryption) + B.6 (SoD matrix) | ✅ drafted |
| 7 | ADR approved | `adr/ADR-0001..0005` | ✅ drafted, PROPOSED |
| 8 | Design walkthrough notes show all findings resolved or accepted with rationale | to be produced after threat-modeler + security-reviewer passes | ⏳ pending review loop |

## Handoffs
- **threat-modeler (STRIDE):** consume `system-context.md` trust boundaries (TB-1…TB-5) and the 3 sensitive flows; produce sequence diagrams + STRIDE findings.
- **security-reviewer (OWASP/PCI):** review `api-spec.yaml`, `hld-lld.md` B.1–B.6. Note: no PCI scope (no card data). Focus on OWASP (input validation, authz/SoD, secrets, rate limiting, cross-border PII).
- Any finding → solution-architect revises the affected artifact (design loop), then walkthrough → G1 sign-off by Architect + Security.

## Compliance obligations carried into design (for the record)
- Art. 11 consent (reproducible): ADR-0005, `hld-lld.md` B.3, `erd.md` ConsentRecord.
- Art. 24 DPIA readiness (R-12): design artifacts enumerate every PII store + purpose.
- Art. 25 cross-border (R-04): `system-context.md` §4 labels every egress.
- R-03 lead-store security: ADR-0002 (proper store, not Google Sheet) + encryption + SoD.
