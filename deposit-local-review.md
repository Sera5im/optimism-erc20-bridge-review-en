# Deposit Local Review

## Reviewed Local Functions

- `L1StandardBridge.depositERC20(...)`
- `L1StandardBridge.depositERC20To(...)`
- `L1StandardBridge._initiateERC20Deposit(...)`
- `StandardBridge._initiateBridgeERC20(...)`
- `StandardBridge.finalizeBridgeERC20(...)`

---

## 1. `L1StandardBridge.depositERC20(...)`

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

### What It Does

This is the user-facing self-deposit wrapper. It opens the direct `L1 -> L2` deposit path where the caller remains both the source and the remote recipient.

### Local Invariants

- `[x]` the direct deposit path must be restricted to `EOA` callers
  Why it matters: the self-deposit wrapper should not silently become a contract-call entrypoint for a different execution pattern.
  Where it is enforced:

  ```solidity
  onlyEOA
  ```

- `[x]` the self-deposit path must not change sender/recipient semantics
  Why it matters: the direct path should preserve the expected model `caller = source = destination`.
  Where it is enforced:

  ```solidity
  _initiateERC20Deposit(_l1Token, _l2Token, msg.sender, msg.sender, _amount, _minGasLimit, _extraData);
  ```

- `[x]` the wrapper must forward the declared token pair and deposit parameters without rewriting them
  Why it matters: the user-facing wrapper should not distort downstream bridge semantics before the core path begins.
  Where it is enforced:

  ```solidity
  _initiateERC20Deposit(_l1Token, _l2Token, msg.sender, msg.sender, _amount, _minGasLimit, _extraData);
  ```

### Local Conclusion

`depositERC20(...)` does not perform accounting by itself. Its local role is to define correct self-deposit entry semantics before handing control into the internal `ERC20` deposit core.

---

## 2. `L1StandardBridge.depositERC20To(...)`

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

### What It Does

This is the generalized deposit wrapper. The caller remains the source-side sender, but the remote recipient is explicitly set through `_to`.

### Local Invariants

- `[x]` the `deposit-to` path must preserve `msg.sender` as the source and `_to` as the explicit destination
  Why it matters: the generalized wrapper must not break the distinction between the source-side owner and the remote-side recipient.
  Where it is enforced:

  ```solidity
  _initiateERC20Deposit(_l1Token, _l2Token, msg.sender, _to, _amount, _minGasLimit, _extraData);
  ```

- `[x]` the wrapper must forward the token pair and deposit parameters into the core path without local rewriting
  Why it matters: the local wrapper must not silently rewrite downstream bridge intent.
  Where it is enforced:

  ```solidity
  _initiateERC20Deposit(_l1Token, _l2Token, msg.sender, _to, _amount, _minGasLimit, _extraData);
  ```

### Local Conclusion

`depositERC20To(...)` extends the deposit path into a generalized recipient model, but it does not change accounting logic itself. It only sets the correct source/destination semantics before entering the internal bridge core.

---

## 3. `L1StandardBridge._initiateERC20Deposit(...)`

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

### What It Does

This is the handoff layer between `L1StandardBridge` wrappers and the shared `StandardBridge` ERC20 core.

### Local Invariants

- `[x]` the `ERC20 deposit` path must converge into a single shared bridge core
  Why it matters: the main accounting and message logic should not be duplicated across multiple wrapper-level entrypoints.
  Where it is enforced:

  ```solidity
  _initiateBridgeERC20(_l1Token, _l2Token, _from, _to, _amount, _minGasLimit, _extraData);
  ```

### Local Conclusion

`_initiateERC20Deposit(...)` is a thin routing layer. It carries no standalone accounting surface and exists to route all `L1 ERC20 deposit` wrappers into one core bridge path.

---

## 4. `StandardBridge._initiateBridgeERC20(...)`

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

This is the main local boundary for `ERC20` deposit initiation. It closes the source-side accounting branch, emits the initiation event, and constructs the cross-domain handoff toward the remote finalize path.

### Local Invariants

- `[x]` the `ERC20 bridge initiation path` must not accept `ETH value`
  Why it matters: the `ERC20` bridge path must not be silently mixed with `ETH`-value semantics.
  Where it is enforced:

  ```solidity
  require(msg.value == 0, "StandardBridge: cannot send value");
  ```

- `[x]` the mintable-local-token branch must confirm the correct `local/remote` pair before `burn`
  Why it matters: the representation-side accounting path must not burn a token against the wrong remote pair assumption.
  Where it is enforced:

  ```solidity
  require(
      _isCorrectTokenPair(_localToken, _remoteToken),
      "StandardBridge: wrong remote token for Optimism Mintable ERC20 local token"
  );
  ```

- `[x]` the mintable-local-token branch must complete source-side accounting through `burn`, not through escrow lock
  Why it matters: the representation-token path must not be mixed with the original-token custody model.
  Where it is enforced:

  ```solidity
  IOptimismMintableERC20(_localToken).burn(_from, _amount);
  ```

- `[x]` the ordinary-token branch must complete source-side accounting through `transfer-in` and internal `deposit accounting`
  Why it matters: the original-token custody path must not create remote credit without a local escrow/accounting step.
  Where it is enforced:

  ```solidity
  IERC20(_localToken).safeTransferFrom(_from, address(this), _amount);
  deposits[_localToken][_remoteToken] = deposits[_localToken][_remoteToken] + _amount;
  ```

- `[x]` the bridge initiation event must reflect the same `token/from/to/amount` semantics that actually entered the message path
  Why it matters: event-level observability must not diverge from the real cross-domain handoff payload.
  Where it is enforced:

  ```solidity
  _emitERC20BridgeInitiated(_localToken, _remoteToken, _from, _to, _amount, _extraData);
  ```

- `[x]` the cross-domain message must target only the counterpart bridge
  Why it matters: the finalize path must not be constructed toward an arbitrary remote target.
  Where it is enforced:

  ```solidity
  _target: address(otherBridge)
  ```

- `[x]` the finalize payload must reverse the `local/remote` token arguments for the remote-side context
  Why it matters: on the remote side the `local` and `remote` semantics refer to a different chain context.
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

`_initiateBridgeERC20(...)` is the main local security boundary of the deposit path. This is where the bridge selects the accounting branch, fixes initiation semantics, and constructs the correct cross-domain handoff into the counterpart finalize path.

---

## 5. `StandardBridge.finalizeBridgeERC20(...)`

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

This function closes the destination-side completion path for an `ERC20` bridge message. It allows completion only from the counterpart bridge and selects the correct branch between `mint` and `release`.

### Local Invariants

- `[x]` the `ERC20 finalize path` must be restricted only to the `counterpart bridge`
  Why it matters: destination-side asset credit/release must not be triggered by an arbitrary caller.
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

- `[x]` the mintable-local-token finalize branch must confirm the correct token pair before `mint`
  Why it matters: representation-side credit must not be issued against a wrong remote pairing.
  Where it is enforced:

  ```solidity
  require(
      _isCorrectTokenPair(_localToken, _remoteToken),
      "StandardBridge: wrong remote token for Optimism Mintable ERC20 local token"
  );
  ```

- `[x]` the mintable-local-token finalize branch must complete destination-side credit through `mint`, not through releasing existing original tokens
  Why it matters: representation-token delivery must not be mixed with the original-asset release branch.
  Where it is enforced:

  ```solidity
  IOptimismMintableERC20(_localToken).mint(_to, _amount);
  ```

- `[x]` the ordinary-token finalize branch must decrement internal `deposit accounting` before `release`
  Why it matters: the release path must not deliver original assets without decrementing the corresponding source-backed accounting.
  Where it is enforced:

  ```solidity
  deposits[_localToken][_remoteToken] = deposits[_localToken][_remoteToken] - _amount;
  IERC20(_localToken).safeTransfer(_to, _amount);
  ```

- `[x]` the ordinary-token finalize branch must complete destination-side delivery through releasing existing tokens, not through `mint`
  Why it matters: the original-token custody model must not create a synthetic credit path.
  Where it is enforced:

  ```solidity
  IERC20(_localToken).safeTransfer(_to, _amount);
  ```

- `[x]` the finalize event must reflect the actual completion semantics that were executed
  Why it matters: post-finalization observability must match the real state transition.
  Where it is enforced:

  ```solidity
  _emitERC20BridgeFinalized(_localToken, _remoteToken, _from, _to, _amount, _extraData);
  ```

### Local Conclusion

`finalizeBridgeERC20(...)` is the main local completion boundary on the destination side. It validates the counterpart context, enforces the pause gate, and completes the path through the correct branch between `representation mint` and `original-token release`.
