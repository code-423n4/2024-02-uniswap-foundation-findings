[L-0] Change the `V3FacotryOwner.claimFees()` to accept the `amount - 1` equal to fees from the uniSwap V3 pools

Uniswap v3 pools doesn't let anyone claim whole fees from the pool so that the slot is not empty. So if someone who claims the fees passing the whole amount as a paramenter will only get `amount - 1`. But this is not checked in the `V3FacotryOwner.claimFees()` functions. Also there is no information given about it anywhere in the natspac or somewhere else. As we know there will be race conditions to claim the fees by users. If someone was successfully able to make call before everybody else but passed the whole `amount` to be claimed from the pool, then the function will revert due to this check:

```solidity
File: V3FacotryOwner.sol

  function claimFees(
    IUniswapV3PoolOwnerActions _pool,
    address _recipient,
    uint128 _amount0Requested,
    uint128 _amount1Requested
  ) external returns (uint128, uint128) {
    PAYOUT_TOKEN.safeTransferFrom(msg.sender, address(REWARD_RECEIVER), payoutAmount);
    REWARD_RECEIVER.notifyRewardAmount(payoutAmount);
    (uint128 _amount0, uint128 _amount1) =
      _pool.collectProtocol(_recipient, _amount0Requested, _amount1Requested);


    // Protect the caller from receiving less than requested. See `collectProtocol` for context.
@>    if (_amount0 < _amount0Requested || _amount1 < _amount1Requested) {
      revert V3FactoryOwner__InsufficientFeesCollected();
    }
    emit FeesClaimed(address(_pool), msg.sender, _recipient, _amount0, _amount1);
    return (_amount0, _amount1);
  }

```

GitHub: [[181-198](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L181C1-L198C4)]

So it is recommended to add the following checks or mention this in the comments above the functions itself.

```diff
  function claimFees(
    IUniswapV3PoolOwnerActions _pool,
    address _recipient,
    uint128 _amount0Requested,
    uint128 _amount1Requested
  ) external returns (uint128, uint128) {
    PAYOUT_TOKEN.safeTransferFrom(msg.sender, address(REWARD_RECEIVER), payoutAmount);
    REWARD_RECEIVER.notifyRewardAmount(payoutAmount);
    (uint128 _amount0, uint128 _amount1) =
      _pool.collectProtocol(_recipient, _amount0Requested, _amount1Requested);


    // Protect the caller from receiving less than requested. See `collectProtocol` for context.
-    if (_amount0 < _amount0Requested || _amount1 < _amount1Requested) {
+    if (_amount0 < (_amount0Requested - 1) || _amount1 < (_amount1Requested - 1)) {
      revert V3FactoryOwner__InsufficientFeesCollected();
    }
    emit FeesClaimed(address(_pool), msg.sender, _recipient, _amount0, _amount1);
    return (_amount0, _amount1);
  }
```

[N-0] Mistakes in Natspac

There is spelling mistake in the following places

```solidity
File: UniStaker.sol

18.    /// deposit a designated, delegable ERC20 governance token and leave it over a period of time.
```

GitHub: [[18](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L18)]

[N-1] No need for else statement

There is no need for else statment if contract doesn't have any further functionality after it. Consider removing it.

```solidity
File: UniStaker.sol

220  function lastTimeRewardDistributed() public view returns (uint256) {
221    if (rewardEndTime <= block.timestamp) return rewardEndTime;
222    else return block.timestamp;
223  }
```

GitHub: [[220=223](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L220C1-L223C4)]
