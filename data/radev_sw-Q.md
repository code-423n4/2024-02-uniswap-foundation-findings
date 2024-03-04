# QA Report - UniStaker Infrastructure Audit Contest

### Findings Summary

| ID  | Title                                            | Severity |
| --- | ------------------------------------------------ | -------- |
| 1   | Inconsistent Admin Setting Logic                 | Low      |
| 2   | Admin Role for Uniswap Governance Not Ensured    | Low      |
| 3   | Incorrect Comment in `UniStaker.sol#_withdraw()` | Low      |
| 4   | Naming Conventions for Mappings                  | Low      |
| 5   | Allow rewards duration to can be changed         | Low      |
| 6   | Unnecessary `else` Block                         | NC       |
| 7   | Redundant IERC20 Import                          | NC       |
| 8   | Constants in Comparisons                         | NC       |

---

## 1. Inconsistent Admin Setting Logic

### Lines of Code

- [V3FactoryOwner.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol) and [UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol)

### Description and Impact

The setting of admin in [`UniStaker.sol#setAdmin()`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L201-L204) is implemented as follow:

    ```solidity
        function setAdmin(address _newAdmin) external {
            _revertIfNotAdmin();
            _setAdmin(_newAdmin);
        }
    ```

The setting of admin in [`V3FactoryOwner.sol#setAdmin()`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L110-L115) is implemented as follow:

    ```solidity
          function setAdmin(address _newAdmin) external {
            _revertIfNotAdmin();
            if (_newAdmin == address(0)) revert V3FactoryOwner__InvalidAddress();
            emit AdminSet(admin, _newAdmin);
            admin = _newAdmin;
        }
    ```

As we can see the logic for setting the admin in both `V3FactoryOwner.sol` and `UniStaker.sol` contracts differs. This inconsistency can create confusion and potential security risks if the contracts are expected to operate under a unified governance model (very very low likelihood). Additionally all of us know that inconsistency isn't good thing.

### Tools Used

Manual code review

### Recommended Mitigation Steps

Ensure that the admin setting logic is consistent across both contracts.
My personal suggestion is to follow the admin setting logic of `UniStaker.sol` contract. That is, create internal `_setAdmin()` function in `V3FactoryOwner.sol` contract that do the zero address check. emit event and set new admin.

---

## 2. Admin Role for Uniswap Governance Not Ensured

### Lines of Code

- Deployment scripts and initial setup documentation at [GitHub Repository](https://github.com/code-423n4/2024-02-uniswap-foundation)

### Description and Impact

The documentation states that Uniswap Governance should be the `admin` of the `V3FactoryOwner` contract, but there's no enforcement of this role in the code or deployment scripts. This lack of enforcement could lead to governance issues or misalignment with the intended administrative control.

> In the documentation writes the following: “Uniswap Governance is the `admin` of the `V3FactoryOwner` contract.” ([here](https://docs.unistaker.io/architecture/overview#:~:text=Uniswap%20Governance%20is%20the%20admin%20of%20the%20V3FactoryOwner%20contract.))

### Tools Used

Manual code review and analysis of deployment scripts

### Recommended Mitigation Steps

Ensure that the deployment scripts or initial setup functions explicitly set Uniswap Governance as the `admin` of the `V3FactoryOwner` contract to align with the documented governance model.

---

## 3. Incorrect Comment in `UniStaker.sol#_withdraw()` Function

### Lines of Code

- [Incorrect comment in UniStaker.sol#\_withdraw()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol)

### Description and Impact

The comment incorrectly states "overflow prevents withdrawing more than balance" when it should accurately describe the situation as preventing underflow. This miscommenting could lead to confusion about the contract's safety mechanisms.

```solidity
deposit.balance -= _amount; *// overflow prevents withdrawing more than balance` →* `deposit.balance -= \_amount; *// underflow prevents withdrawing more than balance\*
```

### Tools Used

Manual code review

#### Recommended Mitigation Steps

Correct the comment to accurately reflect that it is an underflow check: `// underflow prevents withdrawing more than balance`.

---

## 4. Naming Conventions for Mappings

### Lines of Code

- [`beneficiaryRewardPerTokenCheckpoint` and `isRewardNotifier` in UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol)

### Description and Impact

The mappings `beneficiaryRewardPerTokenCheckpoint` and `isRewardNotifier` in `UniStaker.sol` do not follow the naming conventions as other mappings in the contract, leading to code inconsistency and readability issues.

### Tools Used

Manual code review

### Recommended Mitigation Steps

Do the `beneficiaryRewardPerTokenCheckpoint` and `isRewardNotifier` mappings in `UniStaker.sol` contract named mappings as all other mappings are named mappings.

---

## 5. Allow rewards duration to can be changed

### Lines of Code

The recommendation pertains to the `UniStaker.sol` contract within the UniStaker Infrastructure Protocol, specifically regarding the enhancement of the rewards distribution mechanism. The current implementation does not include a function to adjust the rewards duration dynamically. For reference, see the Synthetix `StakingRewards.sol` implementation, which includes a `setRewardDuration()` function: [Synthetix StakingRewards.sol#L141-L148](https://github.com/Synthetixio/synthetix/blob/develop/contracts/StakingRewards.sol#L141-L148).

### Description and Impact

The `UniStaker.sol` contract manages the distribution of rewards to stakers, with a fixed reward duration defined at deployment. This fixed duration lacks flexibility in adjusting to changing conditions or strategies that may benefit the protocol and its participants. In contrast, the Synthetix `StakingRewards.sol` contract includes a mechanism for dynamically adjusting the reward duration, offering greater adaptability.

The ability to adjust the rewards duration can have several impacts:

- **Adaptability**: It allows the protocol to adapt to changing market conditions or strategic goals, optimizing reward distribution for stakers.
- **Incentive Alignment**: Adjusting the duration can help align incentives more closely with the protocol's objectives, potentially increasing participation or securing the protocol more effectively.
- **Risk Management**: In scenarios where the protocol needs to conserve resources or extend reward distributions due to unforeseen circumstances, adjusting the duration can be a valuable tool.

_Note: This is common practice staking contracts that follow the synthetix logic_

### Tools Used

Manual Code Review

### Recommended Mitigation Steps

Step 1. Change the `REWARD_DURATION` constant in `UniStaker.sol` contract to public state variable with initial value of `30 days`.
Step 2. Implement a `setRewardDuration()` function in the `UniStaker.sol` contract, similar to the Synthetix implementation. This function should allow the protocol's governance or designated admin to adjust the reward duration.

Example Implementation:

```solidity
contract UniStaker {
    // Existing contract variables and functions
-   uint256 public constant REWARD_DURATION = 30 days;
+   uint256 public rewardDuration = 30 days; // Assuming this exists
+   event RewardDurationUpdated(uint256 newDuration);

    // Constructor and other functions

    /**
     * @notice Updates the reward duration for the staking contract.
     * @param _newDuration The new reward duration in seconds.
     */
    function setRewardDuration(uint256 _newDuration) external {
        require(msg.sender == admin, "UniStaker: Unauthorized");
        require(_newDuration > 0, "UniStaker: Invalid duration");
        require(
            block.timestamp > rewardEndTime,
            "Previous rewards period must be complete before changing the duration for the new period"
        );

        rewardDuration = _newDuration;
        emit RewardDurationUpdated(_newDuration);
    }
}

```

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

## 9. The slippage protection in `V3FactoryOwner.sol#claimFees()` function isn’t efficient when multiple fee tiers are present

### Description and Impact

The `V3FactoryOwner.sol#claimFees()` function is designed to allow the claiming of protocol fees from Uniswap V3 pools. This function plays a crucial role in the management of liquidity and fee collection within the ecosystem. However, the slippage protection in `V3FactoryOwner.sol#claimFees()` function isn’t efficient when multiple fee tiers are present

This is that, because the slippage protection mechanism does not account for the variances in liquidity and price impact across different fee tiers. In a multi-tier fee environment, pools can be imbalanced in various directions, and even if funds are supplied in the same proportion, they might provide liquidity at a less favorable price due to these imbalances.

### Recommended Mitigation Steps

```diff
  function claimFees(
    IUniswapV3PoolOwnerActions _pool,
    address _recipient,
    uint128 _amount0Requested,
    uint128 _amount1Requested
  ) external returns (uint128, uint128) {
    PAYOUT_TOKEN.safeTransferFrom(msg.sender, address(REWARD_RECEIVER), payoutAmount);
    REWARD_RECEIVER.notifyRewardAmount(payoutAmount);
    (uint128 _amount0, uint128 _amount1) =
      _pool.collectProtocol(_recipient, _amount0Requested, _amount1Requested);

    // Protect the caller from receiving less than requested. See `collectProtocol` for context.
    if (_amount0 < _amount0Requested || _amount1 < _amount1Requested) {
      revert V3FactoryOwner__InsufficientFeesCollected();
    }

+   if (_amount0 * _amount1 > _amount1Requested * _amount2Requested) {
+    revert Errors.Enigma_RebalanceBelowMinAmounts(mint0Amount, mint0Amount, minDeposit0, minDeposit1);
+   }

    emit FeesClaimed(address(_pool), msg.sender, _recipient, _amount0, _amount1);
    return (_amount0, _amount1);
  }

  /// @notice Ensures the msg.sender is the contract admin and reverts otherwise.
  /// @dev Place inside external methods to make them admin-only.
  function _revertIfNotAdmin() internal view {
    if (msg.sender != admin) revert V3FactoryOwner__Unauthorized();
  }
```

---

## There isn't need to update `rewardPerTokenAccumulatedCheckpoint` state variable in the beginning of `UniStaker.sol#notifyRewardAmount()` function, because duting staking, withdrawing and claiming the `rewardPerTokenAccumulatedCheckpoint` state variable is updated together with `beneficiaryRewardPerTokenCheckpoint` state variable

```solidity
  function notifyRewardAmount(uint256 _amount) external {
    if (!isRewardNotifier[msg.sender]) revert UniStaker__Unauthorized("not notifier", msg.sender);

    // We checkpoint the accumulator without updating the timestamp at which it was updated, because
    // that second operation will be done after updating the reward rate.
    rewardPerTokenAccumulatedCheckpoint = rewardPerTokenAccumulated();

    if (block.timestamp >= rewardEndTime) {
      scaledRewardRate = (_amount * SCALE_FACTOR) / REWARD_DURATION;
    } else {
      uint256 _remainingReward = scaledRewardRate * (rewardEndTime - block.timestamp);
      scaledRewardRate = (_remainingReward + _amount * SCALE_FACTOR) / REWARD_DURATION;
    }

    rewardEndTime = block.timestamp + REWARD_DURATION;
    lastCheckpointTime = block.timestamp;

    if ((scaledRewardRate / SCALE_FACTOR) == 0) revert UniStaker__InvalidRewardRate();

    if (
      (scaledRewardRate * REWARD_DURATION) > (REWARD_TOKEN.balanceOf(address(this)) * SCALE_FACTOR)
    ) revert UniStaker__InsufficientRewardBalance();

    emit RewardNotified(_amount, msg.sender);
  }
```

**Referance: [Synthetix StakingRewards.sol implementation](https://github.com/Synthetixio/synthetix/blob/develop/contracts/StakingRewards.sol#L113-L132)**