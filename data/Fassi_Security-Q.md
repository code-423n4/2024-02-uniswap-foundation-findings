## 1. Function `_AlterDelegatee` can be called on the same address as the previous Delegatee.
 
 ## Description
In the `_AlterDelegatee` function, it is possible to set `_NewSurrogate` to the same address as `_OldSurrogate`. This means that you could provide the same address as the current one that is being used. 
```javascript
 function _alterDelegatee(
    Deposit storage deposit,
    DepositIdentifier _depositId,
    address _newDelegatee
  ) internal {
    _revertIfAddressZero(_newDelegatee);
    DelegationSurrogate _oldSurrogate = surrogates[deposit.delegatee];
    emit DelegateeAltered(_depositId, deposit.delegatee, _newDelegatee);
    deposit.delegatee = _newDelegatee;
    DelegationSurrogate _newSurrogate = _fetchOrDeploySurrogate(_newDelegatee);
    _stakeTokenSafeTransferFrom(address(_oldSurrogate), address(_newSurrogate), deposit.balance);
  }
```
## Impact
This can cause for great confusion and later down the line when the code develops and gets more complex it could open up doors to even more issues.
## Tools Used
Manual Review
## Recommended Mitigation Steps
Add a require statement:
```diff
 function _alterDelegatee(
    Deposit storage deposit,
    DepositIdentifier _depositId,
    address _newDelegatee
  ) internal {
    _revertIfAddressZero(_newDelegatee);
+ require(_oldSurrogate != _newSurrogate, "Old and new surrogates must be different");
    DelegationSurrogate _oldSurrogate = surrogates[deposit.delegatee];
    emit DelegateeAltered(_depositId, deposit.delegatee, _newDelegatee);
    deposit.delegatee = _newDelegatee;
    DelegationSurrogate _newSurrogate = _fetchOrDeploySurrogate(_newDelegatee);
    _stakeTokenSafeTransferFrom(address(_oldSurrogate), address(_newSurrogate), deposit.balance);
  }


```

## 2. Function `_AlterBeneficiary` can be called on the same address as the previous Beneficiary

## Description
In the `_AlterBeneficiary` function `_NewBeneficiary` is not checked to be different than the old Beneficiary address.

```javascript
 function _alterBeneficiary(
    Deposit storage deposit,
    DepositIdentifier _depositId,
    address _newBeneficiary
  ) internal {
    _revertIfAddressZero(_newBeneficiary);
    _checkpointGlobalReward();
    _checkpointReward(deposit.beneficiary);
    earningPower[deposit.beneficiary] -= deposit.balance;

    _checkpointReward(_newBeneficiary);
    emit BeneficiaryAltered(_depositId, deposit.beneficiary, _newBeneficiary);
    deposit.beneficiary = _newBeneficiary;
    earningPower[_newBeneficiary] += deposit.balance;
  }

```
## Impact
This means that a user could set the `_NewBeneficiary` to be the same as the old one, this should not be possible and could cause unwanted behaviour 

## Tools Used
Manual review

## Recommended Mitigation Steps
Add a require statement:

```diff
 function _alterBeneficiary(
    Deposit storage deposit,
    DepositIdentifier _depositId,
    address _newBeneficiary
  ) internal {
    _revertIfAddressZero(_newBeneficiary);
+ require(_newBeneficiary != deposit.beneficiary, "New beneficiary must be different from the current one");
    _checkpointGlobalReward();
    _checkpointReward(deposit.beneficiary);
    earningPower[deposit.beneficiary] -= deposit.balance;

    _checkpointReward(_newBeneficiary);
    emit BeneficiaryAltered(_depositId, deposit.beneficiary, _newBeneficiary);
    deposit.beneficiary = _newBeneficiary;
    earningPower[_newBeneficiary] += deposit.balance;
  }


```

## 3. Function `CollectProtocol` can be called with both parameters `Amount0Requested` & `Amount1Requested` set to 0.

## Description
It is possible to call `CollectProtocol` with both parameters `Amount0Requested` & `Amount1Requested` set to 0, and the call will still be successfully made.

```javascript
function collectProtocol(address recipient, uint128 amount0Requested, uint128 amount1Requested)
    external
    returns (uint128 amount0, uint128 amount1);

```
As we can see in this function there is no check to make sure `amount0Requested` & `amount1Requested` are not both 0.

## Impact
`CollectProtocol` gets used in other functions, so the team has to make sure that there is no way this can be called with both parameters being equal to 0.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Make sure to implement a check that checks if 1 or the other is 0 or if both are 0 (depending on the design choice)

```diff
function collectProtocol(address recipient, uint128 amount0Requested, uint128 amount1Requested)
    external
    returns (uint128 amount0, uint128 amount1);
+ require(amount0Requested > 0 || amount1Requested > 0, "Both requested amounts cannot be 0");
```
 
## 4. Function `_stake` can be called with `amount` 0 

## Description
in function `_stake` it is possible to call it with `amount = 0` because of the lack of 0 checks implemented. This can cause unnecessary confusion and could further down the line cause for unnecessary complications 

```javascript
 function _stake(address _depositor, uint256 _amount, address _delegatee, address _beneficiary)
    internal
    returns (DepositIdentifier _depositId)
  {
    _revertIfAddressZero(_delegatee);
    _revertIfAddressZero(_beneficiary);
    _checkpointGlobalReward();
    _checkpointReward(_beneficiary);
  //..Ommitted code
```
## Impact
A user can stake with amount 0, this should not be able to pass and could lead to complex code bugs later down the line.

## Tools Used
Manual Review

## Recommended Mitigation Steps
add a 0 check

```diff
 function _stake(address _depositor, uint256 _amount, address _delegatee, address _beneficiary)
    internal
    returns (DepositIdentifier _depositId)
  {
    _revertIfAddressZero(_delegatee);
    _revertIfAddressZero(_beneficiary);
+require(_amount > 0, "Staked amount should be greater than 0")
    _checkpointGlobalReward();
    _checkpointReward(_beneficiary);
  //..Ommitted code
```


## 5. Function `_withdraw` can be called with `amount` 0

## Description
Since there is no check to ensure the amount withdrawn is > 0, a user can call the function `_withdraw` with `amount = 0` without failing. See the following code.
```javascript
/// @notice Internal convenience method which withdraws the stake from an existing deposit.
  /// @dev This method must only be called after proper authorization has been completed.
  /// @dev See public withdraw methods for additional documentation.
  function _withdraw(Deposit storage deposit, DepositIdentifier _depositId, uint256 _amount)
    internal
  {
    _checkpointGlobalReward();
    _checkpointReward(deposit.beneficiary);

    deposit.balance -= _amount; // overflow prevents withdrawing more than balance
    totalStaked -= _amount;
    depositorTotalStaked[deposit.owner] -= _amount;
    earningPower[deposit.beneficiary] -= _amount;
    _stakeTokenSafeTransferFrom(address(surrogates[deposit.delegatee]), deposit.owner, _amount);
    emit StakeWithdrawn(_depositId, _amount, deposit.balance);
  }

```
## Impact
The code is not supposed to run when an amount being requested to withdraw is 0. This will cause unneccesarry problems 

## Tools Used
Manual Review

## Recommended Mitigation Steps
Make sure to check that `amount > 0`

```diff
function _withdraw(Deposit storage deposit, DepositIdentifier _depositId, uint256 _amount)
    internal
  {
+require(_amount > 0, "Invalid amount)
    _checkpointGlobalReward();
    _checkpointReward(deposit.beneficiary);

//..Ommitted code
```

## 6. Function `feeAmountTickSpacing` is declared in the `IUniswapV3FactoryOwnerActions.sol` but is not used anywhere in the contract. 

## Description
In contract `IUniswapV3FactoryOwnerActions.sol` function `feeAmountTickSpacing` is declared but not used anywhere else in the code. If this is intended not to be used consider removing this function.

```
  /// @notice Returns the tick spacing for a given fee amount, if enabled, or 0 if not enabled
  /// @dev A fee amount can never be removed, so this value should be hard coded or cached in the
  /// calling context
  /// @param fee The enabled fee, denominated in hundredths of a bip. Returns 0 in case of unenabled
  /// fee
  /// @return The tick spacing
  function feeAmountTickSpacing(uint24 fee) external view returns (int24);
}
```
## Impact
As you can see it is called in the Interface but nowhere else in the code is it being used.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Either remove it (if it is not being used) or implement it in the code.

## 7. When using Multicall with different functions, it can become complicated and result in complex situations.
## Description

`Unistaker.sol` inherits [Multicall](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Multicall.sol) , which allows a user to perform multiple function calls in 1 transaction. This results in efficiency and other beneficial outcomes. However, calling Multicall on relay-like functions can cause issues. This would become a problem, especially if we were to batch calls such as StakeOnBehalf, StakemoreOnBehalf, and `AlterDelegateeOnBehalf`. These functions perform several call data validations and can ultimately lead to complex situations where a user malicious user might be able to find a vulnerability. 

As Openzeppelin states in their [Multicall](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Multicall.sol) contract:

```
/**
 * @dev Provides a function to batch together multiple calls in a single external call.
 *
 * Consider any assumption about calldata validation performed by the sender may be violated if it's not especially
 * careful about sending transactions invoking {multicall}. For example, a relay address that filters function
 * selectors won't filter calls nested within a {multicall} operation.
 *
 //..Ommitted Code
```
## Impact
Impact could be huge on this one, especially if the code keeps on getting more complex and there are no restraints on the user regarding which functions can be called using `Multicall`
## Tools Used
Manual Review

## Recommended Mitigation Steps
Make sure to treat `Multicall` with caution especially considering it can be used with many functions that are not directly called by the user but rather called on behalf of the user, which then call other functions that perform call data validation. Consider adding limitations on the use of `Multicall` aswell.