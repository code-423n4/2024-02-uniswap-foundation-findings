Medium Issues

1.Divide Before Multiply:
UniStaker.sol::570=> notifyRewardAmount(uint256) performs a multiplication on the result of a division- scaledRewardRate = (_remainingReward + _amount * SCALE_FACTOR) / REWARD_DURATION

2.Dangerous Strict Equalities:
UniStaker.sol::749=> _claimReward(address) (UniStaker.sol#3873-3883) uses a dangerous strict equality- _reward == 0

Low Issues

1.Block Timestamp:
UniStaker.sol::=> lastTimeRewardDistributed() uses timestamp for comparisons- rewardEndTime <= block.timestamp
UniStaker.sol::=> notifyRewardAmount(uint256) (UniStaker.sol#3703-3732) uses timestamp for comparisons- block.timestamp >= rewardEndTime

### Time spent:
09 hours