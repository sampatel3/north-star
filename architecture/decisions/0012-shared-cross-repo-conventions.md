# 12. Shared cross-repo conventions

- **Status:** Accepted
- **Date:** 2026-05-30
- **Deciders:** Sam
- **Related:** Extends **ADR-0002** (substrate/brand boundary) with a *layout/process* convention, and supports **ADR-0007/0008** (agent injection) by giving every brand repo a predictable docs and CI shape for agents to rely on.

## Context

The Phase 5 audit (`AUDIT_05 §5`, `[P5-03]`, MED) found the three repos have **diverged into three
dialects** across every dimension, and that this compounds as a per-brand onboarding tax and a
drift-accelerant when brand #3 lands:

- **Layout:** `mainspring` = installable `mainspring/{core,platform}`; cadence = thin brand package
  + `deploy_vendor`; `tali-platform` = `backend/app/domains/*` monolith. Three conventions.
- **Tests:** `mainspring` ~112 files (stdlib-fast); cadence 6 files (no-DB, <1s); `tali-platform`
  large suite, flaky-in-batch, run-in-isolation. Different volume, style, **and run-command**.
- **Docs:** wildly uneven — `mainspring` 14 rich docs; `tali-platform` ~25; **cadence `docs/`
  empty** (everything lives in README + design-system).
- **CI:** two philosophies — `mainspring` + cadence wire into the Tier-2 `platform-qa` gate;
  `tali-platform` runs its own CI and is excluded from Tier-2.
- **Deploy:** three shapes — `mainspring` multi-Dockerfile + brain image; cadence vendored single
  image; `tali-platform` Railway web+workers.

No shared convention exists across the repos. With each new brand the divergence multiplies and the
cost of onboarding (human or agent) rises.

## Decision

We adopt a **minimal shared convention** across brand repos and have the brand scaffolder
(`mainspring init`) emit it for every new brand:

1. **Package layout.** A predictable top-level shape every brand repo follows (an installable brand
   package on the substrate, not a bespoke per-repo monolith), so an agent or engineer can navigate
   any brand repo by the same map.
2. **A single test-run command.** Every brand repo exposes the *same* command to run its tests
   (regardless of suite size or DB needs), so "how do I run the tests here?" has one answer across
   the fleet.
3. **A required docs skeleton.** Every brand repo ships a minimum docs set — including an
   **`ARCHITECTURE.md`** and a **lineage header** (stating the repo's role and its relationship to
   the substrate / central north-star, as the per-product `NORTH_STAR.md` files already do). This
   directly remedies cadence's empty `docs/`.
4. **The Tier-2 `platform-qa` CI hook.** Every brand repo wires into the Tier-2 `platform-qa` gate
   (not only its own per-repo CI), so the cross-repo contract covers every live brand — closing the
   `tali-platform`-excluded-from-Tier-2 dialect.

These four are deliberately **minimal** — layout, one test command, a docs skeleton, the Tier-2
hook — not a heavyweight monorepo standard. The brand scaffolder (`mainspring init`) emits this
skeleton at brand-creation time so new brands start conformant rather than being retrofitted.

## Consequences

- **Brand #3 starts conformant.** The scaffolder emits the convention, so the onboarding tax and
  drift-accelerant the audit flagged are paid down at creation time, not after the fact.
- **Agents get a predictable map.** A single layout, one test command, and a guaranteed
  `ARCHITECTURE.md` + lineage header mean agents (ADR-0007/0008) can orient in any brand repo the
  same way — the docs skeleton is load-bearing for agent onboarding, not cosmetic.
- **Retrofit cost on existing repos.** The three current dialects (`mainspring`, cadence,
  `tali-platform`) do not conform yet; bringing them onto the shared convention is incremental work
  — especially `tali-platform`'s monolith layout and its Tier-2 exclusion. Accepted: convention is
  enforced going forward (scaffolder) and existing repos converge when touched.
- **Convention, not straitjacket.** The shared set is intentionally minimal so brands keep latitude
  on deploy shape and internal structure beyond the four points; we standardize the seams that
  matter for cross-repo navigation and QA, not every choice.
