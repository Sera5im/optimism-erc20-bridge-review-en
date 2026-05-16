# Deposit Local Review

## Reviewed Local Functions

- `L1StandardBridge.depositERC20(...)`
- `L1StandardBridge.depositERC20To(...)`
- `L1StandardBridge._initiateERC20Deposit(...)`
- `StandardBridge._initiateBridgeERC20(...)`
- `StandardBridge.finalizeBridgeERC20(...)`

---

## `L1StandardBridge.depositERC20(...)`

```solidity
function depositERC20(
    address _l1Token,
    address _l2Token,
    uint256 _amount,
    uint32 _minGasLimit,
    bytes calldata _extraData
)
    external
    virtual
    onlyEOA
{
    _initiateERC20Deposit(_l1Token, _l2Token, msg.sender, msg.sender, _amount, _minGasLimit, _extraData);
}
```

### Local Invariants

- `[x]` direct deposit path must be restricted to `EOA` callers
- `[x]` self-deposit path must not change caller semantics: the caller must remain both sender and recipient
- `[x]` `depositERC20(...)` must forward the declared token pair and deposit parameters into the internal `ERC20` path without rewriting them

### Local Conclusion

This is a user-facing self-deposit wrapper. It does not carry the main bridge logic itself; it only defines correct entry semantics before handing control into the internal deposit path.

---

## `L1StandardBridge.depositERC20To(...)`

```solidity
function depositERC20To(
    address _l1Token,
    address _l2Token,
    address _to,
    uint256 _amount,
    uint32 _minGasLimit,
    bytes calldata _extraData
)
    external
    virtual
{
    _initiateERC20Deposit(_l1Token, _l2Token, msg.sender, _to, _amount, _minGasLimit, _extraData);
}
```

### Local Invariants

- `[x]` `deposit-to` path must preserve `caller` as `source` and `_to` as the explicit `destination`
- `[x]` `depositERC20To(...)` must forward the declared token pair and deposit parameters into the internal `ERC20` path without rewriting them

### Local Conclusion

This is a generalized deposit wrapper where the sender remains the source, but the recipient may be a different address on the remote chain.

---

## `L1StandardBridge._initiateERC20Deposit(...)`

```solidity
function _initiateERC20Deposit(
    address _l1Token,
    address _l2Token,
    address _from,
    address _to,
    uint256 _amount,
    uint32 _minGasLimit,
    bytes memory _extraData
)
    internal
{
    _initiateBridgeERC20(_l1Token, _l2Token, _from, _to, _amount, _minGasLimit, _extraData);
}
```

### Local Invariants

- `[x]` `_initiateERC20Deposit(...)` must converge the external `ERC20 deposit` path into a single core bridge path

### Local Conclusion

This is a narrow handoff layer with no standalone accounting logic. Its role is to route the deposit path into the shared bridge core.

---

## `StandardBridge._initiateBridgeERC20(...)`

```solidity
function _initiateBridgeERC20(
    address _localToken,
    address _remoteToken,
    address _from,
    address _to,
    uint256 _amount,
    uint32 _minGasLimit,
    bytes memory _extraData
)
    internal
{
    require(msg.value == 0, "StandardBridge: cannot send value");

    if (_isOptimismMintableERC20(_localToken)) {
        require(
            _isCorrectTokenPair(_localToken, _remoteToken),
            "StandardBridge: wrong remote token for Optimism Mintable ERC20 local token"
        );

        IOptimismMintableERC20(_localToken).burn(_from, _amount);
    } else {
        IERC20(_localToken).safeTransferFrom(_from, address(this), _amount);
        deposits[_localToken][_remoteToken] = deposits[_localToken][_remoteToken] + _amount;
    }

    _emitERC20BridgeInitiated(_localToken, _remoteToken, _from, _to, _amount, _extraData);

    messenger.sendMessage({
        _target: address(otherBridge),
        _message: abi.encodeWithSelector(
            this.finalizeBridgeERC20.selector,
            _remoteToken,
            _localToken,
            _from,
            _to,
            _amount,
            _extraData
        ),
        _minGasLimit: _minGasLimit
    });
}
```

### Local Invariants

- `[x]` `ERC20 bridge initiation path` must not accept `ETH value`
- `[x]` `mintable local token path` must confirm the correct `local/remote token pair` before `burn`
- `[x]` `mintable local token path` must complete source-side accounting via `burn`, not `escrow`
- `[x]` `ordinary ERC20 path` must complete source-side accounting via `transfer-in` and internal `deposit accounting`
- `[x]` `ERC20 bridge initiation` must emit a `bridge-initiated event` with the same `token/from/to/amount` semantics that actually went into the message path
- `[x]` `ERC20 bridge initiation` must send a message only to the `counterpart bridge`
- `[x]` `ERC20 bridge initiation` must reverse `local/remote token arguments` when building the `finalize payload` for the remote-side context

### Local Conclusion

This is the main local boundary for the deposit path. It closes source-side accounting, emits the initiation event, and constructs the cross-domain handoff toward the remote finalize path.

---

## `StandardBridge.finalizeBridgeERC20(...)`

```solidity
function finalizeBridgeERC20(
    address _localToken,
    address _remoteToken,
    address _from,
    address _to,
    uint256 _amount,
    bytes calldata _extraData
)
    public
    onlyOtherBridge
{
    require(paused() == false, "StandardBridge: paused");
    if (_isOptimismMintableERC20(_localToken)) {
        require(
            _isCorrectTokenPair(_localToken, _remoteToken),
            "StandardBridge: wrong remote token for Optimism Mintable ERC20 local token"
        );

        IOptimismMintableERC20(_localToken).mint(_to, _amount);
    } else {
        deposits[_localToken][_remoteToken] = deposits[_localToken][_remoteToken] - _amount;
        IERC20(_localToken).safeTransfer(_to, _amount);
    }

    _emitERC20BridgeFinalized(_localToken, _remoteToken, _from, _to, _amount, _extraData);
}
```

### Local Invariants

- `[x]` `ERC20 finalize path` must be restricted only to the `counterpart bridge`
- `[x]` `ERC20 finalize path` must not execute in `paused-state`
- `[x]` `mintable local token finalize branch` must confirm the correct `local/remote token pair` before `mint`
- `[x]` `mintable local token finalize branch` must complete destination-side credit via `mint`, not `release existing tokens`
- `[x]` `ordinary token finalize branch` must decrement internal `deposit accounting` before `release`
- `[x]` `ordinary token finalize branch` must complete destination-side delivery via `release existing tokens`, not `mint`
- `[x]` `ERC20 finalize path` must emit a `bridge-finalized event` with the same `token/from/to/amount` semantics as those that were actually finalized

### Local Conclusion

This function closes the destination-side deposit path with counterpart-gated completion logic and correct branch selection between `representation mint` and `original-token release`.
