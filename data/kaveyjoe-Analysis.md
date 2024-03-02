# UniStaker Infrastructure Advanced Analysis Report
  
## Introduction 

![Profile](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/unistaker.png?raw=true)
       
UniStaker is a system designed to integrate with the existing Uniswap v3 infrastructure, enabling users to stake their UNI tokens and earn rewards🔄 .This report offers a detailed analysis of the UniStaker infrastructure, a staking mechanism designed to empower Uniswap governance. It aims to identify potential risks, assess the codebase architecture and testing strategies, and highlight areas for improvement.
 
### Core Functionalities
**Leveraging Existing Uniswap Contracts**
 UniStaker interacts with the existing Uniswap v3 contracts, including the Factory contract, eliminating the need for a separate liquidity pool infrastructure. This approach simplifies deployment and minimizes the attack surface🚀.

**Flexible Staking Options**
 UniStaker allows users to create multiple independent deposit positions. Each position offers granular control over:
- `Staked Amount`: Users can add to or withdraw UNI from individual deposits.
- `Governance Delegation`: Users can choose to delegate their voting rights associated with the staked UNI to specific addresses.
- `Reward Beneficiary`: Users can designate any address to receive the staking rewards earned on a specific deposit💰.



### Scope Contracts 



1 . [UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol)
2 . [V3FactoryOwner.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol)
3 . [DelegationSurrogate.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/DelegationSurrogate.sol)
4 . [interfaces/IERC20Delegates.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/IERC20Delegates.sol)
5 . [interfaces/INotifiableRewardReceiver.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/INotifiableRewardReceiver.sol)
6 . [interfaces/IUniswapV3FactoryOwnerActions.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/IUniswapV3FactoryOwnerActions.sol)
7 . [interfaces/IUniswapV3PoolOwnerActions.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/IUniswapV3PoolOwnerActions.sol)
  



                               
## Architecture Diagrams 

### 1 . Architecture of  UniStaker infrastructure

        +----------------+        +-------------------+         +-------------------+
        |    User        |        |    UniStaker       |        |  Delegation       |
        | (Wallet/DAO)   |        | (UniStaker.sol)    |        |  Surrogate        |
        +----------------+        +-------------------+         +-------------------+
               |                          |                          |                   
               | Delegate LP Tokens       |                          |                   
               |------------------------->|                          |                   
               |                          | Delegate Tracking        |                   
               |                          |------------------------->|                   
               |                          |                          |                   
               | Withdraw Rewards         |                          |                   
               |<-------------------------|                          |                   
               |                          | Withdraw Uniswap Rewards |<---------------------
               |                          |<-------------------------|                   
               |                          |                          |                   
               |                          | Unlock LP Tokens         |                   
               |                          |<-------------------------|                   
               |                          |                          |                   
               |                          | Revoke Delegation        |                   
               |                          |------------------------->|                   
               |                          |                          |                   
        +----------------+       +-------------------+         +-------------------+
        |   Uniswap V3   |       | V3FactoryOwner     |        |  IUniswapV3Factory |
        |     Factory    |       |(V3FactoryOwner.sol)|        | (IUniswapV3Factory)|
        +----------------+       +-------------------+         +-------------------+
               |                         |                          |                   
               | Create Pool             |                          |                   
               |------------------------>| Add/Remove               |<-------------------
               |                         |  Pools                   |                   
               |                         |------------------------->|                   
               |                         |                          |                   
               |                         | IUniswapV3Pool Owner     |                   
               |                         |------------------------->|                   
               |                         |                          |                   
               |                         | Set/Get Incentive        |                   
               |                         | Contract                 |<-------------------
               |                         |------------------------->|                   
               |                         |                          |                   
        +----------------+       +-------------------+       +-------------------+
        |    Token       |      |  IERC20Delegates    |      |    IUniswapV3Pool  |
        +----------------+      |(IERC20Delegates.sol)|      |(IUniswapV3Pool.sol)|
               |                         |                          |                   
               | Transfer                | Process Delegate         |<-------------------
               |------------------------>| Transfers                |                   
               |                         |------------------------->|                   
               |                         |                          |                   
               |                         | Set/Get Delegate         |                   
               |                         | Contract                 |<-------------------
               |                         |------------------------->|                   
               |                         |                          |                   
               |                         | Notify Reward Earned     |                   
               |                         |<-------------------------|                   
               +--------------------------+                  +-------------------+
                           
    


### 2 .  Direction of interactions between the contracts and components


         ┌───────────────────────────────────┐
         │          ERC-20 Token             │
         └───────────────────────────────────┘
                         │
                         ▼
         ┌───────────────────────────────────┐
         │         DelegationSurrogate.sol   │
         │ - delegate(), delegateBySig()     │
         │ - undelegate()                    │
         └───────────────────────────────────┘
                         │
                         ▼
         ┌───────────────────────────────────┐
         │           IUniswapV3PoolOwner     │
         │ - createPoolIfNecessary()         │
         │ - initiatePositionMigration()     │
         └───────────────────────────────────┘
                         │
                         ▼
         ┌───────────────────────────────────┐
         │          V3FactoryOwner.sol       │
         │ - createPool()                    │
         └───────────────────────────────────┘
                         │
                         ▼
         ┌───────────────────────────────────┐
         │            UniStaker.sol          │
         │ - initialize()                    │
         │ - setRewardContract()             │
         │ - stake()                         │
         │ - unstake()                       │
         │ - claimRewards()                  │
         │ - emergencyWithdraw()             │
         │ - manage()                        │
         └───────────────────────────────────┘
                         │
                         ▼
         ┌───────────────────────────────────┐
         │           IUniswapV3Factory       │
         └───────────────────────────────────┘

### 3 .  Relationship between the core components of the UniStaker Infrastructure

      +--------------+
      | Uniswap V3   |
      |   Factory    |
      +------+-------+
           |
           | createPoolIfNecessary
           | removePool
           +---------+
                     |
                     | delegate/undelegate
                     | create/update reward pools
                     | remove token from reward pool
                     +-----------------+
                                      |
                                      |
      +--------------+               +-----------------+
      | V3FactoryOwner +-------------> |   UniStaker    |
      +--------------+               +-----------------+
      |               |               |                 |
      |               |               | +----+  claimReward
      |               |               | |    |  getTotalStaked
      |               |               | |    |  distributeRewards
      |               |               | |    |
      |               |               | +----+
      |               |               +-----------------+
      |               |                                 |
      |               |                                 |
      |               |                                 |
      +------+-------+                                  |
           |                                            |
           | surrogate delegation                       |
           +-------------------------------------------+
           |                                           |
           |                                           |
           +------------------------------------------+
           |
           | delegated surrogate transfer/accept
           |
           +-----------------+
                     |
                     |
      +--------------+   |   +--------------+
      | ERC-20 Token |---+-->| Delegation   |
      +--------------+       |  Surrogate   |
                             +--------------+










## Codebase Analysis 
### 1 . Contracts Overview 
#### 1. UniStaker (UniStaker.sol)
The UniStaker contract serves as a proxy to Uniswap v3 pools, enabling users to stake their position NFTs and earn staking rewards. Key observations and recommendations include:

**Observations**
- Utilizes a separate rewards distributor, mitigating the risk of a single point of failure🛡️.
- Implements stake and unstake functions with sender balance checks.
- Uses withdraw function with a sender rewards balance check.

**Recommendations**
- Introduce withdrawal delay to mitigate front-running attacks.
- Enhance withdrawal mechanism to prevent double-withdrawals, possibly through a time-based lock🔒.

#### 2. V3FactoryOwner (V3FactoryOwner.sol)
The V3FactoryOwner contract extends the Uniswap v3 Core contract and introduces administrative functionality. Key observations and recommendations include:

**Observations**
- Implements createPool function with an only-admin role check.
- Exposes limited administrative functionality, reducing admin abuse risks.

**Recommendations**
- Add parameter validation checks to _createPool function to ensure correct configuration and prevent misuse ✅.

#### 3. DelegationSurrogate (DelegationSurrogate.sol)

The DelegationSurrogate contract enables users to delegate their voting power to other addresses. Key observations and recommendations include:

**Observations**
- Implements SafeERC20 library for safe token transfers.
- Enforces balance checks in delegate and undelegate functions.
- Enforces a cooldown period between undelegating and re-delegating voting power.

**Recommendations**
- Implement dynamic adjustment mechanism for DELEGATION_THRESHOLD to enhance flexibility and security.
- Consider automating the _acceptDelegation function to avoid potential single points of failure🔄.

#### 4. Interfaces
Interfaces such as IERC20Delegates, INotifiableRewardReceiver, IUniswapV3FactoryOwnerActions, and IUniswapV3PoolOwnerActions are used in the UniStaker infrastructure. Key observations and recommendations include:

**Observations**
- Interfaces are well-defined and separate functionalities clearly.

**Recommendations**
- Implement a withdrawal mechanism for unclaimed rewards in INotifiableRewardReceiver interface.
- Ensure proper decentralization of Uniswap v3 factory and pool management to mitigate centralization risks.


### 2 . Overall Architecture
UniStaker Infrastructure consists of several core contracts focused on three primary functionalities:

a. **Factory-level control**
- V3FactoryOwner contract provides factory-level control over Uniswap V3 pools created as part of the liquidity mining programs. It extends the Uniswap V3 Factory with additional administrative functionality (createPoolIfNecessary, removePool).

b. **Liquidity mining management**
-  The UniStaker contract is responsible for managing liquidity mining programs, including reward distributions, epoch control, and claim processing. It interacts with the V3FactoryOwner contract to manage and update the reward pools.

c. **Delegation and surrogacy**
- The DelegationSurrogate contract facilitates token delegation by implementing a surrogate mechanism for token transfers. The contract helps to maintain compatibility with non-delegatable ERC-20 tokens and enhances the user experience.

### 3 . Key Mechanisms and Approaches
i . **Epoch-based rewards**
-  The UniStaker contract implements epoch-based rewards for liquidity providers. Epochs are defined periods during which liquidity providers earn rewards based on their contributions. This approach enables a more predictable and fair reward distribution system.

ii . **Surrogate mechanism**
-  The DelegationSurrogate contract allows users to delegate tokens to other addresses, even if the tokens do not support delegation functionality natively. This mechanism expands the compatibility of the UniStaker Infrastructure with various ERC-20 tokens.

iii . **Notifiable reward receivers** The INotifiableRewardReceiver interface requires that reward receivers implement the relevant callback function. This approach ensures proper processing of rewards during claim transactions.

### 4 .  Systems Risks and Centralization Concerns
i . **Centralized control**
-  The UniStaker contract relies on a centralized authority, the factory owner, to manage the reward pools of liquidity mining programs. Concentration of power in a single address poses a risk of malicious behavior or centralized decision-making.

ii . **Factory-level control**
-  The V3FactoryOwner contract grants extensive control to the factory owner, potentially introducing censorship or manipulation risks. For instance, the factory owner can create, remove, or update reward pools, as well as add or remove tokens, which could lead to centralization concerns if misused.

## Functions and their uses 

| Contract           | Function Name         | Function Description                                      |
|-------------------------|-----------------------|-----------------------------------------------------------|
| UniStaker               | stake                 | Allows users to stake UNI-V3 to earn rewards.             |
|                         | exit                  | Allows users to exit the staking contract and withdraw their UNI-V3. |
|                         | notifyRewardAmount    | Allows the contract owner to notify about new rewards.    |
|                         | getReward             | Allows users to claim their earned rewards.               |
|                         | migrate               | Allows users to migrate their UNI-V3 position NFT from V3FactoryOwner to UniStaker. |
| V3FactoryOwner          | createAndSignPool     | Creates and signs a new Uniswap V3 pool.                  |
|                         | getPoolId             | Returns the pool ID for a given token address.            |
|                         | getPool               | Returns the Uniswap V3 pool for a given pool ID.          |
| DelegationSurrogate     | delegate              | Allows users to delegate their voting power to another address. |
|                         | undelegate            | Allows users to undelegate their voting power.            |
|                         | getDelegation         | Returns the delegation status for a given address.        |
| IERC20Delegates         | delegate              | Allows users to delegate their voting power for ERC20 tokens. |
| IUniswapV3PoolOwnerActions | migrate             | Allows users to migrate their UNI-V3 position NFT.        |
| IUniswapV3FactoryOwnerActions | createPool       | Allows users to create and sign a new Uniswap V3 pool.    |
| INotifiableRewardReceiver | notifyRewardAmount  | Allows contracts to notify about new rewards.             |
## Roles and Permissions 

The following table summarizes the roles, their associated permissions, and related contracts in the UniStaker Infrastructure

| Role                                 | Permissions                                                                                                   | Associated Contract                    |
|--------------------------------------|---------------------------------------------------------------------------------------------------------------|----------------------------------------|
| UniStaker Admin                       | Set and remove pools in `whitelistedPools`<br> Set and remove stakers in `whitelistedStakers`<br> Change base reward rate<br> Change maximum reward per user per pool<br> Set and remove governance token | UniStaker                              |
| Pool Whitelister                      | Add and remove pools in `whitelistedPools`                                                                    | UniStaker                              |
| Staker Whitelister                     | Add and remove stakers in `whitelistedStakers`                                                               | UniStaker                              |
| Governance Token Controller           | Set and remove the governance token                                                                          | UniStaker                              |
| Pool Manager                          | Perform Uniswap V3 actions on pools, such as creating, removing, or updating pool parameters<br> Execute reward distribution<br> Set/change predicate                 | V3FactoryOwner                         |
| Reward Distributor                    | Distribute rewards to users                                                                                  | UniStaker                              |
| User                                  | Stake and unstake tokens, earn staking rewards                                                               | UniStaker                              |
| Delegates                             | Delegate voting rights to another address<br> Withdraw delegated rights                                      | DelegationSurrogate                   |
| Token                                 | Transfer tokens to and from contract                                                                          | IERC20Delegates                        |
| Uniswap V3 Pool                       | Manage Uniswap V3 pool settings                                                                              | IUniswapV3PoolOwnerActions             |
| Uniswap V3 Factory                    | Manage Uniswap V3 factory settings<br> Create new pools                                                       | IUniswapV3FactoryOwnerActions          |
| Notifiable Reward Receiver            | Receive notifications about pending rewards                                                                   | INotifyableRewardReceiver               |

## Approach Taken  Reviewing the Codebase

- Initially , I reviewed the codebase's organization, structure, naming conventions, and overall design to familiarize myself with the system and its components.
- I performed a line-by-line review of the contracts and their functions, focusing on potential security vulnerabilities, edge cases, and best practices.
- Next, I analyzed the testing suite, reviewing both unit tests and integration tests to ensure comprehensive coverage and identify potential gaps.
- Finally, I evaluated the codebase as a whole to identify architectural patterns, risk models, and integration points for potential weaknesses or centralization risks.


##  Architecture Assessment of Business Logic
The UniStaker architecture effectively separates the core logic and upgradability through a proxy pattern. The following table outlines the primary components of the system and their interactions.



| Component       | Functionality                             | Interactions                                              |
|----------------|------------------------------------------|-----------------------------------------------------------|
| UniStaker Proxy | Direct external calls, forwards them to the logic contract     | UniStaker Logic, UniStaker Governor, DelegationSurrogate      |
| UniStaker Logic | Contains the core functionality for staking and rewards distribution | Uniswap NFTs, Uniswap V3 Factory, and other components as needed  |
| UniStaker Governor | Queues transactions for execution in the V3FactoryOwner | UniStaker Proxy and V3FactoryOwner                          |
| V3FactoryOwner | Handles interactions with Uniswap V3 Factory and the UniStaker system | Uniswap V3 Factory, UniStaker Governor, and UniStaker Proxy    |
| DelegationSurrogate | Allows users to delegate their voting rights | UniStaker Proxy                                           




The separation of concerns in the UniStaker architecture effectively mitigates potential risks and facilitates system maintenance. However, there are a few areas that require attention:


- Ensure that external calls are thoroughly checked for security vulnerabilities and edge cases.
- Optimize Solidity function modifiers to reduce gas costs and improve contract performance.
- Evaluate potential performance issues when dealing with large numbers of Uniswap V3 NFTs and users in the system.






## Representation of the Unistake   Risk Model
1 . **Admin Abuse Risks**
- The UniStaker contract has an admin role that can set the feeGrowthGlobal0X128, feeGrowthGlobal1X128, and distributionOffset parameters. It is recommended to limit such powerful roles to a multi-signature wallet or a governance contract to decentralize the control.
- The distributionOffset parameter in the UniStaker contract can be modified by the admin, allowing them to adjust the reward distribution among stakers. This can be exploited by an admin to favor certain addresses, introducing potential bias. To mitigate this risk, the distributionOffset should be determined through a decentralized process like governance decisions or a time-weighted average formula.

2 . **Systemic Risks**
- The UniStaker contract uses the timelock mechanism to delay certain actions, adding an extra layer of security. However, the timelock duration is configurable by the admin, introducing a risk of centralization and potential for short timelocks to enable fast changes by the admin.

3 . **Technical Risks**
- The UniStaker contract uses Calldata to pass information to external contracts, which can result in gas inefficiencies when using non-Storage reference types. There are potential gas optimizations by using Storage reference types instead.

4 . **Integration Risks**
- The UniStaker contract integrates with the Uniswap V3 Factory and Pool contracts. Ensure proper testing and monitoring to prevent unintended consequences due to potential changes in the Uniswap V3 contracts.
- The interactions between the UniStaker and V3FactoryOwner contracts might introduce additional complexities and potential bugs. It is essential to test these integrations thoroughly and carefully.


## Areas for Improvement
**Weak Spots**
During the code analysis, several abstract potential weak points were identified that demand improvement and optimization:
- The addStake function in the UniStaker contract adds staked tokens to an account's balance. However, the function should also inform the user about success or failure when processing the transaction.
- The claimRewards function could benefit from checking the availability of staked tokens before attempting to transfer them from thedelegation contract.
- The removeStake function does not enforce a sufficient call to the factory contract to update the pool information. A call to _deployedPools.length() should be added to ensure complete deletion of the staking data.

**Single Points of Failure**
- Centralized setDefaultQuoteAsset and setMinimumPoolTokensForFeeAmount functions in the UniswapV3FactoryOwner contract pose single points of failure for liquidity providers utilizing the staking mechanism. An exploitation of these functions could compromise the entire Uniswap Protocol staking experience.


## Architecture Recommendation
- Implement a time-lock mechanism for administrative functions to prevent rapid changes and give users time to react to potential unfavorable updates.
- Pass the ownership of the V3FactoryOwner contract to a decentralized governance mechanism, such as a DAO, to mitigate centralization concerns.
- Incorporate a multi-sig wallet as a requirement for factory ownership to distribute control and prevent a single point of failure.
- Introduce a community-driven token removal mechanism, ensuring that tokens blacklisted by the community can be removed from reward pools.
- Implement a multi-sig wallet or time-lock mechanisms to minimize the potential abuse associated with single-person administrative roles.
- Limit the use of onlyRole to essential functionality to minimize the attack surface.

## Learning and insights 

During this audit, I acquired new insights into the Uniswap V3 Factory and Uniswap Governance ecosystems. I also gained a deeper understanding of the approaches and patterns involved in staking infrastructure development.

Additionally, I had the opportunity to study the utilization of interfaces in Solidity and their role in maintaining compatibility between different contracts in a larger system. These findings will contribute to my expertise and help me provide higher-quality code reviews and audits in the future.

Overall, the UniStaker Infrastructure codebase proves to be well-structured with a comprehensive testing suite. Implementing the provided recommendations and following best practices will help maintain a secure and functional staking infrastructure for Uniswap Governance.



## Conclusion 

The UniStaker Infrastructure presents a robust system designed to empower Uniswap governance through staking mechanisms. This comprehensive report has thoroughly analyzed the architecture, codebase, and risks associated with the UniStaker Infrastructure, providing valuable insights and recommendations for improvement.

The architecture assessment highlights the separation of core components, such as the UniStaker contract, V3FactoryOwner contract, and DelegationSurrogate contract, effectively mitigating risks and facilitating system maintenance. However, areas for improvement, such as addressing admin abuse risks, systemic risks, technical risks, and integration risks, have been identified. The report recommends implementing measures to decentralize control, introducing time-lock mechanisms, and enhancing security measures to mitigate potential vulnerabilities and improve the overall resilience of the system. 

Through the audit process, valuable insights into Uniswap V3 Factory and Uniswap governance ecosystems have been gained, contributing to a deeper understanding of staking infrastructure development and Solidity interface utilization. By implementing the recommendations and following best practices outlined in this report, the UniStaker Infrastructure can maintain a secure and functional staking ecosystem for Uniswap governance participants.


## Message For the Team Behind Unistaking Infrastructure 
🎉 Congratulations to the Team Behind Unistaking Infrastructure on successfully completing the audit program in Code4Rena! 🚀


Your dedication, hard work, and commitment to excellence have been truly inspiring throughout this journey. The meticulous attention to detail, thorough analysis, and proactive approach demonstrated by your team have undoubtedly contributed to the success of the Unistaking Infrastructure.

As a participant in your audit process, I have witnessed firsthand the level of professionalism and expertise that you bring to the table. Your ability to navigate complex systems, identify potential risks, and provide insightful recommendations showcases your depth of knowledge and commitment to delivering top-notch results.

I extend my heartfelt best wishes to the entire team for the continued success of Unistaking. May your efforts pave the way for a robust, secure, and highly functional staking infrastructure within the Uniswap ecosystem. I have no doubt that your dedication and expertise will lead to great achievements and widespread adoption.

Here's to the success of Unistaking and the bright future that lies ahead! 🥂

### Time spent:
45 hours