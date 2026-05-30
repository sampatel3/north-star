# 11. Federated per-brand operational view until convergence

- **Status:** Accepted
- **Date:** 2026-05-30
- **Deciders:** Sam
- **Related:** The operational corollary of **ADR-0010** (Taali↔Mainspring convergence) and **ADR-0009** (Taali-not-on-substrate). The boundary it operates within is **ADR-0002**.

## Context

The holdco narrative is *"one unified operational view / audit trail / console across every
brand."* The Phase 5 audit (`AUDIT_05 §4`, `[P5-02]`, HIGH) verified that **this does not hold
today**:

| Surface | State (verified) |
|---|---|
| Single audit log? | ❌ No — per-brand silos: `mainspring`'s `AuditEvent` / Audit Explorer vs `tali-platform`'s own audit |
| Single budget view? | ❌ No — two separate cost/budget governors: `mainspring`'s `routes/cost.py` + metering (which cadence rides) vs `tali-platform`'s own |
| Single Decision Hub? | ❌ No — two separate consoles: `tali-platform` runs its own agentic Hub (`backend/app/domains/agentic/hub_routes.py` → `/home`); cadence rides `mainspring`'s console (`/hub /audit /governance /cost`) |

This is the operational face of the same root cause as Phase 3: **Taali isn't on the substrate,
so it isn't on the substrate's operational surfaces either.** Cadence *does* demonstrate the
unified model — it lives entirely on `mainspring`'s console, audit log, and budget governor — so
the unified design is real and proven; it is just not yet realized *across all brands* because
the live Taali is a separate operational island.

The risk this ADR addresses is a **narrative-versus-reality** one: asserting "one unified console
across brands" as a shipped capability when it is an unrealized target. That would be an overclaim
the audit explicitly flags.

## Decision

We record that, **until Taali converges onto the substrate (ADR-0010), the holdco operates a
FEDERATED per-brand operational view:**

1. **Federated is the present-tense reality and the accepted interim posture.** Each brand has its
   own operational surfaces where it is not yet on the substrate: separate audit logs, separate
   budget/cost governors, and separate Decision Hubs. Concretely today: **Taali-on-its-own-console
   + cadence-on-mainspring's-console.**
2. **The "one unified console / audit / budget across brands" is the POST-CONVERGENCE target, not
   a present claim.** It must **not** be asserted as shipped — in decks, docs, or to partners —
   while Taali runs its own operational surfaces. The unified view becomes a present-tense fact
   only as brands move onto `mainspring`'s surfaces (per ADR-0010).
3. **Cadence is the existing proof of the unified model.** It already runs on `mainspring`'s
   console, audit log, and budget governor, demonstrating that the unified operational design works
   — the gap is coverage (the live Taali), not design.
4. **Convergence is the path to unification.** The unified operational view is achieved by the same
   work as ADR-0010: as Taali's capabilities (metering first) drain onto the substrate, Taali's
   operational surfaces follow onto `mainspring`'s console/audit/budget. There is no separate
   "build a federation layer" track — federation is the *interim accepted state*, not an
   architecture to invest in.

## Consequences

- **Honesty in the holdco story.** We can say truthfully: "cadence proves the unified model on
  Mainspring's surfaces; Taali is a federated island today; one console across all brands is the
  post-convergence target." We do not claim a unified console as already shipped.
- **No throwaway federation layer.** We deliberately do **not** build a cross-brand
  audit/budget/console aggregation layer as a permanent solution — that would entrench the split we
  intend to remove. The fix is convergence (ADR-0010), not federation tooling.
- **Operational duplication, accepted for now.** Two audit implementations, two budget governors,
  two Decision Hubs is real operational overhead and a real "which console?" cost during the
  interim. Accepted as the cost of the multi-quarter convergence timeline.
- **A clear unification trigger.** Each Taali capability that lands on the substrate moves its
  operational surface with it; the unified-view claim becomes true incrementally, not by
  announcement.
