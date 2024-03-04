# Analysis - UniStaker Infrastructure Contest

![uni](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/unistaker.png?raw=truess)

## Description overview of `UniStaker Infrastructure` Contest

**Overview:**

UniStaker is a system that enables Uniswap protocol fee collection and distribution to UNI token holders who stake their tokens and delegate their governance voting weight.

**Key Features:**

- **Fee Collection:** Uniswap Governance can enable and manage protocol fees on Uniswap V3. Fees are collected permissionlessly.
- **Governance Delegation:** Stakers must delegate their governance voting weight either to themselves or another entity.
- **Rewards Accumulation:** Rewards are not accumulated in fee tokens but in a designated token. Fees are auctioned to entities who pay the designated token in exchange for fee tokens.
- **Staking Mechanics:** Stakers earn rewards proportional to their share of staked UNI. Rewards are earned only while tokens are deposited. Stakers can designate a beneficiary for their rewards.
- **Adoption Path:** Uniswap Governance must pass a vote to upgrade ownership of the V3 factory to this system. Once upgraded, ownership cannot be revoked.

**Benefits:**

- **Trustless Fee Distribution:** Fees are distributed to stakers without the need for intermediaries.
- **Increased Governance Participation:** Stakers are incentivized to participate in governance by earning rewards.
- **Rewards Diversification:** Rewards are not accumulated in fee tokens, reducing concentration risk.
- **Future Revenue Sources:** The system allows for the integration of other contracts as revenue sources for staking rewards.

## System Overview

### High-level System Overview

![UniStaker Infrastructure](https://gist.github.com/assets/70327041/41d9e715-d849-41ef-a730-d60bdae91326)

### Scope

**UniStaker.sol**: The `Unistaker` contract allows `UNI` token holders to stake their tokens and earn rewards from fees collected on Uniswap V3 pools. It provides a way to distribute Uniswap protocol fees to UNI stakers through a staking mechanism.

- **Key Features**:

  - **Fee Collection:** It allows Uniswap governance to enable and manage protocol fees on V3 pools, but the fees themselves are distributed to staked UNI holders, not controlled by governance.

  - **Governance Delegation:** UNI stakers must delegate their voting power to themselves or another entity when staking. This retains their ability to participate in Uniswap governance.

  - **Rewards Accumulation:** Rewards are not paid out in the fee tokens directly, but accumulate in a token defined when Unistaker is deployed. The fee tokens are continually auctioned off in exchange for the rewarded token.

  - **Staking Mechanics:** Stakers earn reward shares proportional to their UNI stake. Rewards are earned only while tokens are deposited. Stakers can designate a reward beneficiary address. Deposits are individually tracked and managed.

  - **Adoption Path:** Uniswap governance must vote to transfer V3 contract ownership to Unistaker for it to take effect. This irrevocably relinquishes fee token control to the staking rewards system.

  - **Staking Balances:** Stakers can add or withdraw from a particular deposit balance at any time. This adjusts their percentage of the UNI supply staked. Withdrawing stops future rewards on that balance.

  - **Reward Beneficiary:** The designated beneficiary address continues earning rewards even if the original depositor withdraws completely. This allows yield farming by staking on others' behalf.

  - **Governance Delegation:** Staked UNI voting power is allocated to the delegated address. This retains governance participation without losing the ability to earn staking rewards.

  - **Multiple Deposits:** A single address can manage numerous independent deposit balances, each with unique settings like beneficiary or delegation.

  - **Future Revenue:** Additional protocols beyond Uniswap can integrate their fees/rewards into Unistaker over time to expand the staking yield opportunities.

**V3FactoryOwner.sol**: The `V3FactoryOwner` contract provides a mechanism for the Uniswap Governance timelock to manage the Uniswap v3 factory and its pools. It allows the timelock to enable fee amounts, set protocol fees, and collect protocol fees. The fee claiming feature creates a competitive market for seekers to arbitrage the fees for pools that have accumulated significant protocol fees.

- **Key Features**

  - **Admin Control:** The admin can enable fee amounts on the factory, set protocol fees on pools, and change the admin address.
  - **Fee Claiming:** Anyone can pay the payout amount in a designated token to claim protocol fees from a pool. The fees are then sent to a recipient specified by the caller.
  - **Payout Token and Amount:** The contract uses a specific ERC-20 token as the payout token, and the amount required to claim fees is set by the admin.
  - **Reward Receiver:** The contract notifies a designated reward receiver when fees are claimed.

- **Fee Claiming Function**

  The `claimFees` function allows anyone to claim protocol fees from a pool by paying the payout amount. The caller specifies the recipient address and the amounts of token0 and token1 to request from the pool. The contract transfers the payout amount to the reward receiver and collects the fees from the pool.

**DelegationSurrogate.sol**: The `DelegationSurrogate` contract is a simple and effective way to delegate voting power while maintaining token custody. It is particularly useful in scenarios where multiple users share ownership of governance tokens and want to delegate their voting weight to different addresses.

- **Key Features**

  - **Delegation:** The contract delegates its voting weight to a designated delegatee.
  - **Token Holding:** The contract holds governance tokens on behalf of users.
  - **Max Approval:** The contract approves the deployer to transfer all tokens, allowing for easy token recovery.

**interfaces**

- **IERC20Delegates.sol**: The `IERC20Delegates` interface is used by contracts that need to interact with governance tokens in a way that is compatible with the Uniswap system. For example, the `DelegationSurrogate` contract uses this interface to hold governance tokens on behalf of users and delegate voting power to a specified delegatee.

- **INotifiableRewardReceiver.sol**: The `INotifiableRewardReceiver` interface is a simple and effective way for the `V3FactoryOwner` contract to communicate with the `UniStaker` contract. It allows the `V3FactoryOwner` contract to notify the UniStaker contract when protocol fees have been claimed, so that the UniStaker contract can distribute the rewards to its stakeholders.

- **IUniswapV3FactoryOwnerActions.sol**: The `IUniswapV3FactoryOwnerActions` interface is a simple and effective way for the factory owner to manage the factory's settings and to enable new fee amounts.

- **IUniswapV3PoolOwnerActions.sol**: The `IUniswapV3PoolOwnerActions` interface is a simple and effective way for the pool owner to manage the pool's protocol fee and to collect protocol fees.

### Roles

1. **UniStaker.sol**:

   - The main roles in the UniStaker contract are:

     - **Stakers:** Users who deposit the `STAKE_TOKEN` to earn rewards. They can stake, stake more, withdraw stakes, and claim rewards.

     - **Beneficiaries:** The addresses designated by stakers to receive reward tokens earned from their stakes.

     - **Delegatees:** The addresses to which stakers delegate the voting power of their staked tokens.

     - **Surrogate contracts:** Contracts that hold the staked tokens on behalf of delegatees. A new one is deployed for each unique delegatee.

     - **Reward notifiers:** Addresses authorized by the admin to notify the contract of new reward amounts.

     - **Admin:** The permissioned actor who can set the admin address and enable/disable reward notifiers.

     - **UniStaker contract:** Handles all staking, rewards distribution, and governance delegation logic. Stores deposit data, distributes rewards proportionally, checkpoints balances.

     - **REWARD_TOKEN:** The token rewards are denominated in.

     - **STAKE_TOKEN:** The delegable governance token users stake to earn rewards.

2. **V3FactoryOwner.sol**

   - **Admin**: The address that can call privileged methods, including passthrough owner functions to the factory itself.

   - **Factory**: The instance of the Uniswap v3 factory contract which this contract will own.

   - **Payout Token**: The ERC-20 token in which payouts will be denominated.

   - **Payout Amount**: The initial raw amount of the payout token required to claim fees from a pool.

   - **Reward Receiver**: The contract that will receive the payout when fees are claimed.

### Invariants Generated

- The contract’s `totalStaked` variable is equal to the sum of all deposits less the sum of all withdrawals.

- The sum of the governance token balances of all the surrogate contracts is equal to the sum of all deposits less the sum of all withdrawals.

- The sum of all users’ `depositorTotalStaked` amounts is equal to the value of `totalStaked`.

- The sum of all users’ deposits balances is equal to the sum of all deposits less the sum of all withdrawals.

- The sum of the amounts delegated to `delegatees` is equal to the contract’s `totalStaked` variable.

- The sum of all amounts applied to `beneficiaries` is equal to the contract’s `totalStaked` variable.

- The sum of all `beneficiaries’ earningsPower` amounts is equal to the sum of all deposits less the sum of all withdrawals.

- The sum of the increases in `beneficiaries’ reward token balances` is equal to the rewards distributed by the system.

- The sum of the increases in `beneficiaries’ reward token balances` plus the reward token balance of the UniStaker contract is equal to the sum of all rewards notified, plus the sum of all reward token donations, less the sum of rewards claimed, less the sum of all rewards not transferred in during reward notification.

- The UniStaker contract’s `reward token balance` is equal to the sum of all rewards notified, plus the sum of all reward token donations, less the sum of rewards claimed, less the sum of all rewards not transferred in during reward notification.

- The `lastCheckpointTime` variable is greater than or equal to the previous value.

- The `rewardPerTokenAccumulatedCheckpoint` amount is greater than or equal to the previous amount.

## Approach Taken-in Evaluating UniStaker Infrastructure Protocol

Accordingly, I analyzed and audited the subject in the following steps;

1.  **Core Protocol Contract Overview**:

    I focused on thoroughly understanding the codebase and providing recommendations to improve its functionality.
    The main goal was to take a close look at the important contracts and how they work together in the UniStaker Infrastructure.

    I start with the following contracts, which play crucial roles in the UniStaker Infrastructure:

    **Main Contracts I Looked At**

    I start with the following contracts, which play crucial roles in the UniStaker Infrastructure:

                UniStaker.sol
                V3FactoryOwner.sol
                DelegationSurrogate.sol

    I started my analysis by examining the intricate structure and functionalities of the `UniStaker Infrastructure` protocol which allows for the distribution of protocol fees generated by Uniswap V3 to UNI stakers, while preserving their ability to participate in governance. It's basically a system designed to help users who own UNI tokens in the Uniswap network. The main part of this protocol is the `UniStaker.sol` contract. It allows UNI token holders to stake their tokens and earn rewards from fees collected on Uniswap V3 pools. This setup encourages users to get involved and rewards them for their participation.

    Another important piece is the `V3FactoryOwner.sol` contract. It gives control over managing Uniswap V3 pools and the fees collected from them. This ensures fairness and transparency in how fees are managed and claimed. Then there's the `DelegationSurrogate.sol`, which helps users delegate their voting power while keeping their tokens safe.

2.  **Documentation Review**:

    Then went to Review [this docs](http://docs.unistaker.xyz/) for a more detailed and technical explanation of the UniStaker Infrastructure.

3.  **Compiling code and running provided tests**:

4.  **Manuel Code Review** In this phase, I initially conducted a line-by-line analysis, following that, I engaged in a comparison mode.

    - **Line by Line Analysis**: Pay close attention to the contract's intended functionality and compare it with its actual behavior on a line-by-line basis.

    - **Comparison Mode**: Compare the implementation of each function with established standards or existing implementations, focusing on the function names to identify any deviations.


## Codebase Quality

Overall, I consider the quality of the UniStaker Infrastructure protocol codebase Excellent. The code appears to be very mature and well-developed with alot of Invariants and Unit test. We have noticed the implementation of various standards adhere to appropriately. Details are explained below:

| Codebase Quality Categories              | Comments                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| ---------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Architecture & Design**                | The protocol features a modular design, segregating functionality into distinct contracts (e.g., `UniStaker`, `V3FactoryOwner`, `DelegationSurrogate`) for clarity and ease of maintenance. The use of interfaces like IERC20Delegates,IUniswapV3FactoryOwnerActions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| **Community Governance & Participation** | The protocol incorporates mechanisms for community governance, enabling token holders to influence decisions. This fosters a decentralized and participatory ecosystem, aligning with the broader ethos of blockchain development.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| **Error Handling & Input Validation**    | Functions check for conditions and validate inputs to prevent invalid operations, though the depth of validation (e.g., for edge cases transactions) would benefit from closer examination.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| **Code Maintainability and Reliability** | The contracts are written with emphasis on sustainability and simplicity. The functions are single-purpose with little branching and low cyclomatic complexity. The protocol includes a novel mechanism for collecting fees and sending them to the staking contract that offers an incentive for this work to be done by outside actors,thereby removing the associated complexity.                                                                                                                                                                                                                                                                                                                                                          |
| **Code Comments**                        | The contracts are accompanied by comprehensive comments, facilitating an understanding of the functional logic and critical operations within the code. Functions are described purposefully, and complex sections are elucidated with comments to guide readers through the logic. Despite this, certain areas, particularly those involving intricate mechanics or tokenomics, could benefit from even more detailed commentary to ensure clarity and ease of understanding for developers new to the project or those auditing the code.                                                                                                                                                                                                   |
| **Testing**                              | The codebase includes a comprehensive fuzz-testing suite with 100% code coverage. The contracts exhibit a commendable level of test coverage which is indicative of a robust testing regime. This coverage ensures that a wide array of functionalities and edge cases are tested, contributing to the reliability and security of the code and the incorporation of fuzz testing and invariant testing.                                                                                                                                                                                                                                                                                                                                      |
| **Code Structure and Formatting**        | The codebase benefits from a consistent structure and formatting, adhering to the stylistic conventions and best practices of Solidity programming. Logical grouping of functions and adherence to naming conventions contribute significantly to the readability and navigability of the code. While the current structure supports clarity, further modularization and separation of concerns could be achieved by breaking down complex contracts into smaller, more focused components. This approach would not only simplify individual contract logic but also facilitate easier updates and maintenance.                                                                                                                               |
| **Strengths**                            | The UniStaker protocol strengthens Uniswap V3 by efficiently collecting protocol fees and distributing rewards to staked UNI holders, ensuring active participation in governance decisions. Inspired by Synthetix, it offers proportional reward shares based on staked UNI, fostering fair distribution. Stakers can delegate governance voting weight, withdraw rewards, and designate beneficiaries flexibly, enhancing user engagement. With a secure adoption path via governance voting, UniStaker guarantees irrevocable fee distribution control, fostering trust and long-term commitment. Its adaptable design allows for potential integration of additional revenue sources, promising sustained growth and yield opportunities. |
| **Documentation**                        | The NatSpec is mostly complete for all external functions,and there are helpful inline comments throughout. However, there currently is no external documentation for users or integrators. Additionally, some user-facing documentation does not identify the risks and nuances of the staking contract, which is important. Technical developer documentation would also be helpful for integrators or MEV searchers interested in collecting fees.                                                                                                                                                                                                                                                                                         |

## Architecturee

The UniStaker system is designed with simplicity and security as its primary goals. It optimizes for these properties.

The system has two core components, the UniStaker contract and the V3FactoryOwner contract. The system is designed to interoperate with existing onchain infrastructure, including Uniswap V3 and the Uniswap Governance contracts.

### Components

**UniStaker**

The core component of the system is the UniStaker contract implemented in `UniStaker.sol`. It manages the distribution of rewards to stakers and the delegation of their governance voting weight. It also allows stakers to designate a beneficiary address, that is, the address that will accrue the rewards distributed.

![UniStaker Infrastructure-1](https://gist.github.com/assets/70327041/076915a6-7ebb-4405-8c2f-04b6db0f3ea7)

The UniStaker contract does not collect rewards. Instead, it outsources the collection of rewards to "rewards notifier" contracts which must be whitelisted by the UniStaker contract's `admin` role. This role is intended to be held by Uniswap Governance.

**V3FactoryOwner**

The first source of rewards being proposed are protocol fees from Uniswap V3 Pools. These fees are currently not active on any pools. The right to turn these fees on is controlled by the owner role of the [UniswapV3Factory](https://docs.uniswap.org/contracts/v3/reference/core/UniswapV3Factory) contract. That role is currently held by Uniswap Governance, specifically by the governance timelock contract.

The V3FactoryOwner contract implemented in `V3FactoryOwner.sol` is intended to replace the timelock contract as the owner of the V3 Factory. For this to occur, Uniswap Governance must pass a governance proposal that transfers the ownership, while enshrining itself as the `admin` of the new `V3FactoryOwner` instance.

### Composition

The components of the system interact with each other, and with the other contracts onchain, across simple abstraction boundaries. Each component assumes as little as possible about other parts of the system.

![UniStaker Infrastructure-1](https://gist.github.com/assets/70327041/3f4061d5-b327-4689-907e-1c78e15fb598)

Here we describe the relationships depicted in the diagram above:

- UNI token holders stake tokens and delegate their governance voting weight in the UniStaker contract.
- Uniswap Governance is the `admin` of the `V3FactoryOwner` contract.
- The `V3FactoryOwner` contract is the owner of the `UniswapV3Factory`.
- The `V3FactoryOwner` contract has passthrough methods which allow it set protocol fees on Uniswap V3 Pools. Only Uniswap Governance can call these methods.
- The `V3FactoryOwner` allows external parties to profitably collect fees from pools by calling the claimFees method. In exchange, they must provide a fixed amount of the reward token.
- Rewards are immediately transferred to the UniStaker contract.
- The `V3FactoryOwner` contract is a whitelisted rewardNotifier in the UniStaker contract.
- The `V3FactoryOwner` notifies the UniStaker contract when it receives a reward from an external party.
- UNI stakers earn and claim rewards from the UniStaker contract.

| File Name                 | Core Functionality                                                                                                                                                                                                                 | Technical Characteristics                                                                                                                                                                                                                                                                                                                                                                                                                                              | Importance and Management                                                                                                                                                                                                                                                                                                                                                           |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `UniStaker.sol`           | The `UniStaker` contract manages the distribution of protocol rewards from Uniswap V3 to UNI stakers through a flexible staking mechanism.                                                                                         | The contract utilizes structures, enums, errors, events, multi-call and signature validation following best practices for gas-optimized, secure smart contract design.                                                                                                                                                                                                                                                                                                 | As the primary system for distributing fees from a major decentralized exchange, its robust implementation and prudent admin control are essential for preserving stakeholders' trust and incentives over time.                                                                                                                                                                     |
| `V3FactoryOwner.sol`      | The `V3FactoryOwner` contract allows the Uniswap Governance timelock to manage the Uniswap v3 factory and its pools. It provides a mechanism for the timelock to enable fee amounts, set protocol fees, and collect protocol fees. | The contract has an admin address that can call privileged methods, including passthrough owner functions to the factory itself. also use a specific ERC-20 token as the payout token, which is used to pay for fees when claiming pool fees.The contract also has a payout amount, which is the raw amount of the payout token required to claim fees from a pool and has a reward receiver address, which is notified when fees are claimed and receives the payout. | The V3FactoryOwner contract is important because it allows Uniswap Governance to manage the Uniswap v3 factory and its pools, and to collect protocol fees. The contract is managed by the admin address, which can call privileged methods, including passthrough owner functions to the factory itself. The admin can also set the payout amount and the reward receiver address. |
| `DelegationSurrogate.sol` | The `DelegationSurrogate` contract is a simple and effective way to delegate voting power while maintaining token custody.                                                                                                         | The contract delegates its voting weight to a designated delegatee, holds governance tokens on behalf of users and approves the deployer to transfer all tokens, allowing for easy token recovery.                                                                                                                                                                                                                                                                     | The `DelegationSurrogate` contract is important because it allows users to delegate their voting power without losing custody of their tokens and it managed by the deployer, who can recover the tokens at any time.                                                                                                                                                               |

## Systemic Risks, Centralization Risks, Technical Risks & Integration Risks

1.  **UniStaker.sol**

    - **Systemic Risks**

      1.  **Reward Distribution Reliability:** The contract's ability to distribute rewards relies on the consistent flow of protocol fees and proper functioning of the reward distribution mechanism. Any disruptions to these processes could impact the reliability of reward distribution, potentially leading to dissatisfaction among stakeholders.

      2.  **Protocol Upgrade Risks:** If the Uniswap V3 protocol undergoes significant upgrades or changes that affect fee collection or reward distribution mechanisms, the UniStaker contract may need to be updated accordingly. Failure to adapt to protocol changes in a timely manner could disrupt reward distribution and stakeholder trust.

      3.  **Upgradability** risk if governance passes undesirable changes to core mechanics

    - **Centralization Risks**

      1.  **Admin Control:** The admin has the power to enable/disable reward notifiers, which could be used to manipulate reward distribution or exclude certain accounts from receiving rewards.

      2.  **Single Point of Failure:** The contract relies on a single admin for critical operations, which could lead to delays or disruptions if the admin becomes unavailable or unresponsive.

      3.  **Surrogate Contracts:** The mapping of governance delegates to surrogate contracts introduces centralization risks, as the control over delegation mechanisms and associated staked tokens is centralized within these surrogate contracts. If these contracts are compromised or manipulated, it could impact the governance process and stakeholder interests.

    - **Technical Risks**

      1.  **Reward Calculation Errors:** The contract's reward calculation logic could contain errors or vulnerabilities that could result in incorrect or unfair reward distribution.

      2.  **External Contract Dependencies:** The contract relies on external contracts (e.g., ERC20, DelegationSurrogate) for key functionality, and any issues or changes in those contracts could affect the operation of this contract.

      3.  **Smart Contract Complexity:** The contract's code is complex and involves multiple interactions between different components, which increases the risk of bugs or vulnerabilities.

    - **Integration Risks**

      1.  **Interoperability Issues:** The contract interacts with multiple external contracts, and any changes or updates to those contracts could break the integration and affect the functionality of this contract.

      2.  **Dependency on External Services:** The contract relies on external services (e.g., Ethereum network) for operation, and any outages or disruptions could impact the availability or reliability of the contract.

2.  **V3FactoryOwner.sol**

    - **Systemic Risks**

      1.  Governance manipulation if timelock is compromised

      2.  Protocol risks if underlying Uniswap contracts are upgraded unsafely

    - **Centralization Risks**

      1.  **Admin privileges:** The admin has the exclusive right to call privileged methods on the v3 factory and deployed pools. This gives them a significant amount of power and could potentially lead to abuse or misuse of the contract.

      2.  **Payout token and amount:** The payout token and amount are set by the admin. This could potentially be used to manipulate the fees collected or to reward specific addresses.

    - **Technical Risks**

      1.  **Payout token and amount:** The payout token and amount are not validated before being used to claim fees. This could potentially lead to errors or unexpected behavior.

      2.  **Insufficient fees collected:** The contract does not guarantee that the caller will receive the requested amount of fees. If the pool does not have enough fees accrued, the caller could lose their payout.

    - **Integration Risks**

      1.**Uniswap v3 factory:** The contract relies on the Uniswap v3 factory to function properly. If the factory is compromised or becomes unavailable, it could disrupt the contract's ability to collect fees.

      2.  **Reward receiver:** The contract relies on the reward receiver to be notified when fees are claimed. If the reward receiver is compromised or becomes unavailable, it could prevent the caller from receiving their payout.

## Issues surfaced from Attack Ideas in [README](https://github.com/code-423n4/2024-02-uniswap-foundation?tab=readme-ov-file#attack-ideas-where-to-look-for-bugs)

- Overflows

- Rounding not favoring the protocol

- Rewards shortfalls

- Boundary conditions on reward notification

- Unexpected conditions due to unexpected interactions with/configuration of a Uniswap V3 Pool

- Governance based economic/incentive attacks

- Unexpected issues caused by (good faith) reward notifier contracts being added or removed in the future

- Issues caused by changes in network conditions due to network upgrade (the contracts are intended to be immutable)

- Attacks or bugs caused by account abstraction or smart contract wallet usage


### Time spent:
20 hours