# Methodology Review

## Scope

This review focuses on the `Optimism Standard Bridge` ERC20 path at the bridge/application layer:

- `L1 <-> L2` standard bridge contract layer;
- `deposit / withdraw` lifecycle paths;
- cross-domain contract behavior at the bridge boundary;
- `representation-token` vs `original-token` accounting semantics;
- privileged/config/init/pause surface that directly affects bridge correctness.

## Explicitly Out of Scope

The following areas were intentionally not treated as the main audit target:

- `CrossDomainMessenger` / `OptimismPortal` as a standalone infra audit;
- proving / relay / portal message delivery mechanics as a separate system review;
- deployment/factory deep-dive of the representation-token layer as a standalone infra/setup audit.

These layers are referenced only where needed to preserve lifecycle continuity.

## Review Approach

The review was organized into four layers:

- `local` review  
  Function-by-function invariants for core entrypoints and bridge internals.

- `flow-level` review  
  Full-path invariants for `deposit` and `withdraw`, covering ordering, accounting transitions, and cross-domain handoff semantics.

- `out-of-flow` review  
  Initialization, configuration, pause hooks, wrappers, and counterpart-bridge setup.

- `global` review  
  System-wide invariants that must remain true across the entire ERC20 bridge model.

## Invariant-Driven Process

Instead of only paraphrasing code, the analysis was driven through invariants:

- who is allowed to trigger paths;
- what source-side accounting must happen first;
- what remote-side payload must contain;
- how `local / remote` semantics must reverse across chain context;
- when `mint` is valid and when `release` is valid;
- how config / pause / counterpart setup must preserve bridge assumptions.

Each candidate invariant was accepted only if it could be tied directly to code behavior.

## L1/L2 Boundary Handling

The bridge flow necessarily crosses infrastructure boundaries:

- `L1 -> L2` messenger delivery segment;
- `L2 -> L1` messenger / portal delivery segment.

These segments were included as continuity boundaries in flow review, but were not expanded into a standalone transport audit. The main focus remained the bridge contract layer and the correctness assumptions it makes about those boundaries.

## AI Usage in Methodology

AI was used to accelerate:

- translation of already-established reasoning into report-ready wording;
- compression of overlapping candidate invariants;
- consistency of formatting between sections.

AI was not treated as an authority on protocol truth. Final inclusion of invariants and all scope decisions were made manually.
