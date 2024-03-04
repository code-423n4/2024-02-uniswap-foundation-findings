# Qa report
# Low Risk Findings 

## L-01 `claimFees()` may revert for the first few transactions when pool fees are being claimed.
### Summary 
Any entity that is willing to pay a fixed amount of the requested token to the staking contract in exchange for the fee tokens accrued by a given pool will receive it, competition between individuals to claim the fees is expected and once the operational cost has been covered by the amount of fees in the pool the function is called. Because of this competition, the function will most likely be called once the pool's fees reach a certain value and in an attempt to extract maximum value and be in profit the exact amount of protocol fess available will be inserted as a parameter in the transaction but this can give rise to a situation where the amount requested by the caller is exactly equal to the amount of protocol fees in the pool leading to a revert.

### Vulnerability Details 
When the [claimFees()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L181-L204) function is called, users input their amount requested for token 0 and token 1 then after transferring to the required amount to the `reward reciever` the function [collectProtocol()](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L848-L868) is called.
```solidity
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


  /// @notice Ensures the msg.sender is the contract admin and reverts otherwise.
  /// @dev Place inside external methods to make them admin-only.
  function _revertIfNotAdmin() internal view {
    if (msg.sender != admin) revert V3FactoryOwner__Unauthorized();
  }
```
This issue here is that in `collectProtocol()` if the amount requested the equal to the protocol fees then the amount is decremented using `(--)` for gas saving reasons, but in `claimFees()` there is a slippage check which reverts if the amount returned is less than the requested amount.

```solidity
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
This means the first few transactions to claim the fees will keep reverting till the protocol fees for the token increases, allowing callers to call and still be in profit.

### Recommended Mitigation Step.
The slippage check in the `claimFees()` should be adjusted to account for the small change in user requested amount, if the amount is exactly equal to the protocol fees for that pool.
```solidity
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
---  if (_amount0 < _amount0Requested || _amount1 < _amount1Requested) {
+++  if (_amount0 < _amount0Requested-- || _amount1 < _amount1Requested--) {
      revert V3FactoryOwner__InsufficientFeesCollected();
    }
    emit FeesClaimed(address(_pool), msg.sender, _recipient, _amount0, _amount1);
    return (_amount0, _amount1);
  }


  /// @notice Ensures the msg.sender is the contract admin and reverts otherwise.
  /// @dev Place inside external methods to make them admin-only.
  function _revertIfNotAdmin() internal view {
    if (msg.sender != admin) revert V3FactoryOwner__Unauthorized();
  }
```




## L-02 `V3FactoryOwner.sol` does not sufficiently control the interval of reward notifications.
### Summary 
According to the [natspec](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L562-L565),
```solidity
/// 1. A misbehaving contract could grief stakers by frequently notifying this contract of tiny
/// rewards, thereby continuously stretching out the time duration over which real rewards are
/// distributed. It is required that reward notifiers supply reasonable rewards at reasonable
/// intervals.

```
Reward notifiers should supply a reasonable amount at reasonable intervals, although `V3FactoryOwner.sol` controls the amount being notified really well, it does not sufficiently control the interval of its calls to [notifyRewardAmount()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L570-L599).

### Vulnerability Details 
In `V3FactoryOwner.sol` the payout amount is set and regulated by the admin ensuring that only reasonable amounts are supplied to the `reward receiver` during [claimFees()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L181-L204) but the expected interval of these call breaks down when you consider that number of pools in `uniswap v3` is growing larger day by day. 
Even if the interval is sufficient now for proper distribution of rewards, as the number of pools grows and maybe even more fee tiers are added to the protocol, in the future the number of pools can grow to an extent that the `payoutAmount` becomes insufficient to regulate the interval of calls to `claimFees()` leading to continuous stretching of the time duration in which real rewards are distributed, thereby griefing stakers.
```solidity
function notifyRewardAmount(uint256 _amount) external {
if (!isRewardNotifier[msg.sender]) revert UniStaker__Unauthorized("not notifier", msg.sender);

// We checkpoint the accumulator without updating the timestamp at which it was updated, because
// that second operation will be done after updating the reward rate.
rewardPerTokenAccumulatedCheckpoint = rewardPerTokenAccumulated();

if (block.timestamp >= rewardEndTime) {
scaledRewardRate = (_amount * SCALE_FACTOR) / REWARD_DURATION;
} else {
uint256 _remainingReward = scaledRewardRate * (rewardEndTime - block.timestamp);
scaledRewardRate = (_remainingReward + _amount * SCALE_FACTOR) / REWARD_DURATION;
}

rewardEndTime = block.timestamp + REWARD_DURATION;
lastCheckpointTime = block.timestamp;

if ((scaledRewardRate / SCALE_FACTOR) == 0) revert UniStaker__InvalidRewardRate();

// This check cannot _guarantee_ sufficient rewards have been transferred to the contract,
// because it cannot isolate the unclaimed rewards owed to stakers left in the balance. While
// this check is useful for preventing degenerate cases, it is not sufficient. Therefore, it is
// critical that only safe reward notifier contracts are approved to call this method by the
// admin.
if (
(scaledRewardRate * REWARD_DURATION) > (REWARD_TOKEN.balanceOf(address(this)) * SCALE_FACTOR)
) revert UniStaker__InsufficientRewardBalance();

emit RewardNotified(_amount, msg.sender);
}

```

### Recommended Mitigation Step.
The `claimFees()` function should have a mechanism which still lets users purchase the fees when necessary but controls the interval at which it calls `notifyRewardAmount()` to ensure that stakers are not griefed of proper rewards.
