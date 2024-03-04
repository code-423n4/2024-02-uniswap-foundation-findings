### [I-01] A user calling `claimFees()` must always request for less fees than in protocolFees in the UniswapV3Pool. This is not clear in the function spec and is a user inconvenience that can result in a reverting transaction.

The protocol relies on users/MEVs calling the `claimFees()` function in `V3FactoryOwner.sol` to distribute fees to the stakers in `UniStaker.sol`. A user calls it to exchange a number of fees collected from a pool for an amount of reward token. As such, the function requires that the exact amount of requested tokens is sent to the caller, since we want the caller to not receive significantly less than what he is paying for. However, a user cannot claim all fees from the pool, because if he/she does, the call to `claimFees()` will revert. This is because the `collectProtocol()` function in `UniswapV3Pool` returns always at least 1 less than the total amount of fees accumulated in the pool(this is done as a gas saving measure). This can be easily avoided if a slippage of only `1` is allowed.

#### Proof of Concept:
Let's take a look at the `claimFees()` function:

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L181
```js
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
    if (_amount0 < _amount0Requested || _amount1 < _amount1Requested) {
      revert V3FactoryOwner__InsufficientFeesCollected();
    }
    emit FeesClaimed(address(_pool), msg.sender, _recipient, _amount0, _amount1);
    return (_amount0, _amount1);
  }

```
As we can see there is the check:

```js
   if (_amount0 < _amount0Requested || _amount1 < _amount1Requested) {
      revert V3FactoryOwner__InsufficientFeesCollected();
    }
```
As a result, if the amount returned by the pool is less than the requested amount, the call will revert.

So let's say a user requests the total amount of fees available, so `amount0` and `amount1` for `token0` and `token1` in the UniswapV3Pool at that moment. The call will unexpectedly revert, because the `collectProtocol()` function will return exactly `amount0 - 1` and `amount1 - 1`. This is the actual implemenation of UniswapV3Pool.sol

https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol
```js
    function collectProtocol(
        address recipient,
        uint128 amount0Requested,
        uint128 amount1Requested
    ) external override lock onlyFactoryOwner returns (uint128 amount0, uint128 amount1) {
        amount0 = amount0Requested > protocolFees.token0 ? protocolFees.token0 : amount0Requested;
        amount1 = amount1Requested > protocolFees.token1 ? protocolFees.token1 : amount1Requested;

        if (amount0 > 0) {
            if (amount0 == protocolFees.token0) amount0--; // ensure that the slot is not cleared, for gas savings
            protocolFees.token0 -= amount0;
            TransferHelper.safeTransfer(token0, recipient, amount0);
        }
        if (amount1 > 0) {
            if (amount1 == protocolFees.token1) amount1--; // ensure that the slot is not cleared, for gas savings
            protocolFees.token1 -= amount1;
            TransferHelper.safeTransfer(token1, recipient, amount1);
        }

        emit CollectProtocol(msg.sender, recipient, amount0, amount1);
    }
```

As we can see, the amounts `amount0` and `amount1`, are reduced by 1 each if the total amount of fees is requested.
Thus, if a user calls `claimFees()` with the total available fees for collection in the pool, their transaction will revert.

#### Recommended Mitigation Steps:
Change:

```js
    if (_amount0 < _amount0Requested || _amount1 < _amount1Requested) {
```

to:

```js
    if (_amount0 + 1 < _amount0Requested || _amount1 + 1 < _amount1Requested) {
```

This will allow for negligible slippage and will save users from this inconvenience.