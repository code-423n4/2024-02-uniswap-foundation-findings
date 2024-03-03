**[[1]]** The comment to the `UniStaker.stakeOnBehalf` function claims:
```
The caller must pre-approve the staking contract to spend at least the
would-be staked amount of the token.
```
This comment is misleading since the caller might not be a depositor.
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L306

**[[2]]** The comment to the `UniStaker.stakeMoreOnBehalf` function claims:
```
A staker should call this method when they have an existing deposit, and wish to stake more while retaining the same delegatee and beneficiary.
```
This comment is misleading because this transaction might not be sent by a staker.
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L376

**[[3]]** The `UniStaker` contract contains the `permitAndStake` function to combine approving and staking in one call. It also contains the `stakeOnBehalf` function to provide a way to execute transactions on behalf of a user. But this function still requires an additional `approve` transaction. It seems that a new function `permitAndStakeOnBehalf` might be useful to allow completely gasless interaction with the `UniStaker` contract. The same reasoning can be applied to a new `permitAndStakeMoreOnBehalf` function. 