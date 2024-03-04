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