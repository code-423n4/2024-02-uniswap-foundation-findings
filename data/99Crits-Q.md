### [01] Rewards are lost if totalStaked == 0

The `UniStaker` contract can still be notified of rewards even if there are no tokens staked. The function `rewardPerTokenAccumulated` handles this specific case by not increasing the accumulator checkpoint:

```js
  function rewardPerTokenAccumulated() public view returns (uint256) {
    
    if (totalStaked == 0) return rewardPerTokenAccumulatedCheckpoint;
```

However, the `lastCheckpointTime` would still continue to progress until `rewardEndTime`. So the rewards are effectively still getting streamed without anyone receiving them.

The likelihood is of 0 tokens being staked at any given time seems very low however, hence I believe the severity to be also low.

**Mitigation:**

Either:
1) Disallow calling `notifyRewardAmount` if there are no tokens staked
2) Extend the rewardEndTime by the time that has passed since the last checkpoint if `totalStaked == 0`

### [02] Governance process needs to be held for every pool

The `setFeeProtocol` function of the V3FactoryOwner needs to go through the governance process for every single pool. This limits the scalability of the feesharing process.

It might be preferable to have default values that were agreed on by the governance. And that can be set by anyone (or maybe only by whitelisted) participants.

### [03] V3FactoryOwner cant pass on factory ownership

The setOwner has no function that enables it to call the `setOwner` of `UniswapV3Factory`. That means once the ownership has been transferred to the V3FactoryOwner, it will remain the owner indefinitely. While there may be no drawbacks to that at the current time, the requirements on the factory owner may change in the future and a migration would not be possible by then.

Since all admin functions of the V3FactoryOwner are access restricted by the DAO governance there would also be no centralization risk or risk of abuse concerning adding such a feature.