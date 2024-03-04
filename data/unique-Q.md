## \[L-01\] Keccak Constant values should used to immutable rather than constant

There is a difference between constant variables and immutable variables, and they should each be used in their appropriate contexts.

While it doesn’t save any gas because the compiler knows that developers often make this mistake, it’s still best to use the right tool for the task at hand.

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

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L102

&nbsp;

## \[L-02\] zero amount should be checked before calling a function or transferring

```
function withdraw(DepositIdentifier _depositId, uint256 _amount) external {
    Deposit storage deposit = deposits[_depositId];
    _revertIfNotDepositOwner(deposit, msg.sender);
    _withdraw(deposit, _depositId, _amount);  <@
    // 
  }
```

## Checking for zero amount can prevent users from accidentally initiating withdrawals with no effect. This can help ensure that the function behaves as expected and maintains consistency in the system.

&nbsp;https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L499

## \[N-03\] Showing the actual values of numbers in NatSpec comments makes checking and reading code easier

```diff
- uint256 public constant SCALE_FACTOR = 1e36; 

+ uint256 public constant SCALE_FACTOR = 1e36; // 1_000_000_000_000_000_000_000_000_000_000_000_000
```

&nbsp;https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L134

## \[N-04\] Use a single file for all system-wide constants

There are many addresses and constants used in the system. It is recommended to put the most used ones in one file (for example constants.sol, use inheritance to access these values).

This will help with readability and easier maintenance for future changes. This also helps with any issues, as some of these hard-coded values are admin addresses.

constants.sol  
Use and import this file in contracts that require access to these values. This is just a suggestion, in some use cases this may result in higher gas usage in the distribution.

## \[N-05\] Floating pragma

Description  
Contracts should be deployed with the same compiler version and flags that they have been tested with thoroughly. Locking the pragma helps to ensure that contracts do not accidentally get deployed using, for example, an outdated compiler version that might introduce bugs that affect the contract system negatively.  
https://swcregistry.io/docs/SWC-103

```
File : src/interfaces/IERC20Delegates.sol
pragma solidity ^0.8.23;

File: src/interfaces/INotifiableRewardReceiver.sol
pragma solidity ^0.8.23;

File: src/UniStaker.sol
pragma solidity 0.8.23;
```
## [S-01] Generate perfect code headers every time

Description:  
I recommend using header for Solidity code layout and readability

/*´:°•.°+.*•´.*:˚.°*.˚•´.°:°•.°•.*•´.*:˚.°*.˚•´.°:°•.°+.*•´.*:*/  
/*                 CONSTANTS                                  */  
/*.•°:°.´+˚.*°.˚:*.´•*.+°.•°:´*.´•*.•°.•°:°.´:•˚°.*°.˚:*.´+°.•*/

&nbsp;