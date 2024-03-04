## Title
Missing validation of Admin argument could lead to incorrect event
emission

## Impact
Because the `setAdmin` lacks input validation, the admin can be updated to the existing admin. Although such an update wouldn’t change the contract state, it would emit an event falsely indicating the admin had been changed.

## Proof of Concept
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/V3FactoryOwner.sol#L110C2-L115C4

```solidity
 function setAdmin(address _newAdmin) external {
    _revertIfNotAdmin();
    if (_newAdmin == address(0)) revert V3FactoryOwner__InvalidAddress();
    emit AdminSet(admin, _newAdmin);
    admin = _newAdmin;
  }
```

## Exploit scenario:
Bob has set up monitoring of the Admin change event to track transfers of the admin role. Alice, the current admin, calls `setAdmin` to update the admin to her address (not actually making a change). Bob is notified that the admin was changed but upon closer inspection discovers it was not.

## Tools Used
Manual review

## Recommended Mitigation Steps
Add a check ensuring that the _newAdmin argument does not equal the existing
admin