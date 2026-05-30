# 10. Taali↔Mainspring convergence strategy

- **Status:** Accepted
- **Date:** 2026-05-30
- **Deciders:** Sam
- **Related:** Builds on **ADR-0009** (which names `tali-platform` the live Taali brand and inventories the migration debt) and **ADR-0002** (the substrate/brand boundary the migration must honour).

## Context

`tali-platform` is the live Taali brand (ADR-0009), but it is **not** connected to
`mainspring` in any meaningful runtime sense. The Phase 3 audit (`AUDIT_03 §2`) verified
**0** `import mainspring` across ~101.7k LoC of Taali backend, a flag-off / severed
brain-feed wire, and a frontend tie that is a vendored CSS-plus-one-React-component copy.
Taali runs its own parallel copy of substrate-class capabilities rather than inheriting them.

That parallel stack is **confirmed substrate-class debt, not brand code**: in each case
`mainspring` has already independently built the generalized version (its `TAALI_PARITY.md`
records exactly these as "generalised from Taali"). The hard substrate-class duplication is
≈ **14–17k LoC** (`AUDIT_03 §2`, `[P3-TALI-03]`), after the audit downgraded
`candidate_graph/` — only ~266 LoC there is generic, the rest is hiring vocabulary, so it is
**not** on the debt list. The detailed line-item inventory lives in **ADR-0009** (the
migration-debt table, TAA-10).

Two facts make this a **multi-quarter program, not a migration**:

1. The volume (14–17k LoC across agent runtime, decision-policy engine + governance loop,
   metering, bias/EEOC audit, and the sub-agent/worker/action harness).
2. **`mainspring` does not yet expose the plug-in seams Taali would need** to inherit rather
   than re-host: a tool-registration seam, a policy-engine plugin seam, a metering-gateway
   dependency surface, a KG package, and an agent-runtime loop. Today `mainspring` carries only
   cadence — a genuinely thin brand — as proof of the inheritance pattern.

A decision is required (TAA-29): up-stream the seams then migrate incrementally, parallel-prod
indefinitely (accept the duplication), or defer with eyes open. This is the single biggest
"are Taali and Mainspring tightly connected?" lever, so it is recorded here rather than left
implicit.

## Decision

We will **converge Taali onto the substrate incrementally, seams-first**:

1. **`mainspring` exposes the plug-in seams first.** Before any Taali code moves, `mainspring`
   must ship the dependency surfaces a brand plugs into: tool registration, a policy-engine
   plugin seam, a metering-gateway dependency surface, a KG package, and the agent-runtime loop.
   You cannot inherit what the substrate does not yet offer.
2. **Then migrate Taali capability-by-capability, starting with the most self-contained,
   highest-value cut: METERING.** `backend/app/llm/core.py` imports nothing from `app`
   (verified, `AUDIT_03 §2`) and `mainspring` already ships an at-parity metering gateway, so
   metering is the lowest-risk, highest-leverage first cut. Each subsequent capability lands only
   once its seam exists and an at-parity target is in place.
3. **We do not parallel-prod indefinitely and we do not rewrite Taali wholesale.** The duplication
   is acknowledged debt with a draining plan, not a permanent fork; convergence proceeds one
   verifiable cut at a time.
4. **The ~14–17k-LoC backlog and the seam list are tracked, not relabelled away.** The line-item
   inventory is ADR-0009's migration-debt table (TAA-10); ordering and sequencing of cuts is
   tracked under TAA-29. The substrate/brand boundary (ADR-0002) is unchanged — these capabilities
   keep `container: mainspring-substrate` / `migratesTo: mainspring`; only their *home* moves.

## Consequences

- **A clear, honest ordering.** Metering first (self-contained, at-parity target ready), seams
  before bodies. The program has a defined first step and a defined gate (seam exists + at-parity)
  for every subsequent step.
- **Multi-quarter, accepted.** This is explicitly a multi-quarter program. Until it lands, Taali
  remains a standalone product that periodically copies a stylesheet from `mainspring`; the holdco
  "tightly connected" claim is a target, not a present-tense fact (see ADR-0011 for the
  operational-view corollary).
- **Front-loaded substrate work.** The substrate team carries the up-front cost of exposing seams
  before Taali sees any benefit; cadence (the thin brand already on the substrate) is the proving
  ground for each seam before Taali plugs in.
- **Reversible if a cut proves wrong.** Because cuts are incremental and gated on at-parity, a
  capability that does not generalize cleanly can be held at the brand boundary without derailing
  the rest of the program.
