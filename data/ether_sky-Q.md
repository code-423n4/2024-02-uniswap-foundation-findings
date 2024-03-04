**[L-1] There may be unallocated `rewards` in the `UniStaker`, but there is no way to recover them.**

When users `claim fees` through `V3FactoryOwner`, it sends the `payout` token to the `UniStaker` without checking whether there are any `staked UNI` tokens.
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L187-L188
```
function claimFees(
    IUniswapV3PoolOwnerActions _pool,
    address _recipient,
    uint128 _amount0Requested,
    uint128 _amount1Requested
) external returns (uint128, uint128) {
    PAYOUT_TOKEN.safeTransferFrom(msg.sender, address(REWARD_RECEIVER), payoutAmount);
    REWARD_RECEIVER.notifyRewardAmount(payoutAmount);
}
```
So there is no guarantee that there will always be some `staked` tokens when the `rewards` are sent to the `UniStaker`.
Once the `rewards` are sent, they are distributed for a certain period. 
If there are `0 staked UNI` tokens for some days, the `rewards` for these days are not allocated to any `depositors`.
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L577-L585
```
function notifyRewardAmount(uint256 _amount) external {
    if (block.timestamp >= rewardEndTime) {
      scaledRewardRate = (_amount * SCALE_FACTOR) / REWARD_DURATION;
    } else {
      uint256 _remainingReward = scaledRewardRate * (rewardEndTime - block.timestamp);
      scaledRewardRate = (_remainingReward + _amount * SCALE_FACTOR) / REWARD_DURATION;
    }

    rewardEndTime = block.timestamp + REWARD_DURATION;
    lastCheckpointTime = block.timestamp;
}
```
However, there is currently no way to recover them.

We should allow the `owner` to recover the excess `payout` token.