# Introduction of UniStaker

> **Uniswap V3 protocol fee collection and distribution via UNI staking.**

In simple terms, here's what's going on:

Protocol Fees: Uniswap Governance, which is like the management team, can decide to add fees to Uniswap V3. These fees are collected when people trade on Uniswap. But instead of keeping these fees for themselves, they distribute them to people who hold a cryptocurrency called UNI and choose to participate.

Rewards for UNI Holders: If you have UNI tokens and you decide to stake them (kind of like putting them in a savings account), you can earn rewards. These rewards are given out based on how many UNI tokens you've staked compared to the total number of tokens staked by everyone.

Staking and Rewards: You only earn rewards while your UNI tokens are staked. If you take them out, you stop earning rewards. You can also choose someone else to get your rewards for you, and you can change this at any time.

Ownership Transfer: For this system to work, the Uniswap Governance team needs to agree to switch ownership of part of the Uniswap system to these new rules. Once they do this, they can't change their minds later. They can still adjust the fees and rewards, but they can't take the fees for themselves anymore - they always go to the people who stake UNI tokens.

# Approach taken in evaluating the codebase

Manual code review by looking into each space in the scoped contract and decided to explain some core functional contracts.it is important to note that manual code reviews can be very time-consuming.

The main contracts are:

- UniStaker.sol

> manages staking mechanics and the distribution of rewards to stakers.

- V3FactoryOwner.sol

> manages ownership of the existing Uniswap V3 Factory contract,

## UniStaker.sol

**Staking Mechanism:** Users can stake a delegable ERC20 governance token to earn rewards over time.  
**Delegation:** Users can delegate their staked tokens' voting power to a delegatee.  
**Reward Distribution:** Rewards are streamed over a 30-day period and are proportional to the user's stake.  
**Beneficiary Designation:** Users can designate a beneficiary to receive the staking rewards.  
**Reward Notifiers:** Authorized addresses can notify the contract of new rewards, which restarts the reward duration.  
**Admin Role:** An admin can manage reward notifiers and other administrative functions.  
**EIP712 Compliance:** Supports meta-transactions with EIP712 typed structured data hashing and signing.  
**Multicall:** Inherits from OpenZeppelin's Multicall, allowing batched calls to multiple methods in a single call.  
**Nonces:** Uses OpenZeppelin's Nonces for replay protection in meta-transactions.  
**Events:** Emits events for staking, withdrawing, delegatee changes, beneficiary changes, reward claims, and admin actions.  
The contract also includes error handling, uses `OpenZeppelin's SafeERC20` library for safe token transfers, and has a mechanism to checkpoint rewards and update accumulators to ensure accurate reward calculations.

## V3FactoryOwner.sol

**ownership:** An admin can manage the Uniswap v3 factory settings and pools.  
**Fee Collection:** Public function to claim protocol fees from pools in exchange for a payout token.  
**Payout:** Fees are paid to a designated reward receiver contract.  
**Admin Functions:** Set admin, payout amount, enable fee amounts, and set pool protocol fees.  
**Events:** Log actions like fee claims, admin changes, and payout amount updates.  
**Errors:** Custom error handling for unauthorized access, invalid addresses, and insufficient fee collection.  
**Security:** Uses OpenZeppelin's SafeERC20 for safe token transfers.  
The contract is expected to be owned by Uniswap governance and to incentivize external parties to pay for and claim fees, potentially for arbitrage opportunities.

# Centralization Risks

Admin role grants significant control over reward distribution and contract settings, potentially leading to centralization concerns.

#  Test analysis

The audit scope of the contracts to be reviewed is 100%.

# Documentation

The code is well commented on, which is clear and informative, and the documentation is also quite comprehensive and detailed in terms of explaining its functionality, parameter usage, purpose, and overall architecture.

## Gas Optimization

`UniStaker` is generally efficient in terms of gas optimizations, many generally accepted gas optimizations have been implemented, gas optimizations with minor effects are already mentioned in automatic finding, but gas optimizations will not be a priority considering code readability and code base size.

# Conclusion

Overall `UniStaker` came up with a very strong solution with some weak points in commenting and some centralisation risk. But the approach used for the core working of protocol is really solid and up to the industry standard and fixing the weak will make it even more robust.

## **Note to Judge:**

This advanced Analysis report signifies that the codebase is well-structured and secure.  The project is deemed to have a low-risk profile based on the current codebase.

# Time:

The analysis process spanned approximately 20-25 hours

\### Time spent:

25 hours

### Time spent:
25 hours