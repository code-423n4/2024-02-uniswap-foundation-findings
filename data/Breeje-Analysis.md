# Analysis

## Summary

| Count | Topic | 
|:--:|:-------|
| 1 | Introduction |
| 2 | Architecture Recommendation |   
| 3 | Centralization Risk |  
| 4 | Systemic risks |
| 5 | Time Spent |

## Introduction

This report highlights insights from auditing `UniStaker` Infrastructure, which enables Governance to activate Protocol fees on Uniswap pools, subsequently distributing them trustlessly to staking `UNI` Holders. Users staking UNI tokens can also delegate voting power, ensuring their continued participation in governance decisions. Here, The staking mechanism of this contract is directly inspired by the Synthetix `StakingRewards` implementation.

## Architecture Recommendation

#### 1. Limiting Future Flexibility Could Result in Gas Inefficiencies

The current architecture of `V3FactoryOwner` is depicted as follows:

![Current Architecture](https://res.cloudinary.com/davyfibzy/image/upload/v1709319396/ue9unucbumxd0fp61gk5.png)

Current Scenario:

1. Transitioning the owner of `UniswapV3Factory` to `V3FactoryOwner` via the `setOwner()` function.
2. Allowing the admin to activate fees from `V3FactoryOwner`.

It's important to recognize that once the `owner` of `UniswapV3Factory` is set to `V3FactoryOwner`, the `setOwner()` function in the `UniswapV3Factory` contract becomes permanently inaccessible. While restricting this capability might seem reasonable, it could pose challenges in future upgrades. In such scenarios, the only available option would be to invoke `setAdmin` and integrate another contract to contact `V3FactoryOwner`, resulting in additional gas overhead for every protocol action.

![Incase of Future Upgrade](https://res.cloudinary.com/davyfibzy/image/upload/v1709320343/osjebo2y0axvqgjbcdpp.png)

Given that the `admin` of `V3FactoryOwner` is also the governance itself, implementing a `setOwner` function to directly assign ownership to a new `V3FactoryOwner` contract could save gas and streamline operations.

#### 2. Implementation of `claimFees` will revert everytime MEV tries to claim 100% fees

MEV bots targeting the `claimFees` function will be the first step in the token distribution lifecycle, typically aiming to withdraw 100% of the fees.

However, a safeguard exists to protect against MEV against losses: if the amount received from fee collection is less than the requested amount, the function reverts to safeguard MEV in case of potential front-running by other bots.

```solidity

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
@->   if (_amount0 < _amount0Requested || _amount1 < _amount1Requested) {
        revert V3FactoryOwner__InsufficientFeesCollected();
      }
      emit FeesClaimed(address(_pool), msg.sender, _recipient, _amount0, _amount1);
      return (_amount0, _amount1);
    }

```

However, if you see the implementation of `collectProtocol` function:

```solidity
File: UniswapV3Pool.sol

    function collectProtocol(
        address recipient,
        uint128 amount0Requested,
        uint128 amount1Requested
    ) external override lock onlyFactoryOwner returns (uint128 amount0, uint128 amount1) {
        amount0 = amount0Requested > protocolFees.token0 ? protocolFees.token0 : amount0Requested;
        amount1 = amount1Requested > protocolFees.token1 ? protocolFees.token1 : amount1Requested;

        if (amount0 > 0) {
 @->        if (amount0 == protocolFees.token0) amount0--; // ensure that the slot is not cleared, for gas savings
            protocolFees.token0 -= amount0;
            TransferHelper.safeTransfer(token0, recipient, amount0);
        }
        if (amount1 > 0) {
 @->        if (amount1 == protocolFees.token1) amount1--; // ensure that the slot is not cleared, for gas savings
            protocolFees.token1 -= amount1;
            TransferHelper.safeTransfer(token1, recipient, amount1);
        }

        emit CollectProtocol(msg.sender, recipient, amount0, amount1);
    }

```

This function subtracts 1 in case the caller attempts to claim 100% of the fees. Although this is done to retain the slot and save gas, it results in the returned amount always being 1 less than the requested amount.

This can lead to frequent reverts and loss of gas for MEV if they call `claimFees` to claim complete fees.

This scenario should be handled in `V3FactoryOwner` itself by changing the code as shown below to protect MEV from both frequent possible reverts and Frontrunning risks.

```diff

    function claimFees(
      // SNIP //

      (uint128 _amount0, uint128 _amount1) =
        _pool.collectProtocol(_recipient, _amount0Requested, _amount1Requested);

      // Protect the caller from receiving less than requested. See `collectProtocol` for context.
-     if (_amount0 < _amount0Requested || _amount1 < _amount1Requested) {
+     if (_amount0 < _amount0Requested - 1 || _amount1 < _amount1Requested - 1) {
        revert V3FactoryOwner__InsufficientFeesCollected();
      }
      
      // SNIP
    }

```

## Centralization Risk

While granting admin access to Governance mitigates direct centralization risks, there remains a risk regarding governance attacks targeting `UNI` tokens outside the contest scope. Such attacks have the potential to inflict significant damage on the protocol.

## Systemic risks

#### 1. Absence of Deadline Enforcement on User Signatures can lead to Issues: 

* `UniStaker` contract allows users to provide intent for executing six actions on behalf of themselves to any arbitrary user by providing a signature.

* Actions include stake, stake more, alter delegatee, alter beneficiary, withdraw, and claim reward.

```solidity

  /// @notice Type hash used when encoding data for `stakeOnBehalf` calls.
  bytes32 public constant STAKE_TYPEHASH = keccak256(
    "Stake(uint256 amount,address delegatee,address beneficiary,address depositor,uint256 nonce)"
  );
  /// @notice Type hash used when encoding data for `stakeMoreOnBehalf` calls.
  bytes32 public constant STAKE_MORE_TYPEHASH =
    keccak256("StakeMore(uint256 depositId,uint256 amount,address depositor,uint256 nonce)");
  /// @notice Type hash used when encoding data for `alterDelegateeOnBehalf` calls.
  bytes32 public constant ALTER_DELEGATEE_TYPEHASH = keccak256(
    "AlterDelegatee(uint256 depositId,address newDelegatee,address depositor,uint256 nonce)"
  );
  /// @notice Type hash used when encoding data for `alterBeneficiaryOnBehalf` calls.
  bytes32 public constant ALTER_BENEFICIARY_TYPEHASH = keccak256(
    "AlterBeneficiary(uint256 depositId,address newBeneficiary,address depositor,uint256 nonce)"
  );
  /// @notice Type hash used when encoding data for `withdrawOnBehalf` calls.
  bytes32 public constant WITHDRAW_TYPEHASH =
    keccak256("Withdraw(uint256 depositId,uint256 amount,address depositor,uint256 nonce)");
  /// @notice Type hash used when encoding data for `claimRewardOnBehalf` calls.
  bytes32 public constant CLAIM_REWARD_TYPEHASH =
    keccak256("ClaimReward(address beneficiary,uint256 nonce)");

```

* Notably, there's no `deadline` parameter incorporated in these actions.

* While users are typically expected to trust the individual to whom they delegate these signatures, the lack of a deadline poses a systemic risk.

* In cases where the caller delays executing the function using the provided signature, it can lead to unexpected issues.

* Particularly concerning is the potential for the caller to strategically time the execution of actions like altering the beneficiary during periods of high reward rates, incentivizing malicious behavior if exploited.

#### 2. First Depositor can inflate rewardPerTokenAccumulatedCheckpoint:

  - `Inflation Potential`: Initiating the stake() function with only 1 wei allows the first user to significantly inflate the rewardPerTokenAccumulatedCheckpoint.

  - `Scenario Overview`:
    - Alice triggers stake() with herself as the beneficiary, depositing 1 wei.
    - This updates earningPower[Alice] to 1 wei and sets totalStaked = 1.
    - _checkpointReward(Alice) assigns beneficiaryRewardPerTokenCheckpoint using the result from rewardPerTokenAccumulated() for Alice.

  - `Calculation Mechanism`:
    - The rewardPerTokenAccumulated() function ideally divides scaledRewardRate (36 decimals) by totalStaked (assumed 18 decimals), resulting in an output of 18 decimals.

  - `Inflation Risk`:
    - If a user initiates stake() with 1 wei, the rewardPerTokenAccumulatedCheckpoint can undergo significant inflation.

  - `Previous Similar Finding`:
    - A similar vulnerability was reported in the [veToken Audit](https://code4rena.com/reports/2022-05-vetoken#m-23-baserewardpools-rewardpertokenstored-can-be-inflated-and-rewards-can-be-stolen), where the mechanism of Synthetic's StakingReward was exploited to illustrate its unfairness.

  - `Fairness Concern`:
    - Even if a user stakes 1e18 tokens initially (non-malicious behavior), it could result in unfairness up to three times the rewards in this edge case.

## Time Spent

| Total Number of Hours | 16 |
|:--:|:--:|

### Time spent:
16 hours