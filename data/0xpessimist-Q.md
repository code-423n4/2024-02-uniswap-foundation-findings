## [L-01] Do not allow depositing with `_amount = 0`

* https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L638-L664

Currently, it is possible to make a deposit by calling one of the stake functions with 0 tokens. Although UniStaker handles this gracefully without harming the accounting, since a Surrogate contract is deployed with each deposit operation and this consumes a significant amount of gas, staking with an amount of 0 (whether accidentally or intentionally) is just a waste of gas.

As stated in the comment lines I referenced below, UniStaker is inspired by Synthetix `StakingRewards.sol`. StakingRewards actually performs this check and [does not allow](https://github.com/Synthetixio/synthetix/blob/develop/contracts/StakingRewards.sol#L81-L95) staking or withdrawing with an amount of 0.

`UniStaker.sol:L23-24`
```solidity
23     /// The staking mechanism of this contract is directly inspired by the Synthetix StakingRewards.sol
24     /// implementation. The core mechanic involves the streaming of rewards over a designated period
```

### Recommended Mitigation Steps

Simply add a requirement of `_amount > 0` in the relevant places.