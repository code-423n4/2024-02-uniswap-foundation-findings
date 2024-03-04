## [L-01] Always emit events before external call
```  
_stakeTokenSafeTransferFrom(_depositor, address(_surrogate), _amount);
    emit StakeDeposited(_depositor, _depositId, _amount, _amount);
    emit BeneficiaryAltered(_depositId, address(0), _beneficiary);
    emit DelegateeAltered(_depositId, address(0), _delegatee);
```
```
  _stakeTokenSafeTransferFrom(_depositor, address(_surrogate), _amount);
    emit StakeDeposited(_depositor, _depositId, _amount, _amount);
    emit BeneficiaryAltered(_depositId, address(0), _beneficiary);
    emit DelegateeAltered(_depositId, address(0), _delegatee);
```

### [L-02] Inconsistent naming conventions: The mix of camelCase and snake_case in variable and function names (e.g., rewardToken, govToken, rewardsNotifier, accumulateDeposits, ghost_stakeSum, ghost_stakeWithdrawn) can be confusing and is not consistent with Solidity style recommendations.