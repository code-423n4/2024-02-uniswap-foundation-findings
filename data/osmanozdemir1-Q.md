### Summary

* \[N-01\] Developer comments above the `UniStaker::stakeOnBehalf` is misleading
    

---

### \[N-01\] Developer comments above the `UniStaker::stakeOnBehalf` is misleading

Users can stake UNI tokens in this protocol, or another user can stake on behalf of other users via `stakeOnBehalf` function. The developer comment above the `stakeOnBehalf` [here](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/UniStaker.sol#L306C22-L306C70) is:

```solidity
  /// @notice Stake tokens to a new deposit on behalf of a user, using a signature to validate the
  /// user's intent. The caller must pre-approve the staking contract to spend at least the 
  /// would-be staked amount of the token.
```

"*The caller must pre-approve the staking contract*" sentence is incorrect since the users themselves must pre-approve the staking contract, not the caller. [User is the depositor](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/UniStaker.sol#L333C25-L333C35) in this function and the transfer is made from the user's account not the caller's account.

I submitted this issue as QA assuming the code is correct but the comment is wrong. However, if the comment is correct, it means the expected behaviour is transferring tokens from the caller and this is more severe bug.