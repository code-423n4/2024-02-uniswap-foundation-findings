## Summary
||Title|Description|
|:-:|:----|:----------|
|[1](#overview-of-the-unistaker-protocol)|Overview of the UniStaker protocol|Services and Features provided by the protocol|
|[2](#system-overview)|System Overview|Illustrating the components of the system|
|[3](#system-architecture)|System Architecture|Explaining the Architecture using workflow Diagram|
|[4](#documentation)|Documentation|Overview of the documentation and its quality|
|[5](#centralization-risks)|Centralization Risks|Risks that should be taken into consideration|
|[6](#my-view-on-the-project)|My view on the project|My opinion about the protocol's success when it launches|
|[7](#time-spent-on-analysis)|Time spent on analysis|The total time I spent analyzing and auditing the codebase| 

---

## Overview of the UniStaker protocol

UniStaker will be used as an infrastructure for all UniSwap foundation protocols and upcoming protocols. It will be the gate for all Stakeholders to earn yields in `WETH` token by investing in this protocol and putting `UNI` token in it.

Rewards will be distributed to the stakes using time-weighted contributions staking algorism.

Fees will be collected from different Uniswap Projects (V3 pools will be activated at this moment) and reward the stakers with that fees.

The protocol fees will be collected by anyone, but the caller should pay an amount of `WETH` token to fire this function and claim protocol fees earned from Uniswap-V3 pools.

Once claiming the fees, `WETH` will be sent to The `Staker` contract, and stakers will earn yields (Amount of `WETH`).

The admin is the one who can control the amount to be paid to claimFees, the pools that will get activated for protocol fees, and the contracts (notifiers) that can increase the staking amount in `Staker` contract.

The admin is willing to be the Uniswap Governance contract, so the stakers who hold their staking tokens will not only earn yields by staking, but also will have the power to vote for the changes that can happen that will affect `Staking` contract (adjusting the payoutAmount, or the pools to colect fees from).

Stakers (people that hold `UNI` in the staker contract), can claim their rewards, withdraw and take their `UNI` tokens if they have no plans to stake, and vote on proposals that can be made by the Governance contract to accept or deny a certain change.

---

## System Overview

The protocol consists of 2 main Components
1. Staking contract
2. Notifier Contracts

### Staking Contract
The staking contract `UniStaker.sol` has a lot of public functionalities that can be fired by anyone:
- Stake their `UNI` tokens.
- The staker provides 2 additional addresses (beneficiary and delegatee)
  - `beneficiary` is the address that will receive rewards, the user can make this his address or another address.
  - `delegatee` is that address that will have the power of votes for `UNI` token.
- Stakes happen using `IDs`, so the same user can stake more than one time, and he will be the owner of two different `deposit IDs`. this will be helpful if he wants to separate his tokens for different `beneficiaries`, and `delegatees`.
- The user who staked for a given `deposit ID`, can increase the value of his `deposit ID` by staking more `UNI` tokens to the same `deposit ID`.
- The user can `withdraw` his staking tokens from a given `deposit ID` anytime.
  - If the user staked more than one time and has more than one `deposit ID`, he can not unstake all his tokens at once. He needs to call `withdraw` two times to withdraw, providing `deposit ID` for each, as the withdrawal happens by providing single `deposit ID`.
- The `beneficiary` address can `claim` his rewards.
  - `beneficiary` can either claim all his rewards at the time of calling or leave them, he can not `withdraw` part from his earnings.
  - The depositor who staked the token can not claim `beneficiary` rewards, he can just unstake tokens, preventing the `beneficiary` from earning.
- The deposited (The owner of a given deposit ID), can change his deposit ID configurations, this includes:
  - Changing the `beneficiary` address, that will receive rewards.
  - Changing `delegatee` address that has the power of voting.
  - Increasing the amount of `deposit ID` balance (staked token), by staking more tokens to the same `deposit ID`.
- All the functions called by the users including staking, withdrawing, claiming, or configuration of a certain deposit ID, can be called without a direct call from the `owner`. The `owner` of the deposit ID can sign the message, and allow 3rd party to fire his transaction. Staking a new `deposit ID` allows this functionality too.

_Other functions are only callable by the admin of the contract these include:_

- Adding a new reward notifier, which is the address that can increase the staking reward amount.
- Changing the admin address.

### Notifier Contract (V3FactoryContract)
The system is designed to be able to have more than one notifier contract. In the context of this codebase, there is only one notifier contract (`V3FactoryOwner`), which is responsible for claiming fees from Uniswap v3 DEX pools.

`V3FactoryOwner.sol` characteristics:

- It will be the owner of the uniswapV3 factory contract, which is used to deploy new pools.
- Have an admin role, which is planned to be set to the Uniswap Governance, and it can do the following:
  - Adding new `fee/tick` support for the future pools that will be deployed (used to manage concentrated liquidity).
  - Enabling protocol fees to a given uni-v3 pool.
  - Changing the admin of the contract.
- Have a function that allows anyone to claim the fees from a given pool, but the caller should pay an amount of `WETH` first, to claim the pool fees.

### UNI tokens holder (DelegateSurrogate)
Once stakers stake their tokens, they provide the `delegatee` address which will have the power of the tokens held by the stakers.

The tokens themselves will not be with the `delegatee` address, just the voting power will be set to `delegatee` (this is achievable as `UNI` is a vote token). The tokens will be sent to a surrogate contract address, which will be like a safe place for Staked tokens. And these tokens can be withdrawn from this contract anytime from the `UniStaker` contract When withdrawing,


## System Architecture

The system works 100% on-chain, with no oracles, and no Web2 development integrations.

![UniStaker Protocol diagram](https://i.ibb.co/1b5kLfT/unistaker-analysis-2.png)

- The Staking Contract (`UniStaker`) uses time-weighted contributions staking algorism, which distributes rewards according to the amount staked and the duration these staked tokens were in the contract.
- Distributing Staking rewards are restricted to the `notifier` contracts.
- The `notifier` contract can only distribute reward after transferring the amount of the reward to the `UniStaker`.
- Users are incentivized to transfer rewards, as they will claim the protocol fees collected from the univ3-pool passed.
- Stakers will have the power to vote for Uni Governance contract using their `UNI` voting power which is set to `delegatee`

---

## Documentation
The documentation was impressive. UniSwap Devs provided detailed information that helps us in our auditing process, this includes:

- Detailed explanation of the system.
- Providing diagrams that facilitate understanding of the protocol.
- Pointing out the known issues, and the points that we should focus on.
- Detailed comments for every function in the .sol files, which explain what is the purpose of the function.

---

## ‎Centralization risks‎

The system will face centralization risks at the beginning of its deployment. But since it is planned to make the admin of the Staker contract and the notifier contracts the Governance contract of the `Uniswap`, The centralization risk will not stay.

---

## My view on the project

The project is impressive. I liked it. The project is insane in my opinion, because of the following features:

- The system will be controlled by the Uni Governance contract, which uses `UNI` token for proposing and voting. and this token which will control the governance contract. So the stakers themselves will be able to control how the system they are investing in should work, and this is a good point.
- Users' funds Staked tokens are safe, and they can receive them back anytime, even the admin can not control their tokens.
- The system is designed with great flexibility, where users can sign messages, without direct contact with the protocol.
- The reward token is `WETH` which is a well-known token, packed by native ETH.

---

## Time spent on analysis
I spent 6 days auditing, reporting, and analyzing the codebase, with a total hours ~30 Hours.

### Time spent:
30 hours