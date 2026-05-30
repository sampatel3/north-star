# Taali → Mainspring Convergence — Execution Plan

> **Status:** approved 2026-05-30 (Sam). Executes ADR-0009 (migration-debt inventory),
> ADR-0010 (convergence strategy), ADR-0002 (substrate/brand boundary), ADR-0011/0012.
> This is the living execution checklist; update statuses as cuts land.

## Reframe (verified 2026-05-30)
Mainspring's substrate is **already built and seamed** — every target capability (metering,
policy engine, agent runtime, budget, HITL/decisions, audit, promotion gate, KG interface,
capability registry) is built, DB-backed, exposed behind a Protocol, and guarded by a parity
CI gate. The ADRs' "seams don't exist yet" is **stale**. So convergence is **not**
build-from-scratch — per capability it is: **vendor the ORM-free seam → shadow it against
tali's copy → confirm parity on live traffic → swap the import.** tali-platform backend has
**zero `from mainspring` imports** today; the duplication is total and parallel.

## Mechanism (decided, proven on metering)
- **Per-capability vendored ORM-free seam** into `backend/vendor/<cap>/`, pinned by
  `MAINSPRING_REF.txt` + a `vendor_*.sh` re-vendor script, fed by mainspring's WS-E release bot.
- **In-process import** (not a pip/git dep, not the HTTP brain-feed boundary).
- **Parallel-prod behind a per-capability flag** (`MAINSPRING_<CAP>_SHADOW`): tali's own
  implementation stays the live source of truth; the seam runs only in shadow (log-only,
  never-raises, no-op when off) until parity is confirmed, then a tiny import-swap cuts over.
- **Stores never move.** tali keeps its own ORM tables; the seam only changes *who computes*,
  not *where rows live*. (KG store stays tali's Graphiti behind the Protocol — interface-only.)

## Cross-cutting prerequisites
| | Prereq | When | Approach |
|---|---|---|---|
| P0 | Vendoring mechanism + metering scaffold on main | ✅ done (under cut #1) | template every seam clones |
| P1 | Re-vendor bot covers backend seams | per-seam | extend WS-E `chore/adopt-mainspring` to backend `vendor_*.sh` |
| P2 | Lightweight per-capability parity CI gate | per-cut | run the vendored-seam unit test + fail on un-bumped `MAINSPRING_REF` drift; defer full Tier-2 platform-qa (TAA-34) until ≥2 cuts over |
| P3 | Brand-per-org + org→brand key map | at cut #6 (first persisting cut) | lazy, per-cut; synthesize 1 Brand/org, field map at the seam adapter; keep tali's alembic chain |
| P4 | Sovereign-key passthrough | at cut #7 | feed tali's resolved key into mainspring `MeteredClient.for_api_key()`; neither resolver changes |
| P5 | Deploy topology | always | stays in-place; each seam ships as ordinary tali code (commit→PR→main→auto-deploy); no data-migration cutover |

## The cuts, in order
| # | Capability | ~LoC | Risk | Status | Recipe (seam → shadow → cutover) |
|---|---|---|---|---|---|
| 1 | **Metering** (client, pricing, usage, reconciliation) | 2.7k | Low | 🟡 in progress | seam ✅ (#12) + pricing parity ✅ (#14) + shadow ✅ (#459, merged). **Remaining: enable `MAINSPRING_METERING_SHADOW` in prod → confirm parity → swap client to delegate to seam.** budget_guard rides as a sub-cut. |
| 2 | Decision-policy **verdict engine** (`decision_policy/engine.py`) | 0.8k | Med | ⬜ | vendor `PolicyEngine` Protocol; shadow each decision point through tali + mainspring engine (DomainSpec translated from `DecisionPolicyRow`); diff verdicts; cutover via `register_spec`. **Real work = row→DomainSpec field map (thresholds exact).** |
| 3 | **Promotion gate** + governance loop (fit→shadow→holdout→bias→activate→retune) | ~1.5k | Med | ⬜ | vendor governance service surface; shadow `GateResult` + activation decisions; cutover the *computation*, keep tali Celery scheduling brand-side. |
| 4 | **Bias-audit / EEOC** mechanism | 0.4k | Low-Med | ⬜ | **verify mainspring at-parity first**; split mechanism (generalizes) from hiring vocab (brand rule pack); contribute-up if not at parity. Compliance-sensitive → byte-exact. |
| 5 | **KG query side + client** (interface only) | 0.8k | Med | ⬜ | vendor `KnowledgeGraphBackend` Protocol; bind tali's live Graphiti+Neo4j+Voyage **behind** it (Option A). Store does NOT move (mainspring's store backend is the one true stub). Ingest/sync deferred to #8. |
| 6 | **HITL / decision queue** + adopt general AuditLog | 0.3k | Med-High | ⬜ | **first persisting cut → triggers P3.** Dual-write into mainspring `NeedsInput` shape, keep tali table authoritative; diff queues; cutover routes. Preserve "never lose a decision". |
| 7 | **Agent orchestrator** (per-role autonomous loop) | 0.6k | High | ⬜ | shadow mainspring orchestrator (deterministic reasoner, no actions) for a role subset; cutover per-role behind agent-on/budget flags; needs P4. **Own sub-plan.** |
| 8 | **Tool registry / harness / action-dispatch** (+ KG ingest) | ~2k | High | ⬜ | capstone. Extract generic dispatch/registration harness; **domain tools stay brand-side**, re-registered via seams. Preserve Workable write-serialization. **Own sub-plan, tool-group by tool-group.** |

Domain/brand code (recruiting tools, hiring vocab, Workable adapter, scoring, tali's tables) **never moves** — only the substrate-class spine does.

## Milestones (each independently reviewable)
- **M0** mechanism + metering scaffold on main ✅
- **M1** metering parity *confirmed in prod* (shadow drift report: zero `unpriced`, drift within tolerance)
- **M2** metering cut over — **first converged capability**
- **M3** policy engine (#2) + governance (#3) cut over
- **M4** bias-audit (#4) + KG-interface (#5) cut over
- **M5** Brand-per-org (P3) + HITL (#6) + AuditLog
- **M6** orchestrator (#7) per-role
- **M7** harness/tools (#8)
- **M8** done — Tier-2 QA wired (TAA-34), ADR-0012 conventions met, parallel copies drained, unified console realized

## Decisions (locked 2026-05-30)
1. Vendor-per-capability ORM-free seams (not pip dep). 2. In-process import (not HTTP/brain-feed).
3. KG: converge interface only (Option A), keep tali's Graphiti store. 4. Bias-audit: verify parity at #4, contribute-up if needed. 5. Parity CI: lightweight now, full Tier-2 after ≥2 cuts. 6. AuditLog: adopt going-forward, no backfill.

## Rollback
Two levers, by design. **Before cutover:** flag-gated shadow (no-op when off, exception-swallowing) → set flag false, instant. **After cutover:** the cutover is a tiny import-swap PR → `git revert` it (tali's implementation stays present — thinned, not deleted, until convergence is proven). Stores never move → no data-migration to unwind. Re-vendor drift caught by `MAINSPRING_REF` pins + CI.

## Definition of done
Taali is converged when tali-platform **imports mainspring seams for every substrate-class
capability** and the parallel local copies in `app/{agent_runtime,decision_policy,services,llm,candidate_graph}`
are drained; only brand-layer code (hiring vocab, recruiting tools, Workable, tali's tables)
remains local; substrate-class capabilities keep `container: mainspring-substrate /
migratesTo: mainspring` (boundary preserved, only HOME moved); operational surfaces unified on
the substrate incrementally; tali wired into Tier-2 platform-qa; no substrate code branches on
brand identity. Metering cutover (M2) is the *first* converged capability, not "done".
