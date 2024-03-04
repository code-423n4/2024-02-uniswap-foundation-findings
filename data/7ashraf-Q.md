# Quality Assurance Report
## Summary 
The codebase requires improvements such as validating return values, implementing deletion mechanisms, and ensuring proper error handling for enhanced reliability and user experience.

| Issue Number | Issue Title                                       | Number of Instances |
|--------------|---------------------------------------------------|---------------------|
| L-01         | Check approve return value                        | 1                   |
| L-02         | Add a deletion mechanism after disabling          | 1                   |
| L-03         | Check if Deposit exists                           | 1                   |
| L-04         | Check if `amount` is greater than 0               | 1                   |
| N-01         | Add `revertIfNotNotifier` modifier for consistency| 1                   |
| N-02         | Consider deleting the user after withdrawing all of his balance | 1                   |
| N-03         | Pass parameter `_newPayoutAmount` in revert message | 1                   |

## [L-01] Check `approve()` return value
### Instance
* [DelegationSurrogate.sol #27](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/DelegationSurrogate.sol#L27)
```solidity
    _token.approve(msg.sender, type(uint256).max);

```

## [L-02] Add a deletion mechanism after disabling
### Instances
* [UniStaker.sol #212](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L212)
```solidity
    isRewardNotifier[_rewardNotifier] = _isEnabled;

```


## [L-03] Check if Deposit exists
The function will revert if the deposit does not exist due to owner check, but an inaccurate error message will be emitted. It is recommended to explicitly check if the deposit exists first before processing.
### Instances
* [UniStaker.sol #343](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L343)
```solidity
    Deposit storage deposit = deposits[_depositId];

```



## [L-04] Check if `amount` is greater than 0
### Instances
* [UniStaker.sol #651](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L651)
```solidity
    totalStaked += _amount;

```


## [N-01]Add `revertIfNotNotifier` modifier for consistency
The code follows consistency in checks by calling functions like `UniStaker::_revertIfNotDepositOwner` and `UniStaker::_revertIfAddressZero`. It is more recommended to follow the code consistency and add `revertIfNotNotifier` instead of explicit check by if statement
### Instances
* [UniStaker.sol #571](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L571)
```solidity
    if (!isRewardNotifier[msg.sender]) revert UniStaker__Unauthorized("not notifier", msg.sender);

```

## [N-02] Consider deleting the user after withdrawing all of his balance
### Instance
* [UniStaker.sol #723](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L723C1-L735C4)
```solidity
  function _withdraw(Deposit storage deposit, DepositIdentifier _depositId, uint256 _amount)
    internal
  {
    _checkpointGlobalReward();
    _checkpointReward(deposit.beneficiary);

    deposit.balance -= _amount; // overflow prevents withdrawing more than balance
    totalStaked -= _amount;
    depositorTotalStaked[deposit.owner] -= _amount;
    earningPower[deposit.beneficiary] -= _amount;
    //@audit delete the user if withdraws all of his balance
    _stakeTokenSafeTransferFrom(address(surrogates[deposit.delegatee]), deposit.owner, _amount);
    emit StakeWithdrawn(_depositId, _amount, deposit.balance);
  }
```

## [N-03] Pass parameter `_newPayoutAmount` in revert message
### Instances
* [V3FactoryOwner.sol #121](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L121)
```solidity
    if (_newPayoutAmount == 0) revert V3FactoryOwner__InvalidPayoutAmount();

```





















