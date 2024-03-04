# 1.
## Possible loss of `adminRole`, can disable important protocol `functionalities`.
In the `_setAdmin` function the process of setting the admin is by the current admin passing the address of the new admin which is then set as admin but this method is  not very ideal as any mistake(typographical error) by current admin when passing the address of the new admin will lead to the loss of the `adminRole` which can deny access to important functionalities of the protocol, like  `_setFeeProtocol`, will be disabled for some V3 pools controlled by the admin. `setPayoutAmount` will also be uncallable. `enableFeeAmount` will also be uncallable, leading to series of issues in the protocol, as `fees` cannot be updated   with respect to current trend in the value of `tokens` in the pool, the `payOutAmount` being obsolete could also lead to issues like loss of funds for the protocol if the value of the payout token increases or users could receive very small reward if it decreases.
```solidity
 function setAdmin(address _newAdmin) external {
    _revertIfNotAdmin();
    if (_newAdmin == address(0)) revert V3FactoryOwner__InvalidAddress();
    emit AdminSet(admin, _newAdmin);
    admin = _newAdmin;
  }
```
This can be mitigated by using the openZeppelin `ownable2steps` contract for setting the admin role or by making the new admin accept the admin role before it becomes effective.

# 2.
## Stakers can lose their profit as `PayOutAmount` can be set to zero
The payout amount is the amount that will be distributed to users but there is no minimum value for this variable which makes it possible that if the admin can mistakenly set payout amount to zero making users, receive no payouts after rewards has been claimed, making users(depositors) to lose their profit.
```solidity
function setPayoutAmount(uint256 _newPayoutAmount) external {
    _revertIfNotAdmin();
    if (_newPayoutAmount == 0) revert V3FactoryOwner__InvalidPayoutAmount();
    emit PayoutAmountSet(payoutAmount, _newPayoutAmount);
    payoutAmount = _newPayoutAmount;
  }
```
This can be mitigated by adding a minimum payout `variable` and checking if the new payout amount is `<=` the minimum payout amount before setting the new value.