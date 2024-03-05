## Findings Summary

| ID  | Description | Severity |
| --- | --- | --- |
| [L-01](#l-01-potential-dust-loss-in-alterbeneficiary-function) | Potential Dust Loss in `alterBeneficiary` Function | Low |
| [L-02](#l-02-event-emission-without-actual-alteration-in-alterbeneficiary-and-alterdelegatee-functions) | Event Emission Without Actual Alteration in `alterBeneficiary` and `alterDelegatee` Functions | Low |
| [N-01](#n-01-redundant-parameter-passing-in-some-of-onbehalf-functions) | Redundant Parameter Passing in Some of (OnBehalf) Functions | Non-Critical |
| [N-02](#n-02-event-parameter-name-correction) | Event Parameter Name Correction | Non-Critical |
| [N-03](#n-03-redundant-condition-checks-in-lasttimerewarddistributed-function) | Redundant Condition Checks in `lastTimeRewardDistributed` Function | Non-Critical |
| [N-04](#n-04-addition-of-deposit-id-retrieval-method) | Addition of Deposit ID Retrieval Method | Non-Critical |
| [N-05](#n-05-public-getters-for-contract-variables) | Public Getters for Contract Variables | Non-Critical |

## [L-01] Potential Dust Loss in `alterBeneficiary` Function

The `alterBeneficiary` function in the UniStaker contract allows the alteration of the beneficiary address for a deposit. However, this function can result in potential dust loss for the user if called multiple times with the same beneficiary address in different times.

This issue arises due to the internal logic of the `_alterBeneficiary` function, which recalculates the reward checkpoints (using `_checkpointReward` ) and updates the beneficiary's earning power each time `alterBeneficiary` is called, even if the new beneficiary address is the same as the current one.

[UniStaker.sol#L711-L717](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol##L710-L716)

```solidity
src/UniStaker.sol:
  711:     _checkpointReward(deposit.beneficiary);
  712:     earningPower[deposit.beneficiary] -= deposit.balance;
  713: 
  714:     _checkpointReward(_newBeneficiary);
  715:     emit BeneficiaryAltered(_depositId, deposit.beneficiary, _newBeneficiary);
  716:     deposit.beneficiary = _newBeneficiary;
  717:     earningPower[_newBeneficiary] += deposit.balance;
```

### Recommendation

- Review the necessity of recalculating the reward checkpoints and updating earning power when the beneficiary address remains unchanged.
  
- Implement a condition to skip the recalculation process in `_alterBeneficiary` if the new beneficiary address is the same as the current one, reducing unnecessary gas consumption and potential dust loss for users.
  

This issue should be addressed to prevent users from incurring unnecessary gas costs and potential loss of dust rewards when altering the beneficiary address multiple times without actually changing it.

## [L-02] Event Emission Without Actual Alteration in `alterBeneficiary` and `alterDelegatee` Functions

The `alterBeneficiary` and `alterDelegatee` functions in the UniStaker contract emit an event indicating an alteration even when the provided new address matches the current one, leading to potentially misleading event emissions.

Here's an example of the event emitted:

```solidity
event DelegateeAltered(
    DepositIdentifier indexed depositId, address oldDelegatee, address newDelegatee
);
```

However, if the new address provided is identical to the current address in deposit, no actual alteration occurs. Despite this, the functions emit an event suggesting a change, which could potentially confuse users or external systems monitoring these events.

### Recommendation

- Implement a check within the `alterBeneficiary` and `alterDelegatee` functions to verify if the provided new address is different from the current one before emitting the alteration event. If the new address matches the current one, avoid emitting the event to prevent potentially misleading event emissions and ensure clarity in contract events.

This enhancement will improve the integrity and clarity of event emissions, providing users and external systems with accurate information about changes in beneficiary and delegatee addresses associated with deposits.

## [N-01] Redundant Parameter Passing in Some of (OnBehalf) Functions

**NOTE**: This report focuses on the `stakeMoreOnBehalf` function, but similar issues are present in multiple other functions that use `depositId` along with `depositor` address. The full list of functions affected includes `stakeMoreOnBehalf`, `alterDelegateeOnBehalf`, `alterBeneficiaryOnBehalf`, and `withdrawOnBehalf`.

The `stakeMoreOnBehalf` function in the UniStaker contract contains a redundant parameter (`_depositor`) that is not required for the signature validation process. This redundancy introduces unnecessary complexity and potential confusion for developers interacting with the function.

The unnecessary parameter is evident in the function's implementation:
[UniStaker.sol#L382-L402](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L382-L402)

```solidity
src/UniStaker.sol:
  382:   function stakeMoreOnBehalf(
  383:     DepositIdentifier _depositId,
  384:     uint256 _amount,
  385:     address _depositor,
  386:     bytes memory _signature
  387:   ) external {
  388:     Deposit storage deposit = deposits[_depositId];
  389:     _revertIfNotDepositOwner(deposit, _depositor);
  390: 
  391:     _revertIfSignatureIsNotValidNow(
  392:       _depositor,
  393:       _hashTypedDataV4(
  394:         keccak256(
  395:           abi.encode(STAKE_MORE_TYPEHASH, _depositId, _amount, _depositor, _useNonce(_depositor))
  396:         )
  397:       ),
  398:       _signature
  399:     );
  400: 
  401:     _stakeMore(deposit, _depositId, _amount);
  402:   }
```

The presence of `_depositor` as a function parameter suggests that the caller needs to specify the depositor's address, even though it can be utilized within the function using `deposit.owner`.

Removing the redundant parameter and relying solely on the signer address (`deposit.owner`) for signature verification would simplify the function interface and improve its clarity.

Additionally, it is worth noting that the `STAKE_MORE_TYPEHASH` used for generating the message hash has the `depositor` param, while verifying the signature with the signer's address may be sufficient, but it is not necessary if the signer's address is always is the owner address.

### Recommendation

- Remove redundant `address _depositor` param from function params.
  
- Remove redundant check for the deposit owner `_revertIfNotDepositOwner(deposit, _depositor);`, since the `deposit.owner` itself will be used for validating the signature in the next step.
  
- Use the `deposit.owner` as the message signer address in `_revertIfSignatureIsNotValidNow`
  
- Remove the deposit address param from `STAKE_MORE_TYPEHASH` and from the message hash in `_revertIfSignatureIsNotValidNow`
  

Here is the diff of the full recommendation for stake on behalf:

```diff
   bytes32 public constant STAKE_MORE_TYPEHASH =
-    keccak256("StakeMore(uint256 depositId,uint256 amount,address depositor,uint256 nonce)");
+    keccak256("StakeMore(uint256 depositId,uint256 amount,uint256 nonce)");

   function stakeMoreOnBehalf(
     DepositIdentifier _depositId,
     uint256 _amount,
-    address _depositor,
     bytes memory _signature
   ) external {
     Deposit storage deposit = deposits[_depositId];
-    _revertIfNotDepositOwner(deposit, _depositor);

     _revertIfSignatureIsNotValidNow(
-      depositor,
+      deposit.owner,
       _hashTypedDataV4(
         keccak256(
-          abi.encode(STAKE_MORE_TYPEHASH, _depositId, _amount, _depositor, _useNonce(_depositor))
+          abi.encode(STAKE_MORE_TYPEHASH, _depositId, _amount, _useNonce(_depositor))
         )
       ),
       _signature
     );

     _stakeMore(deposit, _depositId, _amount);
   }
```

## [N-02] Event Parameter Name Correction

A typo was identified in the event parameter names of the `AdminSet` event. The parameter `oldAmin` should be corrected to `oldAdmin` for consistency and accuracy. This correction ensures clarity and coherence in event logs, facilitating better understanding and interpretation of contract events by developers and users alike.

The issue can be observed in the following event declaration:

[V3FactoryOwner.sol#L48](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/491c7f63e5799d95a181be4a978b2f074dc219a5/src/V3FactoryOwner.sol#L48)

```solidity
src/V3FactoryOwner.sol:
  47:   /// @notice Emitted when the existing admin designates a new address as the admin.
  48:   event AdminSet(address indexed oldAmin, address indexed newAdmin);
```

To maintain consistency and accuracy, it is recommended to correct the parameter name from `oldAmin` to `oldAdmin` in the `AdminSet` event declaration. This small adjustment will improve the readability and comprehensibility of event logs, ensuring a smoother development and debugging experience for stakeholders involved in the contract ecosystem.

## [N-03] Redundant Condition Checks in `lastTimeRewardDistributed` Function

The `lastTimeRewardDistributed` function in the UniStaker contract contains a redundant conditional check that could be simplified for improved clarity and efficiency. This redundancy does not affect the correctness of the function but could lead to confusion for developers reviewing the code.

The unnecessary condition check is evident in the function's implementation:

[UniStaker.sol#L220-L223](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L220-L223)

```solidity
src/UniStaker.sol:
  219    /// @return Timestamp representing the last time at which rewards have been distributed.
  220:   function lastTimeRewardDistributed() public view returns (uint256) {
  221:     if (rewardEndTime <= block.timestamp) return rewardEndTime;
  222:     else return block.timestamp;
  223:   }
```

The condition `rewardEndTime <= block.timestamp` checks if the reward end time is less than or equal to the current block timestamp. However, if `rewardEndTime` equals `block.timestamp`, the condition still holds true, making the check redundant.

Furthermore, since the `if` block contains a return statement, the `else` block becomes redundant too.

### Recommendation

- Remove the redundant condition check and replace it with `if (rewardEndTime < block.timestamp)` to simplify the logic while maintaining the intended functionality.
- Remove the `else` block, as it is unnecessary due to the `return` statement within the `if` block.

This adjustment eliminates the redundancy in the conditional check without altering the behavior of the function.

## [N-04] Addition of Deposit ID Retrieval Method

Consider adding a method in the UniStaker contract that enables users or contracts to retrieve deposit IDs associated with a particular user address. This enhancement would improve usability and flexibility, allowing for smoother interactions and better management of staking activities for users and integrators.

## [N-05] Public Getters for Contract

It is recommended to implement public getter functions for the `token` and `delegatee` variables in the `DelegationSurrogate` contract. Exposing these constants to external visibility would enhance transparency and accessibility, facilitating easier integration with other smart contracts and interfaces.