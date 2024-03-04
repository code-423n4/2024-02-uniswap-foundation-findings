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
## [L-02] Surrogate Deployment Event
The SurrogateDeployed event emits the delegatee and surrogate addresses but does not include the depositId. Including the depositId could provide a clearer traceability between deposits and their corresponding surrogate contracts.

## [L-03] StakeWithdrawn event does not include the owner.
`emit StakeWithdrawn(_depositId, _amount, deposit.balance);`

## [Nc-01] Magic Number
The SCALE_FACTOR is set to 1e36 without a comment explaining why this specific number was chosen. While it's a common practice to use large scale factors to prevent rounding errors, a comment could help clarify the choice.