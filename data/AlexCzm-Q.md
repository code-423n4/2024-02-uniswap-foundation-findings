
### L-01 In the way nonce is used in the protocol, `onBehalf` functions will revert with `UniStaker__InvalidSignature` error when signatures are not used (tx executed) in the same order signatures were generated for (or, better said, in the ascending order of nonce used to generate the signature). 

Let's follow next example.
- a depositor Dan creates 2 signatures to allow Alice and Bob [to stake on behalf](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/UniStaker.sol#L315-L334) of Dan. Dan uses nonce 0 to generate the Alice's signature and nonce 1 to generate Bob's signature.
- Bob calls `stakeOnBehalf` first. because the nonce used to generate his signature is different than the onchain nonce his tx is reverted. He must wait for Alice's tx first.
Since there are 6 `onBehalf` functions things can get easily out of hand, forcing depositor to sign one intention at a time.  

An alternative to this nonce mechanismn  (incrementing nonce) would be to invalidate the nonce once it's used (eg. set a bit in a bitmap for each corresponding nonce used).

### L-02 Users who claim entire protocol fee accrued to the pool will have their tx reverted

For gas savings, UniswapPool does [not clear](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L857) (and [here](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L862)) the accumulated protocolFees. 
Users who calls `claimFees` to claim the entire pool's protocol fee will have their tx reverted due to [this](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/V3FactoryOwner.sol#L193) check.
```solidity
    // Protect the caller from receiving less than requested. See `collectProtocol` for context.
    if (_amount0 < _amount0Requested || _amount1 < _amount1Requested) {
      revert V3FactoryOwner__InsufficientFeesCollected();
    }
```
Consider ignoring the 1 wei substraction made in UniswapPool and either
- request 1 wei less when calling `_pool.collectProtocol`
```solidity 
_pool.collectProtocol(_recipient, _amount0Requested -1, _amount1Requested -1);
```
- subtract 1 wei when checking the requested amount and received amount
```solidity
    if (_amount0 < _amount0Requested - 1 || _amount1 < _amount1Requested - 1) {
      revert V3FactoryOwner__InsufficientFeesCollected();
    }
```

### L-03 Approvals are not cleared when a delegatee is changed
Deposit owner can choose to change the address to which he delegates his governance power by calling `alterDelegatee`. A surrogate contract is [deployed](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/491c7f63e5799d95a181be4a978b2f074dc219a5/src/UniStaker.sol#L609-L612) (or an existing one is fetched) for the new delegatee. The new surrogate contract [approve](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/491c7f63e5799d95a181be4a978b2f074dc219a5/src/DelegationSurrogate.sol#L27) UniStaker as spender. But the approval from old surrogate to UniStaker contract is not cleared. This may became problematic in case of using a different UniStaker in the future. 

Consider resetting approval when transferring stakeToken from one surrogate to another.
