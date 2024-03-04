
# UniStaker Infrastructure Analysis Report

- **Table of Contents**

1. [Executive Summary](#1-executive-summary)
2. [Introduction](#2-introduction)
3. [Methodology](#3-methodology)
4. [Scope of the Analysis](#4-scope-of-the-analysis)
5. [Contract Overview](#5-contract-overview)
   - 4.1. UniStaker.sol
   - 4.2. V3FactoryOwner.sol
   - 4.3. DelegationSurrogate.sol
6. [Systemic Risks](#6-systemic-risks)
7. [Potential Attack Surfaces](#7-potential-attack-surfaces)
8. [Recommendations for Mitigation](#8-recommendations-for-mitigation)
9. [Architectural Weaknesses](#9-architectural-weaknesses)
10. [Conclusion and Future Work](#10-conclusion-and-future-work)
11. [Appendices](#11-appendices)
    - 11.1. Code Snippets
    - 11.2. References

## 1. Executive Summary

This report thoroughly examines the security & risk aspects of the UniStaker Infrastructure. Its purpose is to improve the security & functionality of the project. The analysis points potential risk & attack surfaces, offering practical recommendations for addressing them.

## 2. Introduction

These contracts enable Uniswap Governance to manage fees on V3 pools, while maintaining a approach to fee assets. The generated revenue is shared with stakers in the form of a custom token. Stakers accumulate rewards when their tokens are actively staked, & this earning ceases when they withdraw. Additionally, stakers have the flexibility to designate someone else to receive their rewards. Upon governance approval, ownership permanently transitions to this system, allowing for the incorporation of additional contracts for future rewards.

## 3. Methodology

The analysis process spanned 16 hours, I starting with a deep dive into the contract architecture followed by examination of the code for problems & concluding with of risks & recommendations.

## 4. Scope of the Analysis

There are 4 Interfaces in the scope as well but i will focus on main contracts whick are listed below:

<table><thead><tr><th>Contract</th><th>SLOC</th><th>Purpose</th><th>Libraries used</th></tr></thead><tbody><tr><td><em>Contracts (3)</em></td><td></td><td></td><td></td></tr><tr><td><a href="https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol">src/UniStaker.sol</a></td><td>423</td><td>This contract manages the distribution of rewards to stakers.</td><td><a href="https://openzeppelin.com/contracts/"><code>@openzeppelin/*</code></a></td></tr><tr><td><a href="https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol">src/V3FactoryOwner.sol</a></td><td>87</td><td>A contract that can serve as the owner of the Uniswap v3 factory and manages configuration and collection of protocol pool fees.</td><td><a href="https://openzeppelin.com/contracts/"><code>@openzeppelin/*</code></a></td></tr><tr><td><a href="https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/DelegationSurrogate.sol">src/DelegationSurrogate.sol</a></td><td>8</td><td>A dead-simple contract whose only purpose is to hold governance tokens on behalf of stakers while delegating voting power to a specific delegatee.</td><td>None</td></tr></tbody></table>

## 5. Contract Overview

### 5.1. UniStaker.sol

![Imgur](https://i.imgur.com/AAAyG1S.png)

- **Purpose**: Manages the staking mechanism, including the accumulation & distribution of rewards to stakers. It facilitates users by staking tokens, offers reward based on the staked amount & duration.

### 5.2. V3FactoryOwner.sol

![Imgur](https://i.imgur.com/nuy0MBL.png)

- **Purpose**: Acts as a governance tool for Uniswap V3 pool fee management. This contract enables configuration & collection of protocol fees, serving as a bridge between Uniswap governance decisions & the actual implementation of those decisions in the fee structure.

### 5.3. DelegationSurrogate.sol

![Imgur](https://i.imgur.com/KDyQI47.png)

- **Purpose**: Simplifies the delegation of governance power. It holds governance tokens on behalf of stakers, allowing them to delegate voting power without transferring token ownership, while maintaining security & control of their assets.

## 6. Systemic Risks

**High-Level Access Functions**:

- I have identified in both `UniStaker.sol` & `V3FactoryOwner.sol`, that these functions can be used as advantage by hacker if access control checks are bypassed or improperly implemented, allowing unauthorized users to alter critical settings or steal funds. Without suitable validation this could become a failure.

**No Limits When Setting Fees**:

- In file `V3FactoryOwner.sol`, there is absence of validation on fee settings that could lead to setting excessively high or zero fees, potentially disrupting the system. A lack of checks in the system could lead to operational risks of the system.

**Loss of Precision**:

- As we know Solidity lack of floating-point support, operations in `UniStaker.sol` involving division could result in rounded down outcomes, leading to incorrect reward calculations or Division by large numbers might cause loss of precision.

**Missing Zero Address Check in Constructor:**

- In files `UniStaker.sol` & `DelegationSurrogate.sol` constructors do not have require/validation against the zero address for parameters. This could lead to contracts being initialized with invalid state, rendering them dysfunctional or open to dos attacks.

**Approve type(uint256).max May Not Work with Some Tokens:**

- In the `DelegationSurrogate.sol` it employs an approval pattern that grants maximum allowance. This approach might not be compatible with all ERC-20 tokens, a token that doesn't uses standard implementations, would be resulting in failed transactions or unexpected contract behaviors.

### Additional Analysis

**Division by Zero Not Prevented:**

- I found in UniStaker.sol, that division operations do not include checks for divisor values being zero. This can cause problems like transactions to revert, DOS attack etc.

**Division by a Scale Factor:**

- The division by a scale factor, may result in the loss of precision of reward amounts. This issue can do unfair & affect small stakeholders by reducing their expected rewards to zero.

## 7. Potential Attack Surfaces

There is several potential attack surfaces, including them but not limited to:

- **Admin Function Misuse:** Functions requiring admin permission could be compromised & allow attackers to alter contract behavior, including changing parameters like reward distribution or admin controls.
- **Arithmetic Overflows and Underflows:** without using proper checks arithmetic operations could overflow, leading to unintended behavior or contract remain at risk.
- **Surrogate Contract Deployment and Management:** The mechanism for deploying & managing surrogate contracts for delegation purposes must ensure that these contracts cannot be manipulated or misused to alter the intended delegation logic.
- **Smart Contract Initialization Risks:** There is absence of zero address checks during contract initialization could be used to create contracts in an invalid state, potentially leading to operational disruptions.
- **Approval Mechanism Exploits:** The practice of granting maximum token allowances can introduce problems, especially with tokens having non-standard behaviors, potentially leading to unauthorized fund access or manipulation.

## 8. Recommendations for Mitigation

- Should implement comprehensive access control mechanisms, such as OpenZeppelin's AccessControl, to manage different levels of permissions & reduce the risk of unauthorized access. Ensure all user inputs & contract calls are validated against extreme values & edge cases to prevent manipulation.
- Should Include reentrancy guards, such as OpenZeppelin's ReentrancyGuard in functions performing external calls or state changes to prevent reentrancy attacks.
- Utilize SafeMath and proper checks to prevent overflows & underflows, ensuring arithmetic operations behave as expected.
- Should implement strict access control mechanisms within the UniStaker contract to ensure that only authorized operations can deploy new surrogate contracts or modify delegation settings. This could involve checks to ensure that actions such as changing a delegatee or deploying a surrogate are performed by legitimate person.
- Avoid using type(uint256).max for token approvals. Instead approve only the necessary amount of tokens required to perform operation this will reducing the risk associated with unlimited allowances.
- Should Implements checks to ensure divisor values are never zero before performing division operations, safeguarding against potential reverts & ensuring the reliability of calculations.

## 9. Architectural Weaknesses

Too much dependence on the owner for critical functions introduces centralization making the system vulnerable to single points of failure or malicious actions by the owner. The contracts design & token standards may not be fully compatible across different EVM-compatible chains limiting integration. The use of approve with type(uint256).max could lead to unexpected failures with non-standard tokens. Some patterns & operations in the contracts may lead to higher gas costs affecting the overall efficiency of the system.

## 10. Conclusion and Future Work

The UniStaker Infrastructure exhibits a well approach to staking & governance. However, the identified Risks & architectural concerns highlight the necessity for continuous security practices & improvements. Addressing these issues through the recommended mitigation strategies will not only enhance the project's security posture but also its operational efficiency & user trust.

Future work should focus on access control mechanisms, optimizing gas usage, & ensuring cross chain compatibility to the protocol's resilience & adaptability in the DeFi landscape. Transitioning towards a more decentralized governance structure could mitigate risks associated with centralization & enhance community trust & protocol resilience.

By addressing these identified problems & giving to the recommended mitigation strategies, the UniStaker Infrastructure can significantly improve its system & security, compatibility, & decentralization ensuring the project remains robust against problems & threats while providing the needs of its users.

## 11. Appendices

### 11.1. Code Snippets

This section includes some code snippets related to identified risks a closer look at issues & where improvements can be made.

#### High-Level Access Function in UniStaker.sol

``` solidity
function setAdmin(address _newAdmin) external {
    _revertIfNotAdmin(); // Risk Point
    _setAdmin(_newAdmin);
}
```

Issue: Lack of sufficient access control beyond the _revertIfNotAdmin check could lead to unauthorized changes.

#### Division by Zero Potential in UniStaker.sol

``` solidity
function unclaimedReward(address _beneficiary) public view returns (uint256) {
    return unclaimedRewardCheckpoint[_beneficiary]
        + (
            earningPower[_beneficiary]
            * (rewardPerTokenAccumulated() - beneficiaryRewardPerTokenCheckpoint[_beneficiary])
        ) / SCALE_FACTOR; // Potential Precision Loss
}
```

Issue: Division without prior validation could result in precision loss or unexpected behavior.

```solidity
function calculateReward(uint256 amount) public view returns (uint256) {
    require(SCALE_FACTOR != 0, "SCALE_FACTOR cannot be zero"); // Recommended fix
    return (amount * rewardRate) / SCALE_FACTOR;
}
```

Issue: Without ensuring the SCALE_FACTOR or any divisor is non-zero, the division operation is at risk of causing a revert, disrupting the contract's functionality.

#### Missing Zero Address Check in Constructor

UniStaker.sol Constructor:

```solidity
constructor(IERC20 _rewardToken, IERC20Delegates _stakeToken, address _admin) {
    require(_admin != address(0), "Admin address cannot be zero"); // Recommended fix
    REWARD_TOKEN = _rewardToken;
    STAKE_TOKEN = _stakeToken;
    _setAdmin(_admin);
}
```

Issue: Original constructor lacks a check for _admin being the zero address, which is critical for ensuring the contract is initialized with valid parameters.

#### Approve type(uint256).max May Not Work with Some Tokens

DelegationSurrogate.sol Approval Pattern:

```solidity
_token.approve(msg.sender, type(uint256).max);
```

Issue: This pattern assumes all ERC-20 tokens will accept & correctly handle an approval of type(uint256).max, which may not hold true for tokens with non-standard behaviors.

### 11.2. References

- [Solidity Documentation](https://docs.soliditylang.org)
- [OpenZeppelin Contracts: A library for secure smart contract development](https://openzeppelin.com/contracts/)
- [Ethereum EIP-2612: Standard for permit – 712-signed approvals](https://eips.ethereum.org/EIPS/eip-2612)
- Reentrancy Guard: A mechanism to prevent reentrancy attacks, often implemented using OpenZeppelin's ReentrancyGuard utility.


### Time spent:
16 hours