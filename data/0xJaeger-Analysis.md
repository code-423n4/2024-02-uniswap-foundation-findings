# UniStaker Infrastructure analysis

## Description Overview

The UniStaker Infrastructure system aims to enable Uniswap Governance to collect protocol fees on Uniswap V3. These fees will be distributed to UNI token holders who stake their tokens. A key feature of this protocol is that staked token holders retain their voting capability on governance proposals. This is achieved through a surrogate system that holds the tokens while delegating the voting power back to the user.

## System Overview

---

### Scope

Contracts

- V3FactoryOwner.sol: This contract will act as the owner of the Uniswap V3 Factory, managing deployed pool fees, determining payout amounts, and facilitating the 'claimFees()' race by MEV searchers.
- UniStaker.sol is directly inspired by the battle-tested Synthetix StakingRewards.sol implementation, which manages staking and reward distribution. It introduces unique functionalities, including the ability for depositors to either retain the voting power of their staked tokens or designate a beneficiary to receive the rewards of the staking.
- DelegationSurrogate.sol is a straightforward contract designed to hold staked tokens and delegate their voting power back to the owner.

### Protocol Roles

Roles in the provided protocol are generally associated with different addresses or entities that have specific permissions or responsibilities. Here are the roles and their associated functionalities in the contracts:

 V3FactoryOwner.sol contract has only one role

- Admin: 
The admin role, represented by the admin variable, is restricted to the contract owner. This role enables privileged actions on the V3 factory and deployed pools, such as enabling fee amounts on the factory, setting protocol fees on individual pools, and assigning a new admin.

The UniStaker.sol contract has 4 roles

- Admin: The admin role is represented by the admin variable and is restricted to the contract owner.  It is responsible for setting new reward notifiers through a privileged function. The admin can also set a new admin.
- Depositor: The depositor is the user who has staked UNI tokens and thus created a deposit. Only the deposit owner can add more funds to the same deposit, change the beneficiary of the rewards distributed, change the delegate of the voting power of the tokens staked in this deposit or withdraw some or all funds from this deposit.
- Beneficiary: Is the receiver of the deposit’s earningPower, it can be the depositor or any other address.
- Delegate: The delegate receives the delegated voting power from the surrogate.

The DelegationSurrogate.sol contract 

- Delegate: Represents the user with the delegated voting power of the tokens deposited within the surrogate.

 

## Approach Taken-in Evaluating The UniStaker Infrastructure

Accordingly, I analyzed and audited the subject in the following steps;

1. Protocol Overview 
    
    My primary focus was on thoroughly comprehending the codebase and providing recommendations to enhance its functionality. The overarching goal was to analyze how the contracts within the UniStaker Infrastructure work together to facilitate various operations, including receiving rewards for staked tokens, staking tokens while retaining voting power, distributing rewards, and managing the process of claiming fees from pools.
    
    I initiated my analysis by examining the smallest contract within the system, **DelegationSurrogate.sol**, to understand its mechanisms for safeguarding depositor tokens and delegating their voting power to the designated delegate.
    
    Subsequently, I delved into the **V3FactoryOwner.sol protocol** to grasp how fees are enabled for V3 Uniswap pools, collected, and distributed. Additionally, I reviewed the implementation of passthrough functions in the V3 repository and explored how the protocol incentivizes fee collection through the **Fee Payout Race** mechanism.
    
    The bulk of my analysis was dedicated to understanding the **UniStaker.sol protocol**. Initially, I researched and comprehended the Synthetix StakingRewards.sol contract, as the UniStaker contract drew significant inspiration from it. The core functionalities of staking, unstaking, and reward distribution were inherited from the Synthetix contract, while UniStaker introduced additional abstractions, notably the Deposit struct. This struct enables the specification of the beneficiary for deposited rewards, delegation of voting power by depositing user tokens, and flexibility in adjusting the staked amount for a given deposit.
    
    Furthermore, the protocol implements mechanisms to execute certain contract actions (such as stake, stakeMore, withdraw, etc.) on behalf of a user, authenticated through a signature to validate the user's intent.
    
    1. Documentation Review:
    Then I went to review the [docs](https://docs.unistaker.io/) for a more detailed explanation of Unistaker Infrastructure.
    2. Compiling code and running the provided tests.
    3. Manual Code Review
        
        In this phase, I initially conducted a line-by-line analysis, following that, I engaged in comparison mode. 
        
        - Line by line: Pay close attention to the contract’s intended functionality and compare it with its actual behaviour on a line-by-line basis.
        - Comparison Mode: Compare the implementation of each function with established standards or existing implementations, focusing on the function names to identify any deviations. 
        ****

## Codebase Quality

| Codebase Quality Categories | Comments |
| --- | --- |
| Code Maintainability and Reliability | The UniStaker contracts are meticulously crafted, prioritizing simplicity and security. Each function is purposefully designed to serve a specific task, contributing to the overall robustness of the system. This approach not only enhances the clarity of the codebase but also minimizes the potential for vulnerabilities by reducing complexity and ensuring that each function is focused on its intended functionality. |
| Code Comments | The protocol follows best practices and has annotated all needed parts of the code with NatSpec. The spec is comprehensive and gives the needed depth to understand what the functions are supposed to do.  |
| Documentation | The documentation of the UniStaker project is indeed commendable, offering a comprehensive and detailed overview of the protocol's structure and functionalities. However, there is still room for improvement, particularly in sections that are marked as under construction. Despite this, the inclusion of diagrams and flow charts significantly enhances understanding, providing clarity on the main concepts and processes within the protocol. |
| Testing | From the scope that needs to be audited the following contracts coverage is: UniStaker.sol: 100%, V3FactoryOwner.sol: 100%. The current tests have excellent coverage. |
| Code Structure and Formatting | The codebase contracts are well-structured and formatted, but some functions are not ordered according to the Solidity Style Guide. According to the guide, functions should be grouped based on their visibility and ordered as follows: constructor, receive, fallback, external, public, internal, private. Within each grouping, view and pure functions should be placed last. |
| Strengths | Comprehensive test harness featuring invariant and fuzz tests. Comprehensive NatSpec. Code simplicity.  |

### Centralization Risks

The passthrough functions listed below have **_revertIfNotAdmin()** checks and will be a risk if the admin address is compromised. 
[setFeeProtocol()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L142), [setAdmin()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L110), [enableFeeAmount()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L131), [setPayoutAmount()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L119), [setRewardNotifier()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L210)

## Recommendations

The admin V3FactoryOwner carries numerous important abilities for the system. Consider implementing a two-step process for transferring ownership of the V3FactoryOwner contract.

## Conclusion

Overall, the UniStaker protocol showcases a well-designed architecture, prioritizing simplicity and security in its implementation. Its primary strengths lie in its comprehensive testing, detailed documentation, and meticulous code design. Furthermore, it is strongly advised that the team persists in investing in security measures, including mitigation reviews, audits, and bug bounty programs, to uphold the project's security and reliability.



### Time spent:
35 hours