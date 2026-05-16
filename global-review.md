# Global Review

## Global Invariants

- `[x]` the bridge system must preserve separation between `original-token accounting` and `representation-token accounting`
- `[x]` standard `ERC20 deposit` and `withdraw` must remain mirror lifecycle paths
- `[x]` cross-domain `ERC20 completion` must happen only through `bridge -> counterpart bridge` semantics
- `[x]` `local/remote token semantics` must remain relative to the current chain and must reverse correctly when moving between sides
- `[x]` the bridge system must not allow `asset credit / release` without a correct `source-side accounting step`
- `[x]` the `paused/config layer` must remain able to stop the `finalize completion path`

## Global Conclusion

At the global level, the reviewed ERC20 bridge model remains coherent if the following conditions stay intact:

- source-side accounting always happens before destination-side asset movement;
- `original` and `representation` token models are not mixed inside the same accounting branch;
- counterpart-gated finalization remains enforced;
- config / init / pause surfaces do not break bridge assumptions.

In the reviewed standard model this produces a clean symmetry:

- `deposit = lock -> message -> mint`
- `withdraw = burn -> message -> release`
