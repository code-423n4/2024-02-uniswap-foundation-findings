### Project Overview
UniStaker is a mechanism designed to facilitate the collection and distribution of protocol fees generated by the Uniswap V3 pools through UNI token staking. This setup allows Uniswap Governance to enable and manage these protocol fees effectively. By integrating contracts from this repository, Uniswap Governance could maintain the authority to set protocol fees for Uniswap V3 Pools without directly handling the fee assets. Instead, the fees generated are distributed in a trustless manner to UNI holders who opt to stake their tokens. The unique aspect of this system is that rewards for stakers are not in the form of fee tokens directly but in a predefined token established at the deployment of these contracts. The accumulated fees from each pool are periodically auctioned to entities willing to exchange them for the specified token, thereby facilitating the distribution of rewards to stakers.

The operational framework of UniStaker is built around two core contracts: V3FactoryOwner.sol and UniStaker.sol. The V3FactoryOwner contract functions as the new owner of the Uniswap V3 Factory, allowing governance to transfer factory ownership to this contract while retaining control over fee settings through a governance mechanism. On the other hand, the UniStaker contract is responsible for the distribution of staking rewards, employing a mechanism that allows rewards to drip over a fixed period, similar to the Synthetix StakingRewards.sol model. This contract enables UNI stakers to maintain their governance rights, designate beneficiaries for their rewards, and manage their stakes on a per-deposit basis, introducing efficiencies in terms of precision, gas usage, and code clarity. Additionally, UniStaker is designed to accommodate rewards from various sources, with the potential for future expansion beyond Uniswap V3 protocol fees, under the administration of Uniswap Governance.

File Name | Description
-- | --
UniStaker.sol | The code defines a smart contract, UniStaker, responsible for managing the distribution of staking rewards in the form of ERC20 tokens to participants who deposit a specific governance token. It allows for flexible management of staking positions, enabling users to delegate voting power, specify reward beneficiaries, and alter these designations while participating in a reward distribution mechanism inspired by Synthetix's model.
V3FactoryOwner.sol | The code defines V3FactoryOwner, a contract serving as the owner of the Uniswap v3 factory, allowing an admin (expected to be Uniswap Governance) to manage fee settings on pools and the factory itself. It enables a public function for collecting protocol fees from pools in exchange for a specified token, aiming to create a competitive market for fee collection.
DelegationSurrogate.sol | DelegationSurrogate is a streamlined contract designed to hold governance tokens on behalf of users while delegating their voting power to a specified delegatee. This approach enables individual token holders to maintain their governance rights by using a separate surrogate for each delegatee, even when their tokens are pooled together under a single contract.

### Architecture Digram
The architecture diagram below illustrates the interaction between various components of the system, focusing on governance token delegation and staking rewards distribution. This system allows governance token holders to stake their tokens, delegate their voting power, and earn rewards, all while maintaining their governance rights.


![Screenshot (32)](https://gist.github.com/assets/162044426/88873df9-95b8-4a37-af3a-a4e52c354194)

### Architecture Overview

1. **Token Holders** represent individuals or entities that own governance tokens. They have the option to stake these tokens in a staking contract to earn rewards and participate in governance by delegating their voting power.

2. **Staking Contract** is the central hub where token holders stake their governance tokens to earn rewards. It interacts with other system components to manage staked tokens and distribute rewards.

3. **DelegationSurrogate** is deployed by the staking contract for each delegatee. Its purpose is to hold staked governance tokens and delegate voting power to a specified delegatee, allowing token holders to maintain their governance rights even when their tokens are pooled together.

4. **Rewards Distribution Mechanism** is responsible for distributing rewards to token holders based on the amount of tokens they have staked and other criteria defined by the system.

5. **Delegatee** is an individual or entity to which the DelegationSurrogate delegates voting power. This allows them to vote in governance proposals on behalf of the token holders who have staked their tokens.

6. **Uniswap V3 Factory** is part of the broader ecosystem, where the staking contract might interact with Uniswap V3 to manage liquidity pools, set fees, or perform other actions related to the governance of Uniswap V3 pools.

7. **Uniswap V3 Pools** are liquidity pools managed by the Uniswap V3 Factory, which can be influenced by governance decisions made by the staking contract, delegatees, or directly by token holders.

### Sequence Diagram

This architecture enables a decentralized and democratic governance system where token holders can earn rewards while participating in the governance of the protocol or ecosystem they are invested in. It balances the need for efficient governance token management with the desire to empower individual token holders.
Below is a sequence diagram illustrating the interactions within the system, focusing on the process of staking tokens, delegating voting power, and distributing rewards.



[![Screenshot-30.png](https://i.postimg.cc/9F3THDyy/Screenshot-30.png)](https://postimg.cc/K4N4nvqY)



### Sequence Diagram Overview:

1. **Token Holder** initiates the process by staking their governance tokens in the **Staking Contract**.

2. For each stake, the **Staking Contract** either deploys a new **Delegation Surrogate** or selects an existing one, based on the designated **Delegatee**.

3. The **Delegation Surrogate** then delegates the voting power of the staked tokens to the specified **Delegatee**, ensuring that token holders retain their governance rights.

4. Parallelly or subsequently, the **Staking Contract** communicates with the **Rewards Distribution Mechanism** to calculate the rewards for each token holder based on the staked tokens and other criteria.

5. The **Rewards Distribution Mechanism** distributes the calculated rewards back to the **Token Holder**.

6. Optionally, the **Token Holder** might directly delegate their voting power to a **Delegatee**, bypassing the staking mechanism for governance participation.

7. Optionally, the **Staking Contract** might interact with the **Uniswap V3 Factory** for liquidity pool management or other governance actions. The **Uniswap V3 Factory** updates the **Staking Contract** with any changes to pool status or information.

This sequence outlines the flow of actions from staking tokens to receiving rewards while ensuring governance participation through delegation.

### Overview of Functions in the UniStaker Smart Contract

[![Screenshot-38.png](https://i.postimg.cc/cJ9Vjyg5/Screenshot-38.png)](https://postimg.cc/LqPCYWVz)
#### Admin and Reward Notifier Management

- **setAdmin**: Updates the admin of the contract. Only the current admin can perform this action.
- **setRewardNotifier**: Enables or disables a reward notifier address, allowing or disallowing it from notifying the contract about new rewards. This action is also restricted to the admin.

#### Staking Operations

- **stake**: Allows a user to stake tokens into a new deposit, automatically delegating voting power and setting themselves as the reward beneficiary.
- **stakeMore**: Enables adding more tokens to an existing stake, maintaining the current delegatee and beneficiary settings.
- **permitAndStake**: Similar to `stake`, but includes an ERC-20 permit for token approval, reducing transaction steps.
- **stakeOnBehalf**: Allows staking on behalf of another user, with their permission, enabling the staker to specify delegatee and beneficiary.
- **stakeMoreOnBehalf**: Adds more tokens to an existing deposit on behalf of another user, with their permission.

#### Delegation and Beneficiary Management

- **alterDelegatee**: Changes the delegatee for a specific deposit, allowing the stake's voting power to be redirected.
- **alterDelegateeOnBehalf**: Similar to `alterDelegatee`, but performed on behalf of the deposit owner with their permission.
- **alterBeneficiary**: Changes the beneficiary who earns rewards from a specific deposit.
- **alterBeneficiaryOnBehalf**: Allows changing the beneficiary on behalf of the deposit owner, with their permission.

#### Withdrawal and Reward Claiming

- **withdraw**: Withdraws staked tokens from a deposit, reducing the stake and potentially affecting reward earnings.
- **withdrawOnBehalf**: Performs a withdrawal on behalf of the deposit owner, with their permission.
- **claimReward**: Allows a beneficiary to claim their earned rewards.
- **claimRewardOnBehalf**: Claims rewards on behalf of a beneficiary, with their permission.

#### Reward Notification

- **notifyRewardAmount**: Called by authorized reward notifiers to inform the contract about new rewards being added. It adjusts the reward rate and duration accordingly.

#### Internal Helper Functions

- **_fetchOrDeploySurrogate**: Deploys or retrieves a Delegation Surrogate contract for a specified delegatee.
- **_stakeTokenSafeTransferFrom**: Safely transfers staked tokens from one address to another.
- **_useDepositId**: Generates a unique identifier for a new deposit.
- **_stake**: Core logic for staking operations, handling token transfers, and setting deposit parameters.
- **_stakeMore**: Adds tokens to an existing stake, updating the total staked amount and rewards.
- **_alterDelegatee**: Updates the delegatee for a deposit, managing the delegation of voting power.
- **_alterBeneficiary**: Changes the beneficiary for a deposit, affecting who earns the rewards.
- **_withdraw**: Handles the withdrawal of staked tokens from a deposit.
- **_claimReward**: Processes reward claims, transferring earned rewards to beneficiaries.
- **_checkpointGlobalReward**: Updates the global reward rate and distribution end time based on new rewards.
- **_checkpointReward**: Updates the reward calculation for a specific beneficiary.
- **_setAdmin**: Sets the admin address internally.
- **_revertIfNotAdmin**: Checks if the caller is the admin and reverts if not.
- **_revertIfNotDepositOwner**: Ensures the caller owns the deposit they are trying to modify.
- **_revertIfAddressZero**: Checks for zero addresses in critical parameters.
- **_revertIfSignatureIsNotValidNow**: Validates EIP-712 signatures for actions performed on behalf of others.

This contract facilitates complex staking, delegation, and rewards management operations, integrating with ERC-20 tokens and leveraging DeFi conventions for governance and reward distribution.
## UniStaker Smart Contract Functionalities Overview

### Main Functionalities

- **Stake Tokens**: Allows users to deposit governance tokens into the contract to participate in staking. Users can choose to delegate their voting power to a specific delegatee and designate a beneficiary for their rewards.

- **Withdraw Tokens**: Permits stakers to withdraw their deposited tokens from the contract. This action ceases their participation in reward distribution.

- **Claim Rewards**: Enables beneficiaries to claim their accrued rewards. The rewards are calculated based on the proportion of the user's stake relative to the total staked amount and the duration for which the tokens were staked.

### Delegation and Beneficiary Management

- **Delegate Voting Power**: Through the creation or selection of a Delegation Surrogate, stakers can delegate the voting power of their staked tokens to a chosen delegatee, allowing them to participate in governance decisions.

- **Alter Delegatee**: Stakers have the flexibility to change the delegatee to whom their voting power is assigned.

- **Designate or Change Beneficiary**: Stakers can specify or change the beneficiary address that will receive the staking rewards for their deposit.

### Reward Notification and Distribution

- **Notify Reward Amount**: Authorized entities can notify the contract about new rewards that have been added to the pool. This resets the reward distribution duration and updates the rate at which rewards are distributed.

### Administration and Permissions

- **Set Admin**: Designates a new admin for the contract. Only the current admin can perform this action.

- **Enable/Disable Reward Notifier**: Allows the admin to authorize or revoke the permission of addresses to notify the contract of new rewards.

### Utility and Maintenance

- **Fetch or Deploy Surrogate**: Internally handles the deployment of a new Delegation Surrogate contract or selects an existing one for a specific delegatee.

- **Safe Transfer Operations**: Ensures the safe transfer of tokens to and from the contract, adhering to the ERC20 standard's security practices.

- **Checkpoints and Accumulators**: Manages checkpoints for global reward distribution and individual beneficiary reward accumulation to ensure accurate and fair reward calculations.

### Security and Validation

- **Unauthorized Access Handling**: The contract includes several checks to prevent unauthorized actions, such as altering delegatees or beneficiaries, withdrawing tokens, and managing admin functions.

- **Signature Validation**: Supports operations on behalf of users through EIP-712 compliant signatures, ensuring that actions such as staking, withdrawing, and claiming rewards are securely authorized.

### Events

- **Emitted Events**: The contract emits events for significant actions, including deposits, withdrawals, changes in delegatees or beneficiaries, reward claims, and administrative changes. These events facilitate transparency and allow tracking of contract activities.

This smart contract introduces a comprehensive system for staking governance tokens, managing voting power delegation, and distributing rewards. It emphasizes user autonomy by allowing stakers to retain their governance rights through delegation and to designate beneficiaries for their rewards. The contract's security measures, including checks for unauthorized access and signature validation, ensure the integrity of its operations.
[![Screenshot-39.png](https://i.postimg.cc/NFkhdy1R/Screenshot-39.png)](https://postimg.cc/D40YZwpz)This sequence diagram outlines the interactions between a user, the UniStaker contract, ERC20 tokens, the Delegation Surrogate, and a reward notifier within the UniStaker system. It demonstrates the flow of stake deposits, stake modifications, withdrawals, reward claims, and reward notifications, emphasizing the contract's role in managing staked tokens, delegating voting power, and distributing rewards.
## V3FactoryOwner Smart Contract Functionalities Overview

### Contract Purpose and Overview
The V3FactoryOwner contract acts as the owner of the Uniswap V3 factory, enabling privileged control over factory and pool settings, including fee management. It also allows the collection of protocol fees from pools through a public function, facilitating an arbitrage opportunity by trading a designated token amount for pool fees.

[![Screenshot-41-uml-f2.png](https://i.postimg.cc/XNz9v2dS/Screenshot-41-uml-f2.png)](https://postimg.cc/pyKmGJsq)

### Key Functionalities

#### Administrative Control
- **Set Admin**: Assigns a new admin to the contract, transferring the ability to perform privileged actions. Only the current admin can execute this change.

- **Set Payout Amount**: Updates the amount of the payout token required for claiming fees from a pool. This function is reserved for the admin.

#### Fee Management
- **Enable Fee Amount**: Allows the admin to enable new fee tiers within the Uniswap V3 factory, specifying the fee amount and associated tick spacing.

- **Set Fee Protocol**: Grants the admin the ability to set protocol fee percentages for individual Uniswap V3 pools, adjusting the split between liquidity providers and the protocol.

#### Fee Claiming
- **Claim Fees**: Open to any caller, this function enables the collection of accumulated protocol fees from a specified Uniswap V3 pool. The caller must pay a predetermined amount of a designated payout token, which is then forwarded to a specified reward receiver.

### Constructor and Initialization
Upon deployment, the constructor initializes the contract by setting:
- The admin address, who will have exclusive rights to perform certain actions within the contract.
- The Uniswap V3 Factory contract instance, which this contract will own and manage.
- The payout token, used as payment for claiming pool fees.
- The initial payout amount, specifying how much of the payout token must be paid to claim pool fees.
- The reward receiver contract, which will be notified and receive the payout token when pool fees are claimed.

### Events
- **FeesClaimed**: Emitted when protocol fees are claimed from a pool, indicating the pool address, caller, recipient of the fees, and the amounts of token0 and token1 claimed.
- **AdminSet**: Signals the assignment of a new admin for the contract.
- **PayoutAmountSet**: Announces changes to the payout amount required for claiming pool fees.

### Error Handling
- **Unauthorized**: Indicates an attempt to perform an action reserved for the admin by an unauthorized address.
- **Invalid Address**: Used when an operation involves an address parameter that must not be the zero address, such as setting a new admin.
- **Invalid Payout Amount**: Triggered when attempting to set a zero payout amount, which is not allowed.
- **Insufficient Fees Collected**: Occurs if the actual fees collected from a pool are less than the amount requested by the caller.

### Security and Permission Checks
- **_revertIfNotAdmin**: A modifier-like internal function that ensures only the admin can perform certain actions, reinforcing the contract's security by restricting sensitive operations.



## Summary
The V3FactoryOwner contract is a critical component for managing Uniswap V3 factory settings, including fee structures and protocol fee collection. Its design focuses on providing administrative control over key parameters while enabling an innovative mechanism for protocol fee collection. Through its public claim fees function, it incentivizes external parties to participate in protocol fee collection, creating a competitive market dynamic. This sequence diagram shows the over all flow of the  functionality 

[![Screenshot-42-f2-seq.png](https://i.postimg.cc/zDskJJsY/Screenshot-42-f2-seq.png)](https://postimg.cc/0rCSnq5V)


## DelegationSurrogate Smart Contract Functionalities Detailed Overview

### Contract Purpose
The `DelegationSurrogate` contract is designed to facilitate governance participation for token holders whose tokens are pooled. It addresses the challenge of maintaining individual governance rights in a pooled environment by allowing the delegation of voting power from pooled tokens to a specified delegatee.

[![Screenshot-43-f3-uml.png](https://i.postimg.cc/MGncMBBP/Screenshot-43-f3-uml.png)](https://postimg.cc/8FVkxjtM)

### Key Functionalities

#### Constructor and Initial Setup
Upon deployment, the constructor performs crucial initializations to set up the contract's core functionality:

- **Token Delegation**: The constructor takes two arguments: a governance token (`_token`) and a delegatee (`_delegatee`). It immediately delegates the voting power of any governance tokens that will be held by this contract to the specified delegatee. This delegation is crucial for ensuring that the voting power associated with pooled tokens is not lost and can be exercised according to the preferences of the token holders.

- **Token Approval for Reclaiming**: In addition to delegating voting power, the constructor sets up an approval, allowing the deployer of the contract (most likely a staking pool or another contract pooling governance tokens) to reclaim the tokens without requiring further permissions. This is done by approving the maximum possible amount of tokens (`type(uint256).max`), ensuring that the deployer can manage the tokens as needed without additional transaction overhead.

### Operational Context

- **Maintaining Governance Rights**: The contract serves to ensure that token holders who contribute their tokens to a pool still have their preferences represented in governance decisions. By delegating the voting power of pooled tokens to chosen delegatees, it ensures that the governance influence of individual token holders is preserved.

- **Simplifying Token Management**: By approving the contract deployer to manage the tokens, the `DelegationSurrogate` simplifies the administrative aspect of token pooling. This setup allows for the efficient handling of tokens, enabling their movement without requiring individual approval transactions for each action.

### Security and Permissions

- **Immutable Delegation and Approval**: The actions taken by the constructor - delegating voting power and setting token approval - are performed at the time of contract deployment and cannot be altered afterward. This design choice simplifies the contract's operation and enhances its security by reducing the surface area for potential malicious actions or mistakes after deployment.

### Use Cases

- **Staking Pools and Governance**: The `DelegationSurrogate` is particularly useful in the context of staking pools or other mechanisms where governance tokens are pooled. It allows these structures to maintain the governance participation rights of their contributors, ensuring that the aggregation of tokens does not dilute individual governance influence.

- **Token Management Efficiency**: For contracts that manage pooled governance tokens on behalf of users, the `DelegationSurrogate` offers an efficient way to handle these tokens, particularly for operations like reallocating tokens back to users or moving them based on the pool's needs.

## Summary
The `DelegationSurrogate` contract is a streamlined solution designed to preserve the governance rights of token holders within pooled environments. Through its straightforward mechanism of delegating voting power and setting up token approvals at deployment, it ensures that governance participation remains effective and that token management remains efficient.

[![Screenshot-44-f3-seq.png](https://i.postimg.cc/KYP65qtR/Screenshot-44-f3-seq.png)](https://postimg.cc/rdpZV9nk)

### Centralization Risk

**Admin Control and Privileged Actions**: A significant centralization risk arises from the extensive control and privileged actions that an admin can perform, such as updating admin addresses, setting payout amounts, enabling or disabling reward notifiers, and other administrative functions. This centralized control could lead to potential misuse or abuse if the admin keys are compromised or if the admin acts maliciously.

**DelegationSurrogate and Voting Power**: The use of `DelegationSurrogate` to delegate voting power centralizes the governance influence in the hands of a few, potentially skewing governance decisions. Although it aims to empower token holders, the actual implementation could lead to centralization of voting power, especially if surrogate contracts are managed or influenced by a small group.

### Systematic Risk

**Dependency on External Contracts and Interfaces**: The system's reliance on external contracts and interfaces like `IUniswapV3PoolOwnerActions`, `IUniswapV3FactoryOwnerActions`, and `IERC20` introduces systematic risks. Changes or vulnerabilities in these external contracts could adversely affect the functionality and security of the system.

**Reward Distribution Mechanism**: The reward distribution mechanism, based on the notification of new rewards and the calculation of distributed rewards, introduces a risk of manipulation or errors in reward calculations. This could lead to loss of funds or unfair distribution of rewards, impacting the integrity of the staking and reward system.

### Architectural Risks

**Upgradability and Flexibility**: The contracts' architecture does not explicitly address upgradability or the ability to adapt to future requirements or fixes. This rigidity could lead to challenges in responding to discovered vulnerabilities, evolving governance models, or integrating with new protocols and standards.

**Inter-contract Communication**: The architecture involves multiple contracts interacting with each other, such as the delegation of voting power through `DelegationSurrogate` and the management of rewards in `UniStaker`. This interdependency increases the complexity and the risk of unintended consequences due to errors in communication or execution logic between contracts.

### Complexity Risks

**Contract Complexity and Interactions**: The contracts exhibit a high degree of complexity, particularly in the management of staking, delegation, and rewards distribution. This complexity increases the risk of bugs or vulnerabilities remaining undetected despite testing and audits.

**Understanding and Participation Barrier**: The complexity of contract interactions and the governance model may pose a barrier to understanding for potential users and participants. This could lead to lower participation in governance or staking, affecting the decentralization and security of the system.

In summary, while the system introduces innovative mechanisms for staking, delegation, and rewards distribution, it also presents centralization, systematic, architectural, and complexity risks that should be carefully managed and mitigated through rigorous security practices, audits, and potentially introducing more decentralized governance mechanisms over time.

### Conclusion
The UniStaker system presents an innovative approach to staking, voting delegation, and rewards distribution within the DeFi ecosystem. While it offers significant benefits in terms of governance participation and incentive mechanisms, it also carries risks related to centralization, system dependencies, architectural rigidity, and operational complexity. Addressing these concerns through continuous audits, enhancing decentralization, and simplifying user interactions will be crucial for

### Time spent:
28 hours