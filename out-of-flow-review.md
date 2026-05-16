# Out-of-Flow Review

## Reviewed Out-of-Flow Functions

- `L1StandardBridge.initialize(...)`
- `L1StandardBridge.paused()`
- `L1StandardBridge.superchainConfig()`
- `L1StandardBridge.l2TokenBridge()`
- `L1StandardBridge.finalizeERC20Withdrawal(...)`
- `StandardBridge.__StandardBridge_init(...)`

---

## `L1StandardBridge.initialize(...)`

```solidity
function initialize(
    ICrossDomainMessenger _messenger,
    ISystemConfig _systemConfig
)
    external
    reinitializer(initVersion())
{
    // Initialization transactions must come from the ProxyAdmin or its owner.
    _assertOnlyProxyAdminOrProxyAdminOwner();

    // Now perform initialization logic.
    systemConfig = _systemConfig;
    __StandardBridge_init({
        _messenger: _messenger,
        _otherBridge: StandardBridge(payable(Predeploys.L2_STANDARD_BRIDGE))
    });
}
```

### Out-of-Flow Invariants

- `[x]` `bridge initialization path` must be restricted only to a `ProxyAdmin-authorized caller`
- `[x]` `initialize(...)` must bind the `L1 bridge` to the correct local `systemConfig`
- `[x]` `initialize(...)` must set the correct `messenger` for the base `StandardBridge`
- `[x]` `initialize(...)` must set the correct counterpart `L2 bridge` for the base `StandardBridge`

### Conclusion

This is a setup/config boundary, not a lifecycle bridge path. Its security value comes from correct authorization and correct binding of bridge-critical dependencies.

---

## `L1StandardBridge.paused()`

```solidity
function paused() public view override returns (bool) {
    return systemConfig.paused();
}
```

### Out-of-Flow Invariants

- `[x]` `bridge pause-state` must be read from the linked `systemConfig`, not from an arbitrary hidden source

### Conclusion

This is a narrow config accessor, but it matters because `finalize` paths rely on it to block execution in the paused state.

---

## `L1StandardBridge.superchainConfig()`

```solidity
function superchainConfig() public view returns (ISuperchainConfig) {
    return systemConfig.superchainConfig();
}
```

### Out-of-Flow Invariants

- `[x]` `superchain config access` must resolve through the already-bound `systemConfig`

### Conclusion

This is a simple accessor with little standalone logic, but it reflects the intended config hierarchy.

---

## `L1StandardBridge.l2TokenBridge()`

```solidity
function l2TokenBridge() external view returns (address) {
    return address(otherBridge);
}
```

### Out-of-Flow Invariants

- `[x]` the public `counterpart-bridge view` must reflect the already-configured `otherBridge`

### Conclusion

This function exposes counterpart-bridge configuration rather than implementing bridge flow logic.

---

## `L1StandardBridge.finalizeERC20Withdrawal(...)`

```solidity
function finalizeERC20Withdrawal(
    address _l1Token,
    address _l2Token,
    address _from,
    address _to,
    uint256 _amount,
    bytes calldata _extraData
)
    external
{
    finalizeBridgeERC20(_l1Token, _l2Token, _from, _to, _amount, _extraData);
}
```

### Out-of-Flow Invariants

- `[x]` `finalizeERC20Withdrawal(...)` must pass withdrawal finalize semantics into `finalizeBridgeERC20(...)` without rewriting arguments

### Conclusion

This is a legacy/public wrapper over the shared finalize core.

---

## `StandardBridge.__StandardBridge_init(...)`

```solidity
function __StandardBridge_init(
    ICrossDomainMessenger _messenger,
    StandardBridge _otherBridge
)
    internal
    onlyInitializing
{
    messenger = _messenger;
    otherBridge = _otherBridge;
}
```

### Out-of-Flow Invariants

- `[x]` `__StandardBridge_init(...)` must bind the bridge to the correct `cross-domain messenger`
- `[x]` `__StandardBridge_init(...)` must bind the bridge to the correct `counterpart bridge`

### Conclusion

This is the base setup primitive that wires the bridge into cross-domain routing and counterpart-bridge semantics.
