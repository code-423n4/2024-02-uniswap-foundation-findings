#  **Advanced Analysis Report for Unistaker**

[Audit approach](#1-audit-approach)

[Brief Overview](#2-brief-overview)

[Scope and Architecture Overview](#3-scope-and-architecture-overview)

[Codebase Overview](#4-codebase-overview)

[Centralization Risks](#5-centralization-risks)

[Systemic Risks](#6-systemic-risks)

[Recommendations](#7-recommendations)

[Conclusions](#8-conclusions)

[Resources](#9-resources)

***

## **1. Audit approach**

**Phase 1**: Protocol Familiarization: The review began by analyzing the protocol documentation, including the readMe, AthleaLI blog and whitepaper. followed by a preliminary examination of the relevant contracts and their tests.

**Phase 2**: Deep Contract Review: A meticulous, line-by-line review of the in-scope contracts was conducted, meticulously following the expected interaction flow.

**Phase 3**: Issue Discussion and Resolution: Potential pitfalls identified during the review were discussed with the developers to clarify whether they were genuine issues or intentional features of the protocol.

**Phase 4**: Reporting: Audit reports were finalized based on the findings.

***
## **2. Brief Overview**

-  UniStaker aims to incentivize UNI holders to participate in the protocol governance and liquidity by offering rewards in exchange for staking and delegating their tokens. Uniswap Governance can use UniStaker to enable and manage fees on Uniswap V3. They can set the fees, but the collected fees are automatically distributed to stakers, not controlled directly.
-  Staking doesn't affect voting as stakers choose who can vote on their behalf, including themselves. The staking rewards aren't directly paid in the fee tokens. Instead, stakers earn a dedicated token based on how much UNI they have staked compared to everyone else. Stakers can also choose who receives their rewards. This allows someone to earn rewards on behalf of another person who initially staked the UNI.

***
## **3. Scope and Architecture Overview**

<p align="center">
	High-level Overview <br><br>
    <img width= auto src="https://gist.github.com/assets/112232336/d857cd02-86f0-4010-a7a2-d966175088a5"
alt="Contract Architecture">
</p>


***


<h3 align="center">
<b>System contracts</b>
</h3>

#### **[UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol)**

- This contract is responsible for distributing rewards to users who stake their rewards tokens. Staking involves locking up these tokens and delegating their voting rights to a designated address. Users can change this address at any time. Similar to the Synthetix system, rewards are distributed gradually over a set period and are proportional to the user's share of the total staked tokens. Users can add or remove their staked tokens at their own convenience, and beneficiaries can claim their earned rewards anytime. When new rewards are added, the distribution period restarts, and the rate is adjusted to reflect the new amount and any remaining unclaimed rewards.

**Creating and increasing staking position** - Users create a new staking position by calling the `stake` function entering in their desired beneficiary and delegatee addresses. Prior approval must have been given to the Unistaker contract in a two step transactional method. To combine the approval and stake process into one step, the `permitAndStake` function can be called which uses the token's permit functionality to allow for direct staking using a signature. Also, users can allow other users to open staking positions for them by calling the `stakeOnBehalf` function. Users can also increase their current staking positions through the `stakeMore`, `permitAndStakeMore` functions, or have someone do it for them by calling the `stakeMoreOnBehalf` functions which allows for increasing the total amount staked by the user. 

**Surrogate deployment** - When users stake, they're required to provide an address to which their voting power is delegated to. Doing this requires that the delegatee has a surrogate contract, which functions as an escrow for the user's tokens. The `_fetchOrDeploySurrogate` function attampts to find a delegatee's surrogate contract, and if its non existent, a new one is deployed using the CREATE method. 

**Reducing staking position** - Users can take away some of all of their tokens from stake by calling the `withdraw` function, which essentially behaves like a typical `unstake` function would. The amount to unstake is withdrawn from the position and tokens transferred to the user. This can also be done on behalf of the user by calling the `withdrawOnBehalf` function. Important to note that staking positions are not deleted even after all tokens have been unstaked. The position is held for users to fill up by calling the functions to stake more.

**Claiming staking rewards** - Beneficiaries of staking positions call the `claimReward` which calculates and send out the rewards that they're entitled to. They can allow other users to claim on the behalf using their signature through the `claimRewardOnBehalf` function.

**Changing beneficiary and delegates** - Upon staking, users specify the address to get rewards from their staking i.e the beneficiary, and address to get their voting power, the delegate. These addresses can be changed by calling the `alterBeneficiary` and the `alterDelegatee` functions. The `alterBeneficiary` sets the new beneficiary - Important to note that the previous beneficiaries do not lose their previous rewards, a checkpoint is set for them. The `alterDelegatee` transfers the voting power from the previous delegate to the new delegate. This can also be done for a user through the `alterBeneficiaryOnBehalf` and the `alterDelegateeOnBehalf` functions.

**Reward notification** - Being a fork of Synthetix staking contract, the contract implements the `notifyRewardAmount` function which alerts it that a new reward has been transfered. Before this function is invoked, the payout amount is sent first, so as to prevent issues when claiming rewards. 


```
                                                 UniStaker
                                                |
                                                |---> Constructor
                                                |     |
                                                |     |---> _setAdmin
                                                |
                                                |---> setAdmin
                                                |     |
                                                |     |---> _revertIfNotAdmin
                                                |
                                                |---> setRewardNotifier
                                                |     |
                                                |     |---> _revertIfNotAdmin
                                                |
                                                |---> stake
                                                |     |
                                                |     |---> _stake
                                                |          |
                                                |          |---> _checkpointGlobalReward
                                                |          |---> _checkpointReward
                                                |          |---> _fetchOrDeploySurrogate
                                                |          |---> _useDepositId
                                                |          |---> SafeERC20.safeTransferFrom
                                                |
                                                |---> permitAndStake
                                                |     |
                                                |     |---> STAKE_TOKEN.permit
                                                |     |---> _stake
                                                |
                                                |---> stakeOnBehalf
                                                |     |
                                                |     |---> _revertIfSignatureIsNotValidNow
                                                |     |---> _stake
                                                |
                                                |---> stakeMore
                                                |     |
                                                |     |---> _revertIfNotDepositOwner
                                                |     |---> _stakeMore
                                                |
                                                |---> permitAndStakeMore
                                                |     |
                                                |     |---> _revertIfNotDepositOwner
                                                |     |---> STAKE_TOKEN.permit
                                                |     |---> _stakeMore
                                                |
                                                |---> stakeMoreOnBehalf
                                                |     |
                                                |     |--->> _revertIfNotDepositOwner
                                                |     |---> _revertIfSignatureIsNotValidNow
                                                |     |---> _stakeMore
                                                |
                                                |---> alterDelegatee
                                                |     |
                                                |     |--- _revertIfNotDepositOwner
                                                |     |--- _alterDelegatee
                                                |
                                                |--- alterDelegateeOnBehalf
                                                |     |
                                                |     |--- _revertIfNotDepositOwner
                                                |     |--- _revertIfSignatureIsNotValidNow
                                                |     |--- _alterDelegatee
                                                |
                                                |--- alterBeneficiary
                                                |     |
                                                |     |--- _revertIfNotDepositOwner
                                                |     |--- _alterBeneficiary
                                                |
                                                |--- alterBeneficiaryOnBehalf
                                                |     |
                                                |     |--- _revertIfNotDepositOwner
                                                |     |--- _revertIfSignatureIsNotValidNow
                                                |     |--- _alterBeneficiary
                                                |
                                                |--- withdraw
                                                |     |
                                                |     |--- _revertIfNotDepositOwner
                                                |     |--- _withdraw
                                                |
                                                |--- withdrawOnBehalf
                                                |     |
                                                |     |--- _revertIfNotDepositOwner
                                                |     |--- _revertIfSignatureIsNotValidNow
                                                |     |--- _withdraw
                                                |
                                                |--- claimReward
                                                |     |
                                                |     |--- _claimReward
                                                |
                                                |--- claimRewardOnBehalf
                                                |     |
                                                |     |--- _revertIfSignatureIsNotValidNow
                                                |     |--- _claimReward
                                                |
                                                |--- notifyRewardAmount
                                                |     |
                                                |     |--- _revertIfNotRewardNotifier
```
<p align="center">
    UniStaker.sol; sLOC - 423
</p>

#### **[V3FactoryOwner.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol)**

- This contract functions as the owner contract to Uniswap V3 factory. Here, various admin functionalities on the factory contract can be accessed, being made available to the owner. These admin functions are controlled by Uniswap's governance timelock contract. The contract also functions as the reward notifier of the `Unistaker` contract, which sends rewards to it and notifies the contract that a new batch of rewards have been sent.

**Admin functions** - The uniswap governance timelock contract interact with the v3factory and pools through these functions. By calling the `enableFeeAmount`, fee amount for a pool is set and activated for collection. The `setFeeProtocol` sets the denominator of the protocol's percentage share of the fees. The `setPayoutAmount` allows admin to set the amount of the payout token that needs to be paid to claim the pools fees.

**Fees claiming** - While, arbitraging and race conditions, are mostly viewed in a negative light, the protocol offers a chance to MEV bots and arbitragers to participate in a race to claim a pools fees. The arbitragers have to pay the payout amount for which the fees from activated pools will be sent to them. The payout amount is send to the `Unistake` contract and the contract is notified of the payment, upon which rewards are distributed to stakers.
```
                                        [Admin] --> |V3FactoryOwner|
                                                    |
                                                    |<-- Admin Set
                                                    |
                                                    |--> |setAdmin()|
                                                    |
                                                    |--> |setPayoutAmount()|
                                                    |
                                                    |--> |enableFeeAmount()|
                                                    |
                                                    |--> |setFeeProtocol()|
                                                    |
                                                    |--> |claimFees()|
                                                    |       |
                                                    |       |--> |IUniswapV3PoolOwnerActions|
                                                    |       |
                                                            |       |--> |IUniswapV3FactoryOwnerActions|
                                                            |       |
                                                            |       |--> |INotifiableRewardReceiver|
                                                            |       |
                                                            |       |--> |IERC20|
                                                            |
                                                            |<-- FeesClaimed
```
<p align="center">
    V3FactoryOwner.sol; sLOC - 87
</p>

#### **[DelegationSurrogate.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/DelegationSurrogate.sol)**

- This is an escrow-esque contract deployed by the V3factoryOwner contract to hold users' staked tokens. It has the function of delegating the tokens to a user specified delegates and grants max token approval to the deployer.

```
                            +--------------------------------------------------+
                            |         DelegationSurrogate                      |
                            +--------------------------------------------------+
                            | - _token: IERC20Delegates                        |
                            | - _delegatee: address                            |
                            +--------------------------------------------------+
                            | + constructor(_token.delegate, _token.approve)   | 
                            +--------------------------------------------------+
```
<p align="center">
    DelegationSurrogate.sol; sLOC - 8
</p>

***

<h3 align="center">
<b>Interfaces</b>
</h3>

	
#### **[IERC20Delegates.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/IERC20Delegates.sol)**	

- This holds the interface of a simplified version of the ERC20Votes governance token standard that UNI follows. It includes the basic functionalities of ERC20 tokens and delegation, but it does not include functionalities related to check pointing, past voting weights, and other features


#### **[INotifiableRewardReceiver.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/INotifiableRewardReceiver.sol)**

- This interface, allows the V3FactoryOwner contract to easily send rewards to the UniStaker contract with its major concern being thatw that the UniStaker has received them. 

#### **[IUniswapV3FactoryOwnerActions.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/IUniswapV3FactoryOwnerActions.sol)**

- This acts as the interface tool for creating trading pools and managing fees collected within the protocol.

#### **[IUniswapV3PoolOwnerActions.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/IUniswapV3PoolOwnerActions.sol)**

- This is the interface for pool methods that may only be called by the uniswap factory owner.


***
## **4. Codebase Overview**

**Audit Information** - For the purpose of the security review, the Unistaker codebase consists of three smart contracts totaling 518 SLoC. Its core design principle is inheritance, enabling efficient and flexible integration. It is scheduled to be deployed on Ethereum mainnet, hence subject to its network conditions. It exists as a fork of the Synthetix staking contracts.

**Documentation and NatSpec** - The provided c4 readme and protcol documentation is top tier. They provide a important overview of the the protocol and its functionality including a breakdown of each contract functions and implementations. The contracts are well commented, each function was well explained. It made the audit process easier. 

**Error handling and Event Emission** -  Errors are well handled, custom errors are in use, which among many advantages, is also gas efficient. Events are well handled, emitted for important parameter changes.

**Testability** - Test coverage is about 100% which is very excellent. The implemented tests include basic unit tests for functions, invariant tests to handle the protocol's main invariants and fuzz tests for the more complicated parts of the codebase.

**Token Support** - The protocol aims to work with 3 major token types, the unistaker's reward token - WETH, stake token - UNI and the V3FactoryOwner's payout token - WETH. Ideally, any token type can be set by the admin, upon deployment which includes all token types. However, it's important to note that the contracts' implementations are not suitable for most non-standard ERC20 tokens.

**Attack Vectors** - Points of attack in this contract include gaming rewards distribution, stealing other user's rewards, griefing users' claims and so on. Other areas of attack are vulnerabilities common to synthetix forks and breaking the protocol's main invariants.

***

## **5. Centralization Risks** 

Ideally, the admin functions in the contracts are controlled by uniswap's governance timelock which is controlled by the DAO, so its fair to say there isn't a lot of centralization going on. However, there are ways these admin functions can be used to harm the protocol. Some of them include:

- Setting the payout amount so high that its not profitable for any arbitragers to claim fees, and therefore, no rewards will be distributed to the stakers.
- Disabling the V3FactoryOwner's status as the reward notifier, there by causing fees claims to always fail, and no rewards distributed to the users. 

***
## **6. Systemic Risks** 

- The system is highly dependent on arbitragers claiming fees and users staking their UNI tokens.
- Extreme token price changes in both direction can affect frequency of fees claiming and as a consequence, rewards distribution.  
- Issues common to forks of the Synthetix staking contract.
- Issues with external dependencies, solidity version bugs and open zeppelin contracts. 
- Issues from other smart contracts, which at best may be contained in the erring contracts, at worst affect the whole protocol.
***
## **7. Recommendations**

- The documentation is still missing information about fees and rewards claims. Recommend bringing it up to date.  

- Edge cases for users not staking or a user staking very little to manipulate rewards should be considered. A potential fix is allowing a minimum stake amount.

- Reconsider the decision to have a general payout amount. Due to the difference in pools' liquidity and activity, certain pools might get their fees claimed too often, while some may not have fees claimed at all. A possible fix can be making the payout amounts pool specific.

- A bit of refactoring should be made around the transfer functions and accounting to factor in payout tokens that may not fit the typical ERC20 standard, e.g fee-on-transfer tokens, rebasing tokens etc in case supporrt is offered for them later on. 

- The deployed DelegationSurrogate contract can be beefed up with CREATE2 or CREATE3 to protect from ethereum network conditions like reorgs, forks etc.

- A proper pause function should be implemented to protect users during unstable situations. Important to note that only user entry methods should be pausable.

- Two step variable and address updates should be implemented. Most of the setter functions implement the changes almost immediately which can be jarring for the contracts/users. Adding these fixes can help protect from admin mistakes and unexepected behaviour.

- A sweep function can also be considered for the contracts to carefully retrieve excess tokens sent to them to prevent them from getting lost forever.


***
## **8. Conclusions**

In conclusion, the codebase is probably one of the best-written ever audited on C4, which is quite commendable. Howevever, as is the reason for the audit, the identified risks need to be mitigated. Provided recommendations should be considered and implemented or if not implemented, then documented for user awareness.

***
## **9. Resources**

- [C4 ReadMe](https://code4rena.com/audits/2024-02-unistaker-infrastructure#top)
- [Documentation](https://docs.unistaker.io/)

### Time spent:
32 hours