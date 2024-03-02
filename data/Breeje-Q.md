# QA Report

## Low Risk Issues
| Count | Explanation |
|:--:|:-------|
| [L-01] | MEV will be DoSed whenever it tries to collect complete Protocol Fees |  
| [L-02] | Lack of Enforced deadline for signature can be exploited |  
| [L-03] | First Depositor can inflate `rewardPerTokenAccumulatedCheckpoint` to steal rewards |

| Total Low Risk Issues | 3 |
|:--:|:--:|

## Non-Critical Issues
| Count | Explanation | 
|:--:|:-------|
| [N-01] | `setOwner` function in Factory contract will be permanently disabled | 

| Total Non-Critical Issues | 1 |
|:--:|:--:|

### [L-01] MEV will be DoSed whenever it tries to collect complete Protocol Fees

## Description

The `claimFees` function will be used by MEVs to collect Fees from Pool & pay back `PAYOUT_TOKEN` to the `UniStaker` which will be distributed to `UNI` stakers.

At line `a` in the code snippet below, the contract invokes `collectProtocol` on the Uniswap V3 Pool, transferring the fees to the designated `recipient`.

```solidity

    function claimFees(
      // SNIP //

      (uint128 _amount0, uint128 _amount1) =
a->     _pool.collectProtocol(_recipient, _amount0Requested, _amount1Requested);

      // Protect the caller from receiving less than requested. See `collectProtocol` for context.
b->   if (_amount0 < _amount0Requested || _amount1 < _amount1Requested) {
        revert V3FactoryOwner__InsufficientFeesCollected();
      }
      
      // SNIP
    }

```

But have a close look at `collectProtocol` implementation:

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

On the marked lines (@->), the amount of fees transferred will always be 1 wei less than the requested amount to save gas (In case of complete withdrawal).

As a result, the condition on line `b` in `V3FactoryOwner` will trigger a revert whenever an MEV attempts to withdraw complete fees.

While MEV can avoid this scenario by never requesting 100% withdrawal, it's important to address this edge case at the smart contract level itself, facilitating easier implementation for MEVs to directly call `claimFees` with the maximum requested amount.

## Recommended Mitigation

The following modification streamlines the process for MEVs, allowing them to directly call claimFees with the maximum amount for withdrawal and also protect them in case of frontrunning from other bots:

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

### [L-02] Lack of Enforced deadline for signature can be exploited

## Description

The `UniStaker` contract offers six primary functionalities: Stake, StakeMore, AlterDelegatee, AlterBeneficiary, Withdraw, and ClaimReward.

Users have the option to sign their intent and delegate the execution of these functions to a third party.

However, none of the six type hashes include a parameter for a deadline. This means that once a user signs the intent, there is no time constraint binding the caller to execute it promptly. Consequently, the caller can delay the execution indefinitely and call it at any point in the future.

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

Consider the following Scenario:

1. Alice authorizes an `AlterBeneficiary` call by signing and passing it to Bob.
2. Bob delays calling the function for an extended period.
3. Later, Alice decides to change the `beneficiary` to Charles from Bob for upcoming rewards and executes the `AlterBeneficiary` function herself.
4. At this juncture, Bob exploits the signed transaction to claim the upcoming reward, potentially at a higher rate than the previous one.

Although Bob is typically considered a trusted party by Alice, the absence of enforced deadlines on signed transactions can pose a significant risk to the staker who signs them.

## Recommended Mitigation

Implementing a mechanism to enforce deadlines on signed transactions can mitigate this risk, ensuring timely execution and preventing potential exploitation scenarios.

### [L-03] First Depositor can inflate `rewardPerTokenAccumulatedCheckpoint` and steal reward

## Proof of Concept

The primary user who executes the `stake()` function with a mere 1 wei can significantly inflate the `rewardPerTokenAccumulatedCheckpoint`.

Consider the following scenario:

1. Alice initiates `stake()` with herself as the `beneficiary`, depositing 1 wei.
2. This updates `earningPower[Alice]` to 1 wei and sets `totalStaked = 1`.
3. The `_checkpointReward(Alice)` call assigns the `beneficiaryRewardPerTokenCheckpoint` using the returned value from `rewardPerTokenAccumulated()` for Alice.

```solidity

    function rewardPerTokenAccumulated() public view returns (uint256) {
      if (totalStaked == 0) return rewardPerTokenAccumulatedCheckpoint;

      return rewardPerTokenAccumulatedCheckpoint
@->     + (scaledRewardRate * (lastTimeRewardDistributed() - lastCheckpointTime)) / totalStaked;
    }

```

4. Ideally, this calculation divides `scaledRewardRate` (36 decimals) by `totalStaked` (assumed 18 decimals), resulting in an output of 18 decimals.

However, if a user initiates `stake()` with 1 wei, the `rewardPerTokenAccumulatedCheckpoint` can undergo significant inflation.

A similar finding was reported in the veToken Audit [here](https://code4rena.com/reports/2022-05-vetoken#m-23-baserewardpools-rewardpertokenstored-can-be-inflated-and-rewards-can-be-stolen) where the mechanism of Synthetic's StakingReward was exploited by Alex to illustrate its unfairness.

Even in cases where a user stakes `1e18` tokens initially (non-malicious behavior), it could result in unfairness upto three times the rewards in this edge case.

## Recommendation Mitigation

It is recommended to deposit at least 1e18 `UNI` Tokens initially.

### [NC-01] `setOwner` function in Factory contract will be permanently disabled

The `V3FactoryOwner` contract, being immutable and non-upgradable, is designated to assume control as the owner of the Uniswap V3 Factory contract. 

Consequently, `V3FactoryOwner` becomes the sole entity authorized to invoke owner-controlled functions within the `UniswapV3Factory`.

Owner-controlled functions in the `UniswapV3Factory` contract include:

1. `setOwner`: Used to update the owner.
2. `enableFeeAmount`: Used to enable fees.

However, the `V3FactoryOwner` contract only possesses the capability to call `FACTORY.enableFeeAmount`. It lacks the ability to call `FACTORY.setOwner` for potential future updates to the new owner contract.

In the event that an upgrade is deemed necessary, the only available option would be to modify the `admin` of the `V3FactoryOwner` to point to a different contract. However, this approach would incur additional gas overhead, as the call sequence would be:

`NewOwner` --> `V3FactoryOwner` --> `UniswapV3Factory`

as opposed to the more direct:

`NewOwner` --> `UniswapV3Factory`

## Recommended Mitigation

To ensure efficient upgrades in the future (Planned or unexpected), consider implementing a mechanism within the `V3FactoryOwner` contract to allow for the direct update of the owner contract in Uniswap V3 Factory contract. This would streamline the upgrade process and reduce potential gas overhead in unexpected/planned upgrade events.