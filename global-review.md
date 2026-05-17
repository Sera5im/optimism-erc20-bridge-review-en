# Global Review

## Global Invariants

- `[x]` the bridge system must preserve separation between `original-token accounting` and `representation-token accounting`
  Why it matters: the custody-backed original asset path and the synthetic representation path must not collapse into one accounting model.

- `[x]` standard `ERC20 deposit` and `withdraw` must remain mirror lifecycle paths
  Why it matters: the canonical bridge model depends on the symmetry `lock -> message -> mint` and `burn -> message -> release`.

- `[x]` cross-domain `ERC20 completion` must happen only through `bridge -> counterpart bridge` semantics
  Why it matters: destination-side credit or release must not be triggered by an arbitrary local caller outside the bridge-to-bridge handoff model.

- `[x]` `local/remote token semantics` must remain relative to the current chain and reverse correctly between sides
  Why it matters: the same token pair must be interpreted in a chain-relative way, otherwise the finalize payload may credit or release the wrong side of the pair.

- `[x]` the bridge system must not allow `asset credit / release` without a correct `source-side accounting step`
  Why it matters: remote-side delivery must not exist without a prior local `burn` or `lock` step.

- `[x]` the `paused/config layer` must remain able to stop the `finalize completion path`
  Why it matters: the emergency control surface must be able to block destination-side completion if bridge assumptions are no longer considered safe.

## Global Reasoning

In the reviewed standard `Optimism` ERC20 bridge model, global coherence depends on several core properties:

- the original-token branch uses custody plus internal deposit accounting;
- the representation-token branch uses burn/mint accounting;
- counterpart-gated finalization closes destination-side credit/release only after a bridge-to-bridge handoff;
- `local` and `remote` token arguments are always interpreted relative to the current chain context and reversed during message construction;
- the pause/config surface remains able to stop the completion path before unsafe delivery continues.

If these properties remain intact, the standard model continues to behave like a symmetric canonical bridge lifecycle.

## Global Conclusion

At the global level, the reviewed `Optimism` ERC20 bridge model remains coherent if the following conditions stay intact:

- source-side accounting always happens before destination-side asset movement;
- `original` and `representation` token models are not mixed inside the same accounting branch;
- counterpart-gated finalization remains enforced;
- config / init / pause surfaces do not break the core bridge assumptions.

In the standard model this produces a clean symmetry:

- `deposit = lock -> message -> mint`
- `withdraw = burn -> message -> release`

That symmetry is the main global security shape of the reviewed ERC20 bridge path.
