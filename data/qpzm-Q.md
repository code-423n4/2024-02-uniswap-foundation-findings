## 1. One signature argument can be removed.
In `claimRewardOnBehalf`, `_beneficiary` is unnecessary in the hash. It will be given as a function argument and we can check whether the signature signer is equal to the function argument `_beneficiary`.
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/cde876f5eed60eb2df4104cf031ddc21b1f538b9/src/UniStaker.sol#L544-L553

It is the same for all other onBehalf functions.
`_depositor` is the same unnecessary parameter in `stakeOnBehalf`, `stakeMoreOnBehalf`, `alterDelegateeOnBehalf`, `alterBeneficiaryOnBehalf`, `withdrawOnBehalf`.
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/cde876f5eed60eb2df4104cf031ddc21b1f538b9/src/UniStaker.sol#L102-L121
