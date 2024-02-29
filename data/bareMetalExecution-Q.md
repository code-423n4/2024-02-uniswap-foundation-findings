## A. Redundant Ownership Check in `stakeMoreOnBehalf` Function

[#L382-L402](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/UniStaker.sol#L382-L402)

The `stakeMoreOnBehalf` function allows a user to stake additional tokens to an existing deposit on behalf of another user, authorized by the signer's signature. However, the function unnecessarily verifies the ownership of the deposit by the caller. Since the stake operation is already validated by the user's signature, the ownership check becomes redundant and does not contribute to the function's security or functionality.

Recommendation:
The unnecessary ownership check should be removed from the `stakeMoreOnBehalf` function


## B.Lack Of Reentrancy Guards On External Functions
[#L181-L205](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/V3FactoryOwner.sol#L181-L205)
[#L499-L503](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/UniStaker.sol#L499-L503)
[#L536-L538](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/UniStaker.sol#L536-L538)
The following external functions contain function calls (e.g. `safeTransferFrom`, `safeTransfer`) that pass control to external contracts.Thus, it might allow anY malicious external contract to re-enter to the contract.

Recommendation:
It is recommended to follow the good security practices and apply necessary reentrancy prevention by utilizing the `nonReentrant` modifier from Openzeppelin Library to block possible re-entrancy.