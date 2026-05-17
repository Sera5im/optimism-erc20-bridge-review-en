# Withdraw Local Review

## Reviewed Local Functions

- `L2StandardBridge.withdraw(...)`
- `L2StandardBridge._initiateWithdrawal(...)`
- `StandardBridge._initiateBridgeERC20(...)`
- `StandardBridge.finalizeBridgeERC20(...)`
- `L1StandardBridge.finalizeERC20Withdrawal(...)`

---

## 1. `L2StandardBridge.withdraw(...)`

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

### What It Does

This is the user-facing self-withdraw wrapper. It opens the direct `L2 -> L1` withdrawal path where the caller remains both the source and the final `L1` recipient.

### Local Invariants

- `[x]` the direct withdraw path must be restricted to `EOA` callers
  Why it matters: the self-withdraw wrapper must not unexpectedly become a contract-call entrypoint for a different execution pattern.
  Where it is enforced:

  ```solidity
  onlyEOA
  ```

- `[x]` the self-withdraw path must not change caller semantics
  Why it matters: the direct withdraw model should preserve `caller = source = destination`.
  Where it is enforced:

  ```solidity
  _initiateWithdrawal(_l2Token, msg.sender, msg.sender, _amount, _minGasLimit, _extraData);
  ```

- `[x]` the wrapper must forward the token and withdraw parameters into the internal path without rewriting them
  Why it matters: the user-facing wrapper must not distort downstream withdraw intent before the core path starts.
  Where it is enforced:

  ```solidity
  _initiateWithdrawal(_l2Token, msg.sender, msg.sender, _amount, _minGasLimit, _extraData);
  ```

### Local Conclusion

`withdraw(...)` does not contain the main accounting surface. Its local role is to define correct self-withdraw entry semantics before handing control into the internal withdrawal path.

---

## 2. `L2StandardBridge._initiateWithdrawal(...)`

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

### What It Does

This is the branch-selection layer for `L2` withdrawals. It explicitly separates `ETH` and `ERC20` withdrawal models and, for the standard ERC20 path, derives the corresponding `L1` token directly from the `L2` representation token.

### Local Invariants

- `[x]` the withdraw path must explicitly separate the `ETH` and `ERC20` models
  Why it matters: `ETH` and `ERC20` withdrawal semantics must not be mixed inside one implicit branch.
  Where it is enforced:

  ```solidity
  if (_l2Token == Predeploys.LEGACY_ERC20_ETH) {
      _initiateBridgeETH(...);
  } else {
      ...
  }
  ```

- `[x]` the non-ETH withdraw path must derive the `L1` pair from the `L2 representation token` itself
  Why it matters: the `L2 -> L1` withdraw path must not be built against an arbitrary caller-supplied `L1` pair assumption.
  Where it is enforced:

  ```solidity
  address l1Token = OptimismMintableERC20(_l2Token).l1Token();
  ```

- `[x]` the `ERC20` withdraw handoff must pass `source`, `destination`, and withdraw parameters into the core path without local rewriting
  Why it matters: the branch-selection layer must not distort downstream bridge semantics.
  Where it is enforced:

  ```solidity
  _initiateBridgeERC20(_l2Token, l1Token, _from, _to, _amount, _minGasLimit, _extraData);
  ```

### Local Conclusion

`_initiateWithdrawal(...)` is the gateway between wrapper-level `L2` withdrawal semantics and the shared bridge core. For the `ERC20` branch it fixes the `L1` token pair through the `L2` representation token and then routes the flow into the common bridge path.

---

## 3. `StandardBridge._initiateBridgeERC20(...)` in Withdraw Context

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

### What It Does

In the standard withdraw model this function closes the source-side `L2` accounting branch and constructs the cross-domain handoff toward `L1` finalization.

### Local Invariants

- `[x]` the `ERC20 bridge initiation path` must not accept `ETH value`
  Why it matters: the withdraw-side `ERC20` flow must not be silently mixed with `ETH`-value semantics.
  Where it is enforced:

  ```solidity
  require(msg.value == 0, "StandardBridge: cannot send value");
  ```

- `[x]` the mintable-local-token branch must confirm the correct `local/remote` pair before `burn`
  Why it matters: the representation-side withdrawal accounting path must not burn a token under wrong pair assumptions.
  Where it is enforced:

  ```solidity
  require(
      _isCorrectTokenPair(_localToken, _remoteToken),
      "StandardBridge: wrong remote token for Optimism Mintable ERC20 local token"
  );
  ```

- `[x]` the mintable-local-token branch must complete source-side accounting through `burn`, not through custody lock
  Why it matters: the `L2` representation withdraw model must close as a burn-side accounting path.
  Where it is enforced:

  ```solidity
  IOptimismMintableERC20(_localToken).burn(_from, _amount);
  ```

- `[x]` the cross-domain message must target only the counterpart bridge
  Why it matters: the `L1` completion path must not be constructed toward an arbitrary remote target.
  Where it is enforced:

  ```solidity
  _target: address(otherBridge)
  ```

- `[x]` the finalize payload must reverse the `local/remote` token arguments for the remote-side context
  Why it matters: the `L1` completion side must see pair semantics in its own chain-relative orientation.
  Where it is enforced:

  ```solidity
  this.finalizeBridgeERC20.selector,
  _remoteToken,
  _localToken,
  _from,
  _to,
  _amount,
  _extraData
  ```

### Local Conclusion

In the withdrawal context, `_initiateBridgeERC20(...)` is the main source-side `burn` boundary. It closes `L2` accounting, emits initiation semantics, and sends the flow into the counterpart-targeted `L1` finalize path.

---

## 4. `StandardBridge.finalizeBridgeERC20(...)` in Withdraw Context

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

### What It Does

In the standard withdraw model this function closes the destination-side `L1` release path. For the ordinary-token branch it reduces internal accounting and releases the already-escrowed original asset.

### Local Invariants

- `[x]` the `ERC20 finalize path` must be restricted only to the `counterpart bridge`
  Why it matters: destination-side `L1` asset release must not be triggered by an arbitrary caller.
  Where it is enforced:

  ```solidity
  onlyOtherBridge
  ```

- `[x]` the `ERC20 finalize path` must not execute in the paused state
  Why it matters: the bridge must preserve emergency-stop capability on the completion side.
  Where it is enforced:

  ```solidity
  require(paused() == false, "StandardBridge: paused");
  ```

- `[x]` the ordinary-token finalize branch must decrement internal `deposit accounting` before `release`
  Why it matters: the original-token release path must not deliver custody-backed assets without the matching accounting decrement.
  Where it is enforced:

  ```solidity
  deposits[_localToken][_remoteToken] = deposits[_localToken][_remoteToken] - _amount;
  IERC20(_localToken).safeTransfer(_to, _amount);
  ```

- `[x]` the ordinary-token finalize branch must complete destination-side delivery through releasing existing tokens, not through `mint`
  Why it matters: withdraw completion on the `L1` original-token side must not become a synthetic credit path.
  Where it is enforced:

  ```solidity
  IERC20(_localToken).safeTransfer(_to, _amount);
  ```

- `[x]` the finalize event must reflect the actual withdrawal completion semantics that were executed
  Why it matters: post-finalization observability must match the real state transition.
  Where it is enforced:

  ```solidity
  _emitERC20BridgeFinalized(_localToken, _remoteToken, _from, _to, _amount, _extraData);
  ```

### Local Conclusion

In the withdrawal context, `finalizeBridgeERC20(...)` is the main destination-side `L1` release boundary. It reduces backing accounting and completes delivery of already-escrowed original tokens under counterpart-gated and pause-aware semantics.

---

## 5. `L1StandardBridge.finalizeERC20Withdrawal(...)`

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

### What It Does

This is a thin wrapper for public / legacy compatibility that forwards the withdrawal completion call into the shared bridge finalize core.

### Local Invariants

- `[x]` `finalizeERC20Withdrawal(...)` must pass withdrawal semantics into `finalizeBridgeERC20(...)` without rewriting the arguments
  Why it matters: the compatibility wrapper must not introduce a separate finalize model or distort completion semantics.
  Where it is enforced:

  ```solidity
  finalizeBridgeERC20(_l1Token, _l2Token, _from, _to, _amount, _extraData);
  ```

### Local Conclusion

`finalizeERC20Withdrawal(...)` contains no standalone accounting or release logic. Its local role is to provide a public / legacy entrypoint surface over the shared bridge finalize core.
