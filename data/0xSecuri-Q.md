## [NC-01] Incorrect documentation for `_signature` parameter

### Description

In the `UniStaker.sol` contract, there's a documentation issue with the `_signature` parameter in multiple functions. The current documentation states: "Signature of the user authorizing this stake." This description is inaccurate as it doesn't correspond to the operations being performed by the functions. The affected functions are as follows:

1. `alterDelegateeOnBehalf`
2. `withdrawOnBehalf`
3. `alterBeneficiaryOnBehalf`

### Recommendation

The documentation for the `_signature` parameter should be updated to reflect the correct operation for each function. It should be something like: "Signature of the user authorizing this alteration."

[View the code snippet for `alterDelegateeOnBehalf`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L421)

[View the code snippet for `withdrawOnBehalf`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L510)

[View the code snippet for `alterBeneficiaryOnBehalf`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L464)

## [NC-02] Redundant word in comment for `claimFees` function

### Description

In the `V3FactoryOwner.sol` contract, there's a redundant word in the inline documentation for the `claimFees` function. The comment states: "the the payout amount" The repetition of "the" is unnecessary and should be removed for clarity.

### Recommendation

The comment should be updated to remove the redundant word, resulting in: "buy the pool fees for less than they are valued by paying the payout amount of the payout token."

[View the code snippet in context](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L170)
