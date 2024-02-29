## A. Redundant Ownership Check in `stakeMoreOnBehalf` Function

[#L382-L402](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/UniStaker.sol#L382-L402)

The `stakeMoreOnBehalf` function allows a user to stake additional tokens to an existing deposit on behalf of another user, authorized by the signer's signature. However, the function unnecessarily verifies the ownership of the deposit by the caller. Since the stake operation is already validated by the user's signature, the ownership check becomes redundant and does not contribute to the function's security or functionality.

Recommendation:
The unnecessary ownership check should be removed from the `stakeMoreOnBehalf` function