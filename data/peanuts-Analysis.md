### Summary and Explanation

- UNI token holders can now stake their tokens and earn rewards.
- This rewards is actually from fees accrued in the Uniswap pool.
- Usually, uniswap pool has 2 tokens, so it is very hard for the fees to be distributed properly to the UNI holders
- To combat this issue, another party, the claimer, collects these two tokens and give an equivalent amount (or slightly less, to incentivize this action) of a single token to the Unistaker pool as the rewards to be given to the UNI stakers.
- This amount to be given is set by the admin. (eg 10000 USDC) Only the admin can change this amount.
- The claimer will wait until 10000+ USDC worth of fees is accrued and exchange their one token with the two fee tokens
- Their one token will become rewards for the UNI token holders

- For example, the WBTC-UNI pool has accrued some fees, eg 0.01 BTC and 100 UNI. It's very hard to give these two tokens to the users.
- Assume 0.01 BTC and 100 UNI is worth 5000 USDC each. Assume WETH is worth 1000 each. 
- A claimer comes in, sees that the pool has 0.01 BTC and 100 UNI worth 10000 USDC. He collects these two tokens in exchange for 10 WETH, worth 10000 USDC.
- This 10 WETH will be the reward tokens, and UNI token holders who stake their tokens will earn this 10 weth proportionately
- Note that there will always be a claimer because if someone doesn't want to swap their WETH for the fees, then the fees will continue to accrue, and once it reaches a point where it is profitable, the claimer will want to exchange the fees for the reward amount

- The reward distribution typically lasts for 30 days. Rewards are counted every second. A user who deposits late (eg 15 days) will not get rewards from day 1 to 15.
- UNI token holders can exit their position anytime.
- UNI token holders also have the capability to increase their stake, or change their reward (beneficiary) address.
- UNI tokens can also delegate their votes to a delegatee of their choice when staking.

- This whole protocol is an ongoing process. Claimers will wait to claim their fees in exchange for the payout token, and the UNI stakers waits until the reward tokens is transferred into the Unistaker contract to earn rewards.
- The protocol will only not work if there are no more fees to claim from the pools that enable fees (which is highly unlikely)

### Audit Approach

- Read through the README: https://code4rena.com/audits/2024-02-unistaker-infrastructure#top
- Read through the docs: https://docs.unistaker.io/
- Started UniStaker.sol: https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L605-L615
- Did the reward calculations on RemiX
- Checked the external contracts, UniswapV3Factory and UniswapV3Pool: https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol
- Checked the UNI token contract: https://etherscan.io/token/0x1f9840a85d5af5bf1d1762f925bdaddc4201f984#code. 
- Briefly checked the SignatureChecker contract on OpenZeppelin.
- Started writing reports for findings
- Finished with QA and Analysis Reports

### Codebase quality analysis

##### [DelegationSurrogate.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/DelegationSurrogate.sol)

| Contract  | Function    | Access | Comments                                                                                                |
| --------- | ----------- | ------ | ------------------------------------------------------------------------------------------------------- |
| UniStaker | constructor | -      | Delegates all token in the contract to the delegatee, and approves Unistaker to use all the Uni tokens. |

**Checks**

- Checked approve is called to the right address (The Unistaker contract).
- Checked UNI token has a delegate function and is called correctly.

**Potential Issues**

- constructor has the ` _token.delegate(_delegatee);` call, so the delegation can only be called once. Users can stake more into the surrogate contract, but delegate will not be called. (Medium)

##### [Unistaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol)

| Contract  | Function                     | Access    | Comments                                                                                    |
| --------- | ---------------------------- | --------- | ------------------------------------------------------------------------------------------- |
| UniStaker | setAdmin                     | onlyAdmin | No two step transfer of ownership                                                           |
| UniStaker | setRewardNotifier            | onlyAdmin | Checks whether contract is ready to receive reward via notifyRewardAmount                   |
| UniStaker | lastTimeRewardDistributed    | view      | Called in global checkpoint reward. Sets lastCheckpointTime                                 |
| UniStaker | rewardPerTokenAccumulated    | view      | Called in global checkpoint reward. Sets rewardPerTokenAccumulatedCheckpoint                |
| UniStaker | unclaimedReward              | view      | Calculates the unclaimed reward of the beneficiary                                          |
| UniStaker | stake                        | public    | Entry point of users. Calls \_stake                                                         |
| UniStaker | stake                        | public    | Entry point of users. Calls \_stake. Sets the beneficiary                                   |
| UniStaker | permitAndStake               | public    | Entry point of users. Calls \_stake. Sets the beneficiary. Note permit frontrun issue       |
| UniStaker | stakeOnBehalf                | public    | Helps another person call stake. Must be a trusted signature                                |
| UniStaker | stakeMore                    | public    | Must already have a stake. Only can stake more for oneself                                  |
| UniStaker | permitAndStakeMore           | public    | Must already have a stake. Only can stake more for oneself                                  |
| UniStaker | stakeMoreOnBehalf            | public    | Must already have a stake. Must be a trusted user                                           |
| UniStaker | alterDelegatee               | public    | Must already have a stake. Surrogate contract transfers UNI tokens to other Surrogate       |
| UniStaker | alterDelegateeOnBehalf       | public    | Same as above. Caller must be trusted                                                       |
| UniStaker | alterBeneficiary             | public    | Changes the reward recipient. Both previous and current recipient will update their rewards |
| UniStaker | alterBeneficiaryOnBehalf     | public    | Same as above. Caller must be trusted                                                       |
| UniStaker | withdraw                     | public    | Withdraws UNI tokens from surrogate contract to the deposit owner                           |
| UniStaker | withdrawOnBehalf             | public    | Same as above. Caller must be trusted                                                       |
| UniStaker | claimReward                  | public    | Only beneficiary can call                                                                   |
| UniStaker | claimRewardOnBehalf          | public    | Claims reward for the beneficiary.                                                          |
| UniStaker | notifyRewardAmount           | Claimer   | Called by Claimer in V3FactoryOwner. Reward tokens is deposited into Unistaker              |
| UniStaker | \_fetchOrDeploySurrogate     | internal  | Deploys a new surrogate contract if delegatee doesn't exist yet.                            |
| UniStaker | \_stakeTokenSafeTransferFrom | internal  | Safe transfers token                                                                        |
| UniStaker | \_fetchOrDeploySurrogate     | internal  | Deploys a new surrogate contract if delegatee doesn't exist yet.                            |
| UniStaker | \_useDepositId               | internal  | Called in \_stake. Increments depositId value by 1                                          |
| UniStaker | \_stake                      | internal  | Beneficiary gets earningPower, creates a Deposit struct, call checkpoint global and reward  |
| UniStaker | \_stakeMore                  | internal  | Calls checkpoint global and reward                                                          |
| UniStaker | \_alterDelegatee             | internal  | Deploys new surrogate if doesn't exist. Updates deposit struct with new delegatee           |
| UniStaker | \_alterBeneficiary           | internal  | Updates previous and current beneficiary, update deposit struct with new beneficiary        |
| UniStaker | \_withdraw                   | internal  | Opposite of \_stake, transfers withdraw amount to the deposit.owner                         |
| UniStaker | \_claimReward                | internal  | Beneficiary will claim the reward. Reward will be set to zero                               |
| UniStaker | \_checkpointGlobalReward     | internal  | Updates rewardPerTokenAccumulatedCheckpoint, updates lastCheckpointTime                     |
| UniStaker | \_checkpointReward           | internal  | Updates unclaimedRewardCheckpoint, updates beneficiaryRewardPerTokenCheckpoint              |

**Scenarios Checked**

1. Can first depositor steal all the funds?

- No, first depositor cannot steal all the funds
- The reward token per uni is split evenly. If user A is first depositor, and subsequent users deposit before rewards are in, nothing will be broken.

2. Can totalStaked be manipulated?

- It can be changed to a huge number, but the change will not affect the collection of rewards
- First depositor can deposit 1 wei, and when rewards are distributed, can deposit 100e18 UNI tokens. This will make rewardPerTokenAccumulatedCheckpoint extremely large.
- This large amount does not matter because when 100e18 UNI tokens is deposited, `_checkpointReward()` is called and `beneficiaryRewardPerTokenCheckpoint[_beneficiary] = rewardPerTokenAccumulatedCheckpoint`. That means that from now on, although the number is extremely large, it doesn't matter because the calculation starts from that second.
- The 1 wei holder also will not get extra rewards because of the `unclaimedReward()` calculation

3. Is rewards distributed proportionately?

- Yes, rewards are checked and distributed proportionately. Assume 300 Reward tokens for 30 days. User A is the only user with 100 UNI staked. 10 days later, user B stakes 100 UNI. 10 days after this, User C stakes 800 UNI.
- First 10 days, user A gets 100 Reward tokens
- Second 10 days, user A gets 50 reward tokens, user B gets 50 reward tokens
- Last 10 days, user A gets 10, user B gets 10, user C gets 80.

- This is because everytime someone stakes, they will start with rewardPerTokenAccumulatedCheckpoint as their `beneficiaryRewardPerTokenCheckpoint[_beneficiary]`. If they decide to claim the second they stake, `rewardPerTokenAccumulated()` - `beneficiaryRewardPerTokenCheckpoint[_beneficiary]` = 0. They will start earning their share the second they start staking

4. What if a user doesn't claim their reward?

- It will just be stuck in unclaimedReward. Everytime a depositor stake more, the beneficiary will get more rewards, and `unclaimedReward()` simplay adds the previous unclaimed rewards

5. Can a reward caller grief another reward caller?

- No, because of the ` if (_amount0 < _amount0Requested || _amount1 < _amount1Requested) {` check. If the claimer do not get the fees they asked for, they will not transfer their reward tokens to the UniStaker contract.

6. Can a staker claim more rewards than expected?

- No, because all reward emission will end after the 30 day mark, and they will not earn anything else until the next rewards are funnelled in.

7. Can transferring beneficiary result in double rewards?

- No. When transferring beneficiary, checkpointglobalcheckpoint and checkpointreward is called to update their collected rewards. Transferring beneficiary, even if the beneficiary is the same address, will not affect anything.
- `earningPower[_beneficiary] * (rewardPerTokenAccumulated() - beneficiaryRewardPerTokenCheckpoint[_beneficiary])` will return zero as `rewardPerTokenAccumulated() - beneficiaryRewardPerTokenCheckpoint[_beneficiary]` is zero.

8. Does transferring delegatee break anything?

- No, old surrogate contract will simply transfer UNI to new surrogate contract.
- Transferring delegatee, even if delegatee is ownself, does not affect the transfer of uni tokens

9. Does any reward token get stuck in contract?

- Not really, all reward tokens are claimable once the rewardEndTime is reached. It is up to the beneficiary to collect their rewards

10. Does flashloaning work?

- Nope, because flashloan happens within the second, and no rewards will be accrued within a second. Only 1 second later, which defeats the purpose of a flash loan.

**Potential Issues**

- No potential issues found thus far, other than those already known eg Permit Frontrunning.

##### [V3FactoryOwner.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol)

| Contract       | Function        | Access    | Comments                                                                 |
| -------------- | --------------- | --------- | ------------------------------------------------------------------------ |
| V3FactoryOwner | setAdmin        | onlyAdmin | No two step transfer of ownership                                        |
| V3FactoryOwner | setPayoutAmount | onlyAdmin | Sensitive function, claimers will exchange payout amount for fees        |
| V3FactoryOwner | enableFeeAmount | onlyAdmin | Enables collection of fees, called to UniswapV3Factory.sol               |
| V3FactoryOwner | setFeeProtocol  | onlyAdmin | Sets fee amount, called to UniswapV3Pool.sol                             |
| V3FactoryOwner | claimFees       | Public    | Called by anyone who wants to collect fees in exchange for payout amount |

**Notes**
- Note that the Admin of this V3FactoryOwner is different from the Owner of the FactoryOwner contract
- Admin is the Timelock contract
- FactoryOwner is V3FactoryOwner itself
- Original UniswapV3Factory: https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Factory.sol
- UniswapV3Pool: https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol

**Checks**
- Checked calls to UniswapV3Factory and UniswapV3Pool is correct
- Checked all sensitive functions have access control enabled
- Checked call to Unistaker contract is correct
- Checked transferring of payout amount to unistaker is correct
- Checked collection of fees from pool is correct

**Potential Issues**
- Overhauling UniswapV3Factory as the new factory owner will affect `createPool()` in UniswapV3Factory (Medium)
- Change of owner is irreversible (Low/Medium)
- Protocols with fees enabled that doesn't earn enough fees will have their fees stuck in contract since no one wants to exchange payout amount (Medium)

### Mechanism Review

##### For Users
**Staking and withdrawing**
- Mechanism is simple to understand, staking UNI tokens is easy and withdrawing UNI tokens is also easy. There is no delay, and users can earn rewards even if tokens are just staked for a second

**Altering Delegatees and Beneficiaries**
- Users can alter their delegatees and beneficiaries easily without any fees, and the change is seamless.

**Collecting Rewards**
- Beneficiaries can collect rewards through the click of one function. Note that there might be some issues if the beneficiary is a smart contract.
- Reward tokens go directly to the beneficiary which is convenient.

#### For Claimers
- Claimers are protected against frontrunning attack by specifying minimum amounts which is good.
- For pools with fee-on-transfer tokens, claimers must be careful of their amount requested (it should be the original amount, and not the fee-on-transferred amount)
- Note that the amount of fees is calculated as totalAmount - 1 wei, not totalAmount. 

### Centralization Risks
- Not much centralization risks, except two important parts.
- In Unistaker.sol, the admin can set the reward notifier via `setRewardNotifier()`. If it is not set, the whole protocol will not work
- In V3FactoryOwner.sol, the admin can change the payout amount at any time through `setPayoutAmount()`. Able to grief the claimers and the protocol itself.

### Systemic Risks
- Not much systemic risks, users do not really stand to lose anything. The protocol also does not face any infrastructure problem. At the most, no fees are collected and UNI stakers will not earn anything since no claimers will exchange the fee for reward tokens, but then users can simply withdraw their UNI tokens.

### Time spent:
025 hours