# 9. `tali-platform` is the live Taali brand (not legacy); `taali-brand` is retired

- **Status:** Accepted (ratified 2026-05-30)
- **Date:** 2026-05-30
- **Deciders:** Sam
- **Supersedes / Superseded by:** Amends **ADR-0002** (its closing "tali-platform is legacy" note) and **ADR-0008** (role assignment for `tali-platform`). Does not change the substrate/brand boundary itself.

## Context

The North Star currently encodes a topology that no longer matches reality:

- `NORTH_STAR.md` says *"`tali-platform` is legacy"*; `model.yaml` lists `taali-brand` as `role: brand` ("Taali Surface") and `tali-platform` as `role: legacy` ("Legacy Backend"); `agent/sync.config.json` ships `taali-brand` the `brand` block and `tali-platform` the `legacy` block.
- The live reality (confirmed by direct inspection on 2026-05-30, and by SUPERSEDED banners added across `mainspring/docs/` on 2026-05-29) is the inverse: **`tali-platform` is the live Taali product** (the hiring brand, in production), and **`taali-brand` is a throwaway validation harness slated for deletion** — never deployed.
- The model already half-admits this: `taali-brand` ("Taali Surface") carries **no `implementation` paths** — it is declared but never verified — while every invariant-bearing component (`metered-anthropic-client`, `decision-engine`, `brand-migrations`, `billing`) points its real `implementation.repo` at **`tali-platform`** with `migratesTo: mainspring`. `tali-platform` is the drift checker's only verifiable Taali anchor.
- Cross-cutting QA inherits the stale model: `platform-qa` registers `BRANDS = ("taali", "cadence")` where `"taali"` resolves to `taali-brand`, and its `model.yaml` marks `tali-platform` "FROZEN … never cloned in CI." So the harness contracts against the disposable repo and never exercises the live one.
- Sam has now authored per-product north-stars and placed `tali-platform/NORTH_STAR.md` as *Taali's* product north-star — making the "tali-platform is the live Taali" decision explicit. Per the North Star's amendment rule, that flip is ratified here rather than left as drift.

A necessary nuance: `tali-platform` is **both** the live Taali brand surface **and** the current host of un-migrated, substrate-class capabilities (the metered client, decision engine, billing, the agentic decision-policy stack). Reclassifying the repo must not erase that migration debt — those capabilities are still cross-cutting and still belong in `mainspring` (ADR-0002, ADR-0003/0004/0005). The relabel changes the repo's *role*, not the *home* those components should migrate to.

## Decision

We will reconcile the model to the live topology:

1. **`tali-platform` is the live Taali brand**, not legacy. Its role becomes `brand`. It owns the hiring surface, hiring-specific data + migrations, and — for now — un-migrated substrate-class capabilities that remain on the convergence queue to `mainspring`.
2. **`taali-brand` is retired.** Its role becomes `deprecated` (throwaway validation harness, pending deletion). It is no longer a live target, no longer the migration vehicle, and is removed from active QA/sync.
3. **The substrate/brand boundary (ADR-0002) is unchanged.** The invariant-bearing capabilities currently hosted in `tali-platform` keep `container: mainspring-substrate` and `migratesTo: mainspring`; their `implementation.repo` stays `tali-platform` because that is where the code lives today. The migration debt is named explicitly, not relabelled away.
4. **`tali-platform` carries a standing "drain to substrate" directive.** As the live Taali, it grows new *brand* surface; as the host of substrate-class code, it migrates that code into `mainspring` when touched — and the Taali north-star principle *"fair-hiring posture is inherited from Mainspring, not bolted on"* sets inheritance (not local reimplementation) as the target end-state.

## Consequences

**Easier:** the model stops contradicting reality; the drift checker's verifiable anchor (`tali-platform`) is now correctly typed; QA can be pointed at the live brand; agents in `tali-platform` receive guidance that matches what the repo actually is.

**Harder / trade-offs accepted:** `tali-platform` becomes a *hybrid* (live brand + substrate-debt host), which is messier than a clean brand. We accept that, because pretending the substrate code already lives in `mainspring` (it does not — `tali-platform` has 0 `mainspring` imports in its backend) would be the dishonest alternative. The convergence remains real work, now tracked against a correctly-labelled repo.

### Migration-debt inventory (the convergence queue) — TAA-10

The substrate-class capabilities `tali-platform` hosts locally and must drain into `mainspring`
(`container: mainspring-substrate`, `migratesTo: mainspring`), with the audit's LoC estimate:

| Capability | tali-platform path | ~LoC | mainspring target |
|---|---|---|---|
| Metering (MeteredAnthropicClient + wire-tap + reconciliation) | `backend/app/llm/core.py` + `services/metered_*.py` | ~2,690 | `platform/metering/` (at-parity gateway) — **highest-value, most self-contained first cut** |
| Decision-policy engine | `backend/app/decision_policy/engine.py` | ~5,295 (with governance) | `core/policy.py` |
| Policy governance loop (fit → shadow → promotion gate → bias audit → retune) | `backend/app/decision_policy/{nightly_policy_fit,promotion_gate,shadow_mode,nightly_retune,bias_audit}.py` | (part of ↑) | `platform/services/{promotion_gate,shadow_runner,holdout_eval,bias_audit,fitted_policy}.py` |
| Agent runtime (orchestrator + tool registry + stores) | `backend/app/agent_runtime/` | ~6,450 | `platform/agent/` + a tool-registration seam mainspring must expose |
| EEOC / bias-audit content | `backend/app/decision_policy/bias_audit.py` | ~254 | `platform/services/bias_audit.py` + a hiring rule pack |
| Sub-agent harness / worker / action-dispatch | `backend/app/{sub_agents,tasks,actions}/` | ~3–4k (harness) | `platform/agent/subagents.py`, `platform/workers/`, `platform/services/action_dispatcher.py` |

`candidate_graph/` is **mostly brand-local** (only ~266 LoC is generic substrate) — it is **not** on the debt list. Total hard substrate-class debt ≈ **14–17k LoC**, multi-quarter; mainspring does not yet expose the plug-in seams (tool registration, policy-engine plugin, metering-gateway dependency surface, KG package, agent-runtime loop). Strategy is tracked in **TAA-29**.

**Implementation of acceptance:**

- **✅ `model.yaml`** (this PR): `taali-brand.role: deprecated`; `tali-platform.role: brand`; `legacy-backend` → `taali-backend`; invariant components keep `implementation.repo: tali-platform` + `migratesTo: mainspring`; `metadata.updated` bumped.
- **✅ `NORTH_STAR.md`** (this PR): the "legacy" line now reads "live Taali brand … migration debt into `mainspring`", pointing here.
- **✅ `agent/sync.config.json`** (this PR): `taali-brand` dropped; `tali-platform` role `legacy` → `brand`. *(Running the sync into the repos' `CLAUDE.md` remains — audit GAP #3.)*
- **⏳ `platform-qa`** (TAA-34): repoint `"taali"` → `tali-platform` (`brands/__init__.py` + `contracts/architecture/model.yaml`), give it a GitHub slug, regenerate `contracts/taali/substrate.json`, set `REPOS_TOKEN`. Until then the cross-repo gate does not cover the live Taali.
- **⏳ `ADR-0002`:** annotate its closing legacy note with a pointer here.

**Guardrail:** `scripts/check_drift.py` / `validate_model.py` must stay green after these edits (the `tali-platform` implementation paths are unchanged; only roles/descriptions moved).
