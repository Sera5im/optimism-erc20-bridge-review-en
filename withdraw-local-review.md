# Withdraw Local Review

## Reviewed Local Functions

- `L2StandardBridge.withdraw(...)`
- `L2StandardBridge._initiateWithdrawal(...)`
- `StandardBridge._initiateBridgeERC20(...)`
- `StandardBridge.finalizeBridgeERC20(...)`
- `L1StandardBridge.finalizeERC20Withdrawal(...)`

---

## `L2StandardBridge.withdraw(...)`

```solidity
function withdraw(
    address _l2Token,
    uint256 _amount,
    uint32 _minGasLimit,
    bytes calldata _extraData
)
    external
    payable
    virtual
    onlyEOA
{
    _initiateWithdrawal(_l2Token, msg.sender, msg.sender, _amount, _minGasLimit, _extraData);
}
```

### Local Invariants

- `[x]` direct withdraw path must be restricted to `EOA` callers
- `[x]` self-withdraw path must not change caller semantics: the caller must remain both sender and recipient
- `[x]` `withdraw(...)` must forward the declared token and withdraw parameters into the internal path without rewriting them

### Local Conclusion

This is a user-facing self-withdraw wrapper. It defines correct caller semantics before handing control into the internal withdrawal path.

---

## `L2StandardBridge._initiateWithdrawal(...)`

```solidity
function _initiateWithdrawal(
    address _l2Token,
    address _from,
    address _to,
    uint256 _amount,
    uint32 _minGasLimit,
    bytes memory _extraData
)
    internal
{
    if (_l2Token == Predeploys.LEGACY_ERC20_ETH) {
        _initiateBridgeETH(_from, _to, _amount, _minGasLimit, _extraData);
    } else {
        address l1Token = OptimismMintableERC20(_l2Token).l1Token();
        _initiateBridgeERC20(_l2Token, l1Token, _from, _to, _amount, _minGasLimit, _extraData);
    }
}
```

### Local Invariants

- `[x]` withdraw path must explicitly separate `ETH` and `ERC20` models
- `[x]` `non-ETH withdraw path` must derive the `L1 token pair` from the `L2 token representation` itself
- `[x]` `ERC20 withdraw handoff` must pass `source`, `destination`, and withdraw parameters into the core bridge path without rewriting them

### Local Conclusion

This function is the branch-selection layer. For the standard ERC20 model it derives the `L1` pair directly from the `L2 representation token` and then routes execution into the shared bridge core.

---

## `StandardBridge._initiateBridgeERC20(...)` in Withdraw Context

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
- `[x]` `ERC20 bridge initiation` must send a message only to the `counterpart bridge`
- `[x]` `ERC20 bridge initiation` must reverse `local/remote token arguments` for the remote-side context

### Local Conclusion

In the standard withdraw model this is the source-side `burn` boundary. It closes `L2` accounting and constructs the counterpart-targeted handoff into the `L1` finalize path.

---

## `StandardBridge.finalizeBridgeERC20(...)` in Withdraw Context

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
- `[x]` `ordinary token finalize branch` must decrement internal `deposit accounting` before `release`
- `[x]` `ordinary token finalize branch` must complete destination-side delivery via `release existing tokens`, not `mint`
- `[x]` `ERC20 finalize path` must emit a `bridge-finalized event` with the same semantics that were actually finalized

### Local Conclusion

For the standard withdraw model this function closes the destination-side `L1` release path. In the expected branch it reduces accounting and releases already-escrowed original tokens.

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

### Local Invariants

- `[x]` `finalizeERC20Withdrawal(...)` must pass withdrawal finalize semantics into `finalizeBridgeERC20(...)` without rewriting arguments

### Local Conclusion

This is a thin wrapper used for legacy/public compatibility. It does not implement a separate finalize model; it forwards directly into the shared bridge finalize core.
