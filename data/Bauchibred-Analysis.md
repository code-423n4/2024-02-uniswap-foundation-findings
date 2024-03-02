 Analysis Report for Unistaker

## Table of Contents

- [Approach](#approach)
- [Brief Overview](#brief-overview)
- [Scope and Architecture Overview](#scope-and-architecture-overview)
- [Centralization Risks](#centralization-risks)
- [Systemic Risks](#systemic-risks)
- [Recommendations](#recommendations)
- [Security Researcher Logistics](#security-researcher-logistics)
- [Conclusion](#conclusion)
- [Resources](#resources)

## Approach

Started with a comprehensive analysis of the provided docs to understand protocol functionality and key points, ambiguities were discussed with the sponsors(deva), and possible risk areas were outlined.

Followed by a manual review of each contract in scope, testing function behavior, protocol logic against expectations, and working out potential attack vectors. Vulnerabilities related to dependencies and inheritances, were also assessed. Comparisons with similar protocols was also performed to identify recurring issues and evaluate fix effectiveness.

Finally, identified issues from the security review were compiled into a comprehensive audit report.

## Brief Overview

The UniStaker protocol facilitates the collection and distribution of Uniswap V3 protocol fees through UNI staking, enabling Uniswap Governance to set fees without directly controlling the assets. Instead, fee revenue is distributed to UNI stakers who delegate their voting power. Staking rewards are paid in a predetermined token, with pool fees auctioned for this token. Stakers can designate a beneficiary for rewards, manage deposits individually, and adjust governance delegation. Implementation requires Uniswap Governance to transfer V3 factory ownership to UniStaker contracts, permanently delegating fee distribution to stakers and allowing future expansion of revenue sources.

## Scope and Architecture Overview

There are three core contracts in scope:

### V3FactoryOwner.sol

This contract is established as the owner of the Uniswap V3 Factory. Governance may choose to transfer ownership of the factory to an instance of this contract. While the factory owner becomes the 'V3FactoryOwner', the 'V3FactoryOwner' administrator will be in charge of governance. In this approach, governance maintains the ability to configure pool protocol costs using permissioned techniques.
The 'V3FactoryOwner' class offers a public method that allows anybody to claim the protocol fees that have collected for a certain pool. To claim the fees, the caller must pay a predetermined amount for a token defined when the 'V3FactoryOwner' is deployed (the 'PAYOUT_TOKEN'). This creates a constant "race" in which external parties strive to claim the expenses incurred by each pool.

```css
+--------------------------------------------+
| V3FactoryOwner  87 nSLOC  |
+--------------------------------------------+
| - FACTORY: IUniswapV3FactoryOwnerActions   |
| - PAYOUT_TOKEN: IERC20                     |
| - payoutAmount: uint256                    |
| - REWARD_RECEIVER: INotifiableRewardReceiver|
| - admin: address                           |
+--------------------------------------------+
| + constructor(_admin, _factory, _payoutToken,|
| _payoutAmount, _rewardReceiver)            |
| + setAdmin(_newAdmin)                      |
| + setPayoutAmount(_newPayoutAmount)        |
| + enableFeeAmount(_fee, _tickSpacing)      |
| + setFeeProtocol(_pool, _feeProtocol0,     |
| _feeProtocol1)                             |
| + claimFees(_pool, _recipient,             |
| _amount0Requested, _amount1Requested)      |
+--------------------------------------------+
| # _revertIfNotAdmin()                      |
+--------------------------------------------+
| > Events:                                  |
| - FeesClaimed                              |
| - AdminSet                                 |
| - PayoutAmountSet                          |
+--------------------------------------------+
| > Errors:                                  |
| - V3FactoryOwner__Unauthorized             |
| - V3FactoryOwner__InvalidAddress           |
| - V3FactoryOwner__InvalidPayoutAmount      |
| - V3FactoryOwner__InsufficientFeesCollected|
+--------------------------------------------+
```

### UniStaker.sol

The ['UniStaker'](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol) contract mechanisms are primarily influenced by the Synthetix ['StakingRewards.sol'](https://github.com/Synthetixio/synthetix/blob/develop/contracts/StakingRewards.sol) implementation. The contract controls the distribution of rewards to stakeholders by dripping them out over a certain length of time. This time resets if additional rewards are awarded (for example, through the public fee claiming procedure on 'V3FactoryOwner' described above). This Synthetix-style staking method has been widely discussed in the DeFi ecosystem and should be explored by anybody interested in understanding UniStaker's mechanics.

The UniStaker contract introduces several enhancements over StakingRewards.sol:

1. **Governance Rights Retention:** UniStaker allows stakers to maintain their UNI governance rights by utilizing DelegationSurrogate contracts for delegating voting power.

2. **Designated Reward Beneficiaries:** Stakers can specify any address as the beneficiary for their staking rewards, enabling flexibility in reward distribution.

3. **Per-Deposit Stake Tracking:** It manages stakes individually, allowing stakers to modify or move UNI within specific deposits, with each address able to hold multiple independent deposit positions.

4. **Efficiency and Clarity Improvements:** UniStaker implements optimizations for better precision and reduced gas costs, along with code refactoring for enhanced readability.

5. **Flexible Reward Sources:** Initially fueled by Uniswap V3 protocol fees, UniStaker is structured to accommodate additional reward sources, with Uniswap Governance holding the authority to expand these sources.

```css
+------------------------------------------+
| UniStaker 423 nSLOC        |
+------------------------------------------+
| - REWARD_TOKEN: IERC20                   |
| - STAKE_TOKEN: IERC20Delegates           |
| - REWARD_DURATION: uint256 (constant)    |
| - SCALE_FACTOR: uint256 (constant)      |
| - nextDepositId: DepositIdentifier       |
| - admin: address                         |
| - totalStaked: uint256                   |
| - depositorTotalStaked: mapping          |
| - earningPower: mapping                  |
| - deposits: mapping                      |
| - surrogates: mapping                    |
| - rewardEndTime: uint256                 |
| - lastCheckpointTime: uint256            |
| - scaledRewardRate: uint256              |
| - rewardPerTokenAccumulatedCheckpoint: uint256 |
| - beneficiaryRewardPerTokenCheckpoint: mapping |
| - unclaimedRewardCheckpoint: mapping     |
| - isRewardNotifier: mapping              |
+------------------------------------------+
| + constructor(_rewardToken, _stakeToken, _admin) |
| + setAdmin(_newAdmin)                    |
| + setRewardNotifier(_rewardNotifier, _isEnabled) |
| + lastTimeRewardDistributed()           |
| + rewardPerTokenAccumulated()           |
| + unclaimedReward(_beneficiary)         |
| + stake(_amount, _delegatee, _beneficiary) |
| + permitAndStake(...)                   |
| + stakeOnBehalf(...)                    |
| + stakeMore(_depositId, _amount)        |
| + permitAndStakeMore(...)               |
| + stakeMoreOnBehalf(...)                |
| + alterDelegatee(_depositId, _newDelegatee) |
| + alterDelegateeOnBehalf(...)           |
| + alterBeneficiary(_depositId, _newBeneficiary) |
| + alterBeneficiaryOnBehalf(...)         |
| + withdraw(_depositId, _amount)         |
| + withdrawOnBehalf(...)                 |
| + claimReward()                         |
| + claimRewardOnBehalf(_beneficiary, _signature) |
| + notifyRewardAmount(_amount)           |
+------------------------------------------+
| # _fetchOrDeploySurrogate(_delegatee)   |
| # _stakeTokenSafeTransferFrom(_from, _to, _value) |
| # _useDepositId()                       |
| # _stake(_depositor, _amount, _delegatee, _beneficiary) |
| # _stakeMore(...)                       |
| # _alterDelegatee(...)                  |
| # _alterBeneficiary(...)                |
| # _withdraw(...)                        |
| # _claimReward(_beneficiary)            |
| # _checkpointGlobalReward()             |
| # _checkpointReward(_beneficiary)       |
| # _setAdmin(_newAdmin)                  |
| # _revertIfNotAdmin()                   |
| # _revertIfNotDepositOwner(deposit, owner) |
| # _revertIfAddressZero(_account)        |
| # _revertIfSignatureIsNotValidNow(_signer, _hash, _signature) |
+------------------------------------------+
| > Events:                                |
| - StakeDeposited                         |
| - StakeWithdrawn                         |
| - DelegateeAltered                       |
| - BeneficiaryAltered                     |
| - RewardClaimed                          |
| - RewardNotified                         |
| - AdminSet                               |
| - RewardNotifierSet                      |
| - SurrogateDeployed                      |
+------------------------------------------+
| > Errors:                                |
| - UniStaker__Unauthorized                |
| - UniStaker__InvalidRewardRate           |
| - UniStaker__InsufficientRewardBalance   |
| - UniStaker__InvalidAddress              |
| - UniStaker__InvalidSignature            |
+------------------------------------------+
```

### DelegationSurrogate.sol

This contract is designed to maintain governance rights for token holders in pooled scenarios by delegating voting power to a chosen delegatee. This addresses the limitation where a single address can only delegate to one other address, potentially disenfranchising token holders in pools. By deploying a DelegationSurrogate for each delegatee and transferring or deploying tokens accordingly, users keep their governance participation intact. The contract itself focuses on delegating voting power and ensuring the deployer can reclaim tokens, with the deploying pool contract responsible for all related accounting.

```css
+------------------------------------------+
| DelegationSurrogate  8 nSLOC |
+------------------------------------------+
| - No state variables                     |
+------------------------------------------+
| + constructor(_token, _delegatee)        |
+------------------------------------------+
```

## Systemic Risks

- The `permit` griefung attack seems to be an accepted risk, which faults protocol since an attacket can directly determine the avaialability of the `permitAndStake()` & `permitAndStakeMore()` functions.
- There are currently no emergency stop functionality which would come in very handy if a red pill attack ever occurs.
- Consider implementing two-step procedure for updating protocol addresses, cause currently the admin functionality could become inacessible if a wrong admin is set.
- Protocol does not consider the fact that `UNI` is also a weird token and has this faulty logic in regards to approvals/transfers, [source](https://github.com/d-xo/weird-erc20?tab=readme-ov-file#revert-on-large-approvals--transfers). This should be clearly documented on and also incorporatiokn of tests purely on this should be made
- External dependencies from pragma oracle (floating in some contracts), inherited contracts, etc.
- Incorrect implementations of supported EIPs, not following the CEI pattern and not using local variables for emitting packed storage variables.

## Centralization Risks

- An admin could maliciously set the `PayoutAmount` very high affecting users or even the fee amount
- An admin could maliciously set the fee amount very high, since they are currently no limits in regards to this

## Recommendations

- Enhance documentation quality, currently the code's natspec and protocol's provided docs, do not always align, for example in the docs, it's stated that the permit grieffacnce attack is acceptable, but this is no where notified within in the code since this piece of information is now updated
- Onboard more developers, cause multiple eyes checking out a protocol can be invaluable. More contributors can significantly reduce potential risks and oversights. Evidence of where this could come in handy can be gleaned from codebase typos. Drawing from the [broken windows theory](https://en.wikipedia.org/wiki/Broken_windows_theory), such lapses may signify underlying code imperfections.
- Re-enhance event monitoring, current implementation subtly suggests that event tracking isn't maximized. Instances where events are missing or seem arbitrarily embedded have been observed.
- Improve protocol's testability, one thing to note about tests is that _there's always room for further refinement and improvement._, a few out of the box ideas need to also be attached to test cases.
- To support the above, even more fuzz and invariant tests could be further incorporated to help secure protocol.
- There seems to be a need to improve the naming conventions in some areas of protocol to ease user/developer experience while trying to use/understand protocol.
- Two step variable and address updates should be implemented including zero address checks. A timelock can also be considered for the setter functions to give users time to react to protocol changes. Adding these fixes can help protect from mistakes and unexepected behaviour.

## Security Researcher Logistics

My attempt on reviewing the Codebase spanned around 35 hours distributed over 4 days:

- 2 hours dedicated to writing this analysis.
- 3 hours (first day) spent exploring the whole system to grasp the foundations before diving into it line by line
- 2 hours were allocated for discussions with sponsors on the private discord group regarding potential vulnerabilities and also lurking in the public discord group to get more ideas based on questions other security researchers ask and explanation provided by sponsors
- The remaining time was spent on finding issues and writing the report for each of them on the spot, later on _editing a few with more knowledge gained on the protocol as a whole, or downgrading them to QA reports._

## Conclusion

The codebase was a very great learning experience, though it was a pretty hard nut to crack, due to this being linke an update contest and the quality of devs from uniswwap, nonetheless during my review, I uncovered a few issues within the protocol and they need to be fixed. Recommended steps should be taken to protect the protocol from potential attacks. Timely audits and codebase cleanup for mitigations should also be conducted to keep the codebase fresh and up to date with evolving security times.

## Resources

- [Documentation](http://docs.unistaker.xyz)

- [Website](https://aiarena.io/#/)

- [Audit Page](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/README.md)

- Contract for `PAYOUT_TOKEN` - [WETH](https://etherscan.io/token/0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2)

- Contract for `STAKE_TOKEN` - [UNI](https://etherscan.io/token/0x1f9840a85d5af5bf1d1762f925bdaddc4201f984)


### Time spent:
40 hours