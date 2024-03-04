# QA Report - UniStaker Infrastructure Audit Contest

### Findings Summary

| ID  | Title                                            | Severity |
| --- | ------------------------------------------------ | -------- |
| 1   | Inconsistent Admin Setting Logic                 | NC       |
| 2   | Incorrect Comment in `UniStaker.sol#_withdraw()` | Low      |
| 3   | Naming Conventions for Mappings                  | Low      |
| 4   | Missing Logic for `_delegatee` Parameter Check   | Medium   |
| 5   | Admin Role for Uniswap Governance Not Ensured    | High     |
| 6   | Unnecessary `else` Block                         | Low      |
| 7   | Redundant IERC20 Import                          | Low      |
| 8   | Constants in Comparisons                         | Low      |

---

## 1. Inconsistent Admin Setting Logic

### Lines of Code

- [V3FactoryOwner.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol) and [UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol)

### Description and Impact

The logic for setting the admin in both `V3FactoryOwner.sol` and `UniStaker.sol` contracts differs, potentially leading to inconsistent administrative controls across the contracts. This inconsistency can create confusion and potential security risks if the contracts are expected to operate under a unified governance model.

### Tools Used

Manual code review

### Recommended Mitigation Steps

Ensure that the admin setting logic is consistent across both contracts. If both contracts are meant to operate under the same governance mechanism, consider implementing a shared interface or base contract that standardizes the admin setting and governance controls.

---

## 2. Incorrect Comment in `UniStaker.sol#_withdraw()` Function

### Lines of Code

- [Incorrect comment in UniStaker.sol#\_withdraw()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol)

### Description and Impact

The comment incorrectly states "overflow prevents withdrawing more than balance" when it should accurately describe the situation as preventing underflow. This miscommenting could lead to confusion about the contract's safety mechanisms.

### Tools Used

Manual code review

#### Recommended Mitigation Steps

Correct the comment to accurately reflect that it is an underflow check: `// underflow prevents withdrawing more than balance`.

---

## 3. Naming Conventions for Mappings

### Lines of Code

- [`beneficiaryRewardPerTokenCheckpoint` and `isRewardNotifier` in UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol)

### Description and Impact

The mappings `beneficiaryRewardPerTokenCheckpoint` and `isRewardNotifier` in `UniStaker.sol` do not follow the naming conventions as other mappings in the contract, potentially leading to code inconsistency and readability issues.

### Tools Used

Manual code review

### Recommended Mitigation Steps

Rename the mappings to follow a consistent naming convention used throughout the contract, enhancing code readability and maintainability.

---

## 4. Missing Logic for `_delegatee` Parameter Check

### Lines of Code

- [UniStaker.sol#stake(uint256 \_amount, address delegatee)](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol)

### Description and Impact

The documentation suggests functionality for handling `address(0)` as a `_delegatee` parameter, but the actual function does not implement this logic. This discrepancy between documentation and implementation can lead to unexpected behavior or limitations in functionality.

### Tools Used

- Manual code review

### Recommended Mitigation Steps

Implement the missing logic to handle `address(0)` for the `_delegatee` parameter as described in the documentation, or update the documentation to reflect the current implementation accurately.

---

## 5. Admin Role for Uniswap Governance Not Ensured

### Lines of Code

- Deployment scripts and initial setup documentation at [GitHub Repository](https://github.com/code-423n4/2024-02-uniswap-foundation)

### Description and Impact

The documentation states that Uniswap Governance should be the `admin` of the `V3FactoryOwner` contract, but there's no enforcement of this role in the code or deployment scripts. This lack of enforcement could lead to governance issues or misalignment with the intended administrative control.

### Tools Used

Manual code review and analysis of deployment scripts

### Recommended Mitigation Steps

Ensure that the deployment scripts or initial setup functions explicitly set Uniswap Governance as the `admin` of the `V3FactoryOwner` contract to align with the documented governance model.

---

## 6. Unnecessary `else` Block

### Lines of Code

- [UniStaker.sol, Lines 221-222](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol/#L221-L222)

### Description and Impact

The use of an unnecessary `else` block for a return statement can slightly increase complexity and reduce code readability. Simplifying control structures where possible can enhance code clarity.

#### Tools Used

Manual code review

#### Recommended Mitigation Steps

Refactor the conditional to remove the `else` block, simplifying the control flow for improved readability.

---

## 7. Redundant IERC20 Import

### Lines of Code

- Multiple files, including [UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol) and [V3FactoryOwner.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol)

```solidity
📁 File: src/UniStaker.sol

7: import {IERC20} from "openzeppelin/token/ERC20/IERC20.sol";
8: import {SafeERC20} from "openzeppelin/token/ERC20/utils/SafeERC20.sol";

```

[7](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol/#L7-L8)

```solidity
📁 File: src/V3FactoryOwner.sol

7: import {IERC20} from "openzeppelin/token/ERC20/IERC20.sol";
8: import {SafeERC20} from "openzeppelin/token/ERC20/utils/SafeERC20.sol";

```

[7](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol/#L7-L8)

### Description and Impact

The redundant import of `IERC20` alongside `SafeERC20` increases code verbosity and can lead to confusion about the usage of ERC20 interfaces within the contract.

### Tools Used

Manual code review

### Recommended Mitigation Steps

Consolidate the imports by removing the direct import of `IERC20` where `SafeERC20` is also imported, as `SafeERC20` already includes `IERC20`.

---

## 8. Constants in Comparisons

### Lines of Code

- Instances in [UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol) and [V3FactoryOwner.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol)

```solidity
📁 File: src/UniStaker.sol

/// @audit move 0 to the left
230:     if (totalStaked == 0) return rewardPerTokenAccumulatedCheckpoint;

/// @audit move 0 to the left
587:     if ((scaledRewardRate / SCALE_FACTOR) == 0) revert UniStaker__InvalidRewardRate();

/// @audit move 0 to the left
745:     if (_reward == 0) return;

```

[230](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol/#L230-L230), [587](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol/#L587-L587), [745](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol/#L745-L745)

```solidity
📁 File: src/V3FactoryOwner.sol

/// @audit move 0 to the left
96:     if (_payoutAmount == 0) revert V3FactoryOwner__InvalidPayoutAmount();

/// @audit move 0 to the left
121:     if (_newPayoutAmount == 0) revert V3FactoryOwner__InvalidPayoutAmount();

```

[96](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol/#L96-L96), [121](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol/#L121-L121)

### Description and Impact

Constants appearing on the right side of comparisons diverge from the best practice of Yoda conditions. While not critical in Solidity, adhering to consistent practices can improve code readability.

Putting constants on the left side of comparison statements is a best practice known as Yoda conditions. Although solidity's static typing system prevents accidental assignments within conditionals, adopting this practice can improve code readability and consistency, especially when working across multiple languages.

### Tools Used

Manual code review

### Recommended Mitigation Steps

Adopt Yoda conditions by placing constants on the left side of comparisons for consistency and readability.

---

---

---

#

#

#

---

---

---

1. The logic for admin setting in `V3FactoryOwner.sol` and `UniStaker.sol` should be implemented identically.
2. Wrong comment in one of the lines in `UniStaker.sol#_withdraw()` function.

   `deposit.balance -= _amount; *// overflow prevents withdrawing more than balance` →* `deposit.balance -= \_amount; *// underflow prevents withdrawing more than balance\*`

3. Do the `beneficiaryRewardPerTokenCheckpoint` and `isRewardNotifier` mappings in `UniStaker.sol` contract named mappings as all other mappings are named mappings.
4. In the dev comments for `UniStaker.sol#stake(uint256 _amount, address delegatee)` function writes “The delegatee may not be the zero address. The deposit will be owned by the message sender, and the beneficiary will also be the message sender”. However this function doesn’t contains logic that allow user to specify `address(0)` for `_delegatee` parameter (so, implement something like this `delegatee == address(0) ? delegatee = msg.sender...`). [proof](https://www.notion.so/9d823b70a510468ba4450f77fb5e0e3e?pvs=21)
5. In the documentation writes the following: “Uniswap Governance is the `admin` of the `V3FactoryOwner` contract.” (_[here](https://www.notion.so/Audit-Contests-6bf37770dcb04b6a9b8d1afc878bd43f?pvs=21)_), But this is not ensured anywhere (even in deploy scripts).
6. `else`block not required

   One level of nesting can be removed by not having an `else` block when the `if`-block returns, and `if (foo) { return 1; } else { return 2; }` becomes `if (foo) { return 1; } return 2;`

   <i>There is one instance of this issue:</i>

   ```solidity
   📁 File: src/UniStaker.sol

   221:     if (rewardEndTime <= block.timestamp) return rewardEndTime;
   222:     else return block.timestamp;
   ```

   [221](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol/#L221-L222)

7. Redundant IERC20 import

   IERC20 is already present in SafeERC20 or ERC20, no need to import it twice

   Replace

   ```solidity
   import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
   import { SafeERC20 } from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

   ```

   with

   ```solidity
   import { SafeERC20, IERC20 } from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

   ```

   <details>
   <summary><i>There are 4 instances of this issue:</i></summary>

   ```solidity
   📁 File: src/UniStaker.sol

   7: import {IERC20} from "openzeppelin/token/ERC20/IERC20.sol";
   8: import {SafeERC20} from "openzeppelin/token/ERC20/utils/SafeERC20.sol";

   ```

   [7](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol/#L7-L8)

   ```solidity
   📁 File: src/V3FactoryOwner.sol

   7: import {IERC20} from "openzeppelin/token/ERC20/IERC20.sol";
   8: import {SafeERC20} from "openzeppelin/token/ERC20/utils/SafeERC20.sol";

   ```

   [7](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol/#L7-L8)

   </details>

8. Constants in comparisons should appear on the left side

   Putting constants on the left side of comparison statements is a best practice known as [Yoda conditions](https://en.wikipedia.org/wiki/Yoda_conditions). Although solidity's static typing system prevents accidental assignments within conditionals, adopting this practice can improve code readability and consistency, especially when working across multiple languages.

   <details>
   <summary><i>There are 5 instances of this issue:</i></summary>

   ```solidity
   📁 File: src/UniStaker.sol

   /// @audit move 0 to the left
   230:     if (totalStaked == 0) return rewardPerTokenAccumulatedCheckpoint;

   /// @audit move 0 to the left
   587:     if ((scaledRewardRate / SCALE_FACTOR) == 0) revert UniStaker__InvalidRewardRate();

   /// @audit move 0 to the left
   745:     if (_reward == 0) return;

   ```

   [230](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol/#L230-L230), [587](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol/#L587-L587), [745](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol/#L745-L745)

   ```solidity
   📁 File: src/V3FactoryOwner.sol

   /// @audit move 0 to the left
   96:     if (_payoutAmount == 0) revert V3FactoryOwner__InvalidPayoutAmount();

   /// @audit move 0 to the left
   121:     if (_newPayoutAmount == 0) revert V3FactoryOwner__InvalidPayoutAmount();

   ```

   [96](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol/#L96-L96), [121](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol/#L121-L121)

   </details>

9. Allow rewards duration to can be changed.
10. https://github.com/code-423n4/2022-02-concur-findings/issues/209
11. The slippage protection in `V3FactoryOwner.sol#claimFees()` function isn’t efficient when multiple fee tiers are present
