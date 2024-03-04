https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L110
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L113

`V3FactoryOwner.setAdmin` does not check if `_newAdmin` is same as existing admin. This would cause it to wrongly emit an event for `AdminSet(admin, _newAdmin)` when no actual
 change in admin access has occurred. 
It is recommended that `_newAdmin` is checked for if it is current admin and revert if so.

Alternatively, a two-step process should be used in this scenario to change a critical access such as the admin. 