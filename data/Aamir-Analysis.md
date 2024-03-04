# Comprehensive Audit Report

## Introduction

The UniStaker contract serves as a pivotal component within the Uniswap V3 ecosystem, facilitating the distribution of rewards to stakers who delegate and stake their UNI tokens. This report offers a detailed analysis of the UniStaker contract and other contracts in scope, encompassing their functionality, architecture, audit methodology, findings, recommendations, and broader implications within the decentralized finance (DeFi) landscape.

## Protocol Overview

![UniStaker Banner](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/unistaker.png?raw=true)

Uniswap V3 revolutionizes decentralized finance (DeFi) with its advanced automated market maker (AMM) solution, introducing concentrated liquidity positions and flexible fee structures. Unlike its predecessors, Uniswap V3 empowers liquidity providers with granular control over their capital allocation within specified price ranges, enhancing capital efficiency and reducing impermanent loss. Additionally, the protocol enables governance token holders to set and manage protocol fees, fostering community participation and sustainability.

Within this context, the UniStaker contract plays a crucial role by incentivizing UNI token holders to actively engage in governance and liquidity provision. Through staking and delegation mechanisms, users contribute to the protocol's growth and earn rewards, thereby enhancing decentralization and community ownership. Here is a brief overview about contracts in Scope of this audit.

### UniStaker Contract

The UniStaker contract is responsible for managing the distribution of rewards to stakers based on their delegated UNI tokens. By staking UNI tokens and delegating governance voting weight, users actively participate in protocol governance while earning rewards proportional to their staked amount. The contract's modular architecture, built on OpenZeppelin standards, ensures robustness and security.

### Integration with UniV3FactoryOwner

The UniStaker contract collaborates closely with the UniV3FactoryOwner contract, which serves as the owner of the Uniswap V3 factory. The UniV3FactoryOwner contract enables governance to set and manage protocol fees for Uniswap V3 pools. By seamlessly integrating with UniStaker, the UniV3FactoryOwner contract ensures that rewards generated from protocol fees are trustlessly distributed to stakers who delegate their UNI tokens.

### DelegationSurrogate Contract

In addition to the UniV3FactoryOwner contract, the UniStaker contract interacts with the DelegationSurrogate contract. This contract facilitates delegation of governance voting weight while holding UNI tokens on behalf of stakers. By delegating voting power to specified addresses, users enhance their governance participation and flexibility within the Uniswap V3 ecosystem.

## Contract Analysis

1. [**UniStaker.sol**](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol):

   - **SLOC**: 423
   - **Purpose**: Manages reward distribution to stakers based on delegated UNI tokens.
   - **Functionality**: Implements staking, delegation, reward distribution, and beneficiary designation functionalities.
   - **Architecture**: Structured and modular, leveraging OpenZeppelin contracts for standardization and security.
   - **Security Features**: Utilizes access control mechanisms and safe math operations to mitigate potential vulnerabilities.

2. [**V3FactoryOwner.sol**](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol):

   - **SLOC**: 87
   - **Purpose**: Acts as owner of the Uniswap v3 factory, facilitating configuration and collection of protocol fees.
   - **Functionality**: Enables governance to set and manage protocol fees for Uniswap V3 pools.
   - **Integration**: Seamlessly interacts with UniStaker contract for reward distribution and governance delegation.

3. [**DelegationSurrogate.sol**](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/DelegationSurrogate.sol):

   - **SLOC**: 8
   - **Purpose**: Facilitates delegation of governance voting weight while holding UNI tokens on behalf of stakers.
   - **Functionality**: Delegates voting power to specified addresses, enhancing governance participation and flexibility.
   - **Security Considerations**: Simplified design reduces attack surface and potential vulnerabilities.

4. **Interfaces**:
   - [**IERC20Delegates.sol**](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/IERC20Delegates.sol):
     - **SLOC**: 22
     - **Purpose**: Subset of the ERC20Votes-style governance token to which UNI conforms.
   - [**INotifiableRewardReceiver.sol**](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/INotifiableRewardReceiver.sol):
     - **SLOC**: 4
     - **Purpose**: Communication interface between the V3FactoryOwner contract and the UniStaker contract.
   - [**IUniswapV3FactoryOwnerActions.sol**](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/IUniswapV3FactoryOwnerActions.sol):
     - **SLOC**: 6
     - **Purpose**: Required subset of the interface for the Uniswap V3 Factory.
   - [**IUniswapV3PoolOwnerActions.sol**](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/IUniswapV3PoolOwnerActions.sol):
     - **SLOC**: 7
     - **Purpose**: Interface for pool methods that may only be called by the factory owner.

## Findings Summary

### Severity Categorization

#### QA (Quality Assurance)

Includes both Non-critical (code style, clarity, syntax, versioning, off-chain monitoring (events, etc)) and Low risk (e.g., assets are not at risk: state handling, function incorrect as to spec, issues with comments). Excludes Gas optimizations, which are submitted and judged separately.

#### Severity Levels:

- 1 — Low: Non-critical issues that do not pose a direct risk to assets or protocol functionality but may impact code quality or clarity.
- 2 — Med: Assets not at direct risk, but the function of the protocol or its availability could be impacted, or leak value with a hypothetical attack path with stated assumptions, but external requirements.
- 3 — High: Assets can be stolen/lost/compromised directly (or indirectly if there is a valid attack path that does not have hand-wavy hypotheticals).

### Audit Findings

| Finding Title                              | Severity | File          | GitHub Link                                                                                                                                    |
| ------------------------------------------ | -------- | ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Change the claimFee() to accept `amount-1` | QA-1     | UniStaker.sol | [Link](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L220C1-L223C4) |
| Mistakes in Natspac                        | QA-1     | UniStaker.sol | [Link](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L18)           |
| No need for else statement                 | QA-1     | UniStaker.sol | [Link](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L220C1-L223C4) |

## Approach Used

![Screenshot 2024-03-03 155016](https://github.com/code-423n4/2024-02-uniswap-foundation/assets/90125875/89e48d38-01a0-4796-a4d1-65a5720c5e94)

For this audit, a comprehensive approach combining manual analysis and rigorous fuzzing techniques was implemented to ensure thorough scrutiny of the smart contracts. The following methodologies and tools were employed:

- **Manual Analysis**:

  - Extensive review of the codebase to identify potential vulnerabilities, inconsistencies, and deviations from best practices.
  - Verification of code clarity, logic flow, and compliance with specifications.
  - Assessment of contract architecture and design patterns to identify potential security risks.

- **Fuzzing Techniques**:
  - Utilization of Echidna, a powerful fuzzer, to generate over `700` lines of fuzzing for `100,000` runs.
  - Focused testing on critical functions and edge cases to uncover potential vulnerabilities.
- **Automated Tools**:

  - Deployment of Slither for automated static analysis to identify common vulnerabilities, potential bugs, and security issues.
  - Utilization of Foundry to automate various testing scenarios and detect vulnerabilities.

- **Total Time Spent**: Approximately 80 hours were dedicated to the audit process, ensuring thoroughness and meticulousness in analysis and testing.

## Centralization Risks

The current codebase entails certain centralization risks due to the presence of functions managing critical aspects such as setting payout amounts, adding new reward notifiers, and configuring fees for Uniswap V3 pools. While these functions provide flexibility and control over protocol parameters, they also introduce potential vulnerabilities if not handled with caution.

One notable concern is the management of the `PAYOUT_AMOUNT`, a parameter crucial for reward distribution. It's imperative to ensure that this value is set to reasonable and rounded amounts, such as `25 WETH`, to maintain stability and prevent potential issues. However, if excessively large or irregular values are used, such as `3234342342343243232`, it could lead to significant disruptions in the reward notification process.

For instance, setting excessively high PAYOUT_AMOUNT values could result in a denial-of-service (DoS) scenario during reward notification. The system may struggle to handle the processing load associated with distributing rewards, potentially leading to delays, inefficiencies, or even system failures. Moreover, irregular values may introduce complexities in calculations, affecting the accuracy and reliability of reward distribution mechanisms.

Additionally, the ability to add new reward notifiers and configure fees for Uniswap V3 pools introduces centralization risks related to governance and control. Improper management or manipulation of these parameters could lead to unfair advantage for certain participants, manipulation of reward distribution, or disruption of protocol operations.

To mitigate these centralization risks, it's essential to implement robust governance mechanisms, transparent decision-making processes, and thorough testing protocols. Regular audits, community participation, and adherence to best practices can enhance the resilience and decentralization of the protocol, ensuring its long-term sustainability and security. Continued diligence, community involvement, and adherence to best practices will be paramount in ensuring the stability, integrity, and resilience of the system

### Conclusion

In summary, the audit of the UniStaker protocol unveiled a robust and well-structured codebase, demonstrating a high level of solidity and resilience. The protocol's architecture showcases thoughtful design considerations and meticulous attention to detail, contributing to its overall reliability and security.

Throughout the audit process, only a few informational findings were identified, underscoring the protocol's strong security posture and resistance to vulnerabilities. The development team's adherence to best practices and thorough testing methodologies has contributed to the protocol's solidity and robustness.

Moving forward, it is recommended that the development team continues to prioritize ongoing monitoring and maintenance efforts to ensure the protocol remains resilient to emerging threats and challenges. Additionally, proactive engagement with the community and adoption of industry best practices will further strengthen the protocol's position as a trusted and dependable solution within the decentralized finance (DeFi) ecosystem.

### Acknowledgements

The audit team extends its appreciation to the development team for their cooperation and support throughout the audit process.

### Disclaimer

This audit report is based on the code and documentation available at the time of the audit and is subject to change based on future updates or revisions to the protocol. The findings and recommendations provided in this report are intended for informational purposes only and should not be considered as financial or investment advice.

### End of Report


### Time spent:
80 hours