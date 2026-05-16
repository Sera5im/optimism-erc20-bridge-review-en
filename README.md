# Optimism Standard Bridge ERC20 Review

Author: `Sera5im`  
Date: `2026-05-16`

## AI Usage

AI was used as an audit copilot during this review:

- to speed up wording refinement;
- to help structure the review into local, flow, out-of-flow, and global layers;
- to challenge or compress candidate invariants after manual reasoning had already been formed.

AI was not used as a substitute for protocol understanding. The core reasoning, scope control, and final security interpretation were driven manually.

## What This Review Is

This repository contains an audit-style review of the `Optimism Standard Bridge` ERC20 path. The focus is on the bridge/application layer:

- `deposit` and `withdraw` lifecycle paths;
- cross-domain contract logic at the bridge layer;
- `mint / burn / lock / release` accounting semantics;
- privileged/config/init surface relevant to the bridge path.

This review does **not** attempt to be a full standalone audit of the entire messenger / portal / proving infrastructure.

## Review Structure

- [Methodology Review](./methodology-review.md)
- [Deposit Local Review](./deposit-local-review.md)
- [Deposit Flow Review](./deposit-flow-review.md)
- [Withdraw Local Review](./withdraw-local-review.md)
- [Withdraw Flow Review](./withdraw-flow-review.md)
- [Out-of-Flow Review](./out-of-flow-review.md)
- [Global Review](./global-review.md)

## Scope Boundary

Primary scope:

- `L1StandardBridge`
- `L2StandardBridge`
- `StandardBridge`
- `ERC20 deposit / withdraw` paths
- config/init/pause surface that directly affects bridge correctness

Transport boundary:

- `messenger.sendMessage(...)`
- `L1CrossDomainMessenger / L2CrossDomainMessenger`
- `OptimismPortal`
- proving / relay / portal execution path

Transport-layer continuity is acknowledged where required for end-to-end path understanding, but the deep transport infrastructure is treated as outside the main review scope.
