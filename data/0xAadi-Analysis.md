# Analysis Report - UniStaker Infrastructure

---

## Index
1. [Overview](#overview)
2. [Contracts](#contracts)
3. [Auditing Approach](#auditing-approach)
4. [Codebase Quality Analysis](#codebase-quality-analysis)
5. [Modularity](#modularity)
6. [Comments](#comments)
7. [Access Controls](#access-controls)
8. [Error Handling](#error-handling)
9. [Modifiers](#modifiers)
10. [Interfaces](#interfaces)
11. [Libraries](#libraries)
12. [Centralisation Risk](#centralization-risk)
13. [Systemic Risk](#systemic-risk)

---

## Overview

The UniStaker system is engineered to reward UNI token holders for staking their tokens within the Uniswap protocol. Rewards, denominated in an ERC20 token, are distributed to stakers who may also delegate their governance voting rights. Below are the key features of the system as per the contract functionalities:

- **Proportional Rewards**: Stakers are entitled to rewards proportional to their stake in the total pool of staked UNI tokens. Rewards accrue as long as the tokens remain staked within the system.

- **Reward Distribution**: Stakers begin earning rewards upon depositing their tokens. The earning process is active until the tokens are withdrawn, aligning the stakers' interests with the duration of their investment.

- **Governance Participation**: While staking UNI tokens, holders retain the option to delegate their governance voting power. This delegation is not mandatory, allowing stakers to choose their level of involvement in Uniswap Governance.

- **Reward Token**: Rewards are accumulated in a specific token that is defined at the deployment of the UniStaker contract. This token is used to distribute rewards to the stakers.

## Contracts
The pivotal contracts in this Audit for the protocol are:
- **DelegationSurrogate.sol**: The DelegationSurrogate is a straightforward contract designed to custody governance tokens for users, simultaneously delegating their associated voting rights to a single chosen delegatee.
- **UniStaker.sol**: The contract oversees the allocation of rewards to participants who stake their tokens. These rewards are expressed in an ERC20 token format and are dispatched to the contract by entities with the authority to notify about rewards.
- **V3FactoryOwner.sol**: The V3FactoryOwner contract manages the Uniswap v3 factory, allowing an admin to control settings and fees. It also lets anyone pay to claim collected fees, rewarding a specified receiver.

## Auditing Approach
During my audit of the UniStaker Infrastructure protocol's smart contracts and accompanying documentation, I conducted an in-depth review of the codebase, with a focus on Solidity practices and security aspects. My approach entailed actively compiling the code and rigorously testing it across various scenarios to assess the contracts' functionality and robustness. This comprehensive analysis allowed me to scrutinize the contracts' behavior and security under a range of conditions. I also verified key invariants, contemplated potential attack vectors identified by the protocol, and took into account extra contextual details to complete my exhaustive audit process.

The attack vectors I considered included potential overflows, rounding errors not in the protocol's favor, reward distribution shortfalls, edge cases in reward notifications, and unforeseen outcomes from interactions with or configurations of a Uniswap V3 Pool. I also evaluated the possibility of governance-related economic attacks, the impact of future good-faith reward notifier contract changes, and issues arising from network upgrades, given the contracts' immutable nature. Additionally, I explored potential vulnerabilities related to account abstraction or smart contract wallet usage.

The main invariants I verified were: the accuracy of the totalStaked variable against the net sum of deposits and withdrawals; the consistency of the governance token balances across all surrogate contracts with the net deposits and withdrawals; the alignment of users' depositorTotalStaked with totalStaked; the congruence of the sum of all users' deposit balances with the net deposits and withdrawals; the match of the total delegated amounts to totalStaked; the equivalence of the sums applied to beneficiaries to totalStaked; the equality of beneficiaries' 

## Codebase Quality Analysis
The codebase stands out for its exceptional clarity and quality, with each function and module being well-documented and self-explanatory. The simplicity of the implemented features adds to the ease of understanding, while the extensive inline comments throughout the core contracts significantly improve readability.
The meticulous documentation and clear inline explanations contribute to a reduced learning curve, facilitating quicker comprehension and more efficient auditing of the code.

### Modularity
The codebase exhibits a high level of modularity, particularly within the UniStaker contract, where complex additional logics are adeptly segregated into separate functions. There is, however, room to further improve modularity by extracting the events and errors out of the UniStaker and V3FactoryOwner contracts. Adopting this strategy would lead to a codebase that is even more modular and easier to maintain.

### Comments
The contracts are well-commented, significantly clarifying the purpose and functionality of each function, which allows auditors and developers to grasp the intended behavior with less effort. Enhanced commentary that meticulously elucidates the various functionalities has substantially improved the code's accessibility and ease of understanding.

### Access Controls
The codebase features a robust implementation of access control that facilitates seamless and conflict-free execution of functions. However, the access controls are predominantly managed by a single admin role, which centralizes significant power. While this can streamline operations, it also introduces a risk; should the admin be compromised or act with malicious intent, the integrity of the entire system could be jeopardized.

### Error Handling
The contract employs strong error handling mechanisms, including input validation and conditional checks using if statements, to maintain operational integrity and provide users with clear feedback through custom error messages in case of exceptions. It utilizes custom errors, which not only offer descriptive messages for scenarios like unauthorized access, invalid inputs, and unexpected reward rates or balances but also optimize gas usage and enhance code readability, marking an improvement over the conventional require statements with error strings.

### Modifiers
The contract's preference for view functions over a modifier for access control aligns with the contract's overall coding style. This approach provided clearer code, improved reusability, and consistency throughout the contract. Ultimately, this decision reflects a preference that does not inherently compromise the contract's functionality or security.

### Interfaces
The current codebase effectively leverages interfaces and employs named imports, which streamline the integration of external functionalities and enhance the clarity of the source of various components within the contract.

### Libraries
Presently, the contracts do not make use of any libraries. Nevertheless, it is recommended to explore the incorporation of a dedicated library to encapsulate the events and errors. This approach can contribute to improved modularity and maintainability within the codebase, providing a structured and reusable foundation for the events and errors.

## Centralization Risk
The protocol addresses centralization risks and the potential for a single point of failure by incorporating admin-only functions within the UniStaker and V3FactoryOwner contracts that are under the control of Uniswap DAO Governance. This structure ensures that key decisions and sensitive operations are subject to a decentralized governance process, thereby reducing

 the likelihood of centralized control and enhancing the system's resilience against unilateral failures or manipulations.

Despite the mitigation strategies in place, if the admin roles are not assigned to a DAO but instead to individuals or a single entity, there remains a potential risk of centralization. In such scenarios, these centralized admins wield significant control over the protocol, which could lead to points of vulnerability where decisions or actions by these admins could disproportionately affect the protocol's operations and security.

It is advisable to implement functionalities that require the new admin to accept the role before the change is finalized, rather than allowing direct assignment of a new admin. This additional step acts as a safeguard against the risk of setting an invalid or unwilling admin, which could lead to operational disruptions or security vulnerabilities. Such a consent mechanism ensures that only willing and validated participants can assume critical administrative responsibilities, enhancing the protocol's robustness.

## Systemic Risk
### Lack of Upgradability Provisions
The contract is not currently structured with upgradability in mind. Should vulnerabilities or the need for improvements be identified, deploying updates to the contract could be problematic. It is advisable to consider integrating upgradability features to facilitate future enhancements and fixes.
### Absence of Pausability Feature
The contract currently lacks a pausability mechanism, which is a critical feature for emergency stopping of contract functions in case of vulnerabilities or attacks. The inclusion of such a feature would allow for a swift response to protect user funds and maintain system integrity during unforeseen events.


### Time spent:
40 hours