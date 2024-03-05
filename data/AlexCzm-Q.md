
### L-01 In the way nonce is used in the protocol, `onBehalf` functions will revert with `UniStaker__InvalidSignature` error when signatures are not used (tx executed) in the same order signatures were generated for (or, better said, in the ascending order of nonce used to generate the signature). 

Let's follow next example.
- a depositor Dan creates 2 signatures to allow Alice and Bob [to stake on behalf](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/UniStaker.sol#L315-L334) of Dan. Dan uses nonce 0 to generate the Alice's signature and nonce 1 to generate Bob's signature.
- Bob calls `stakeOnBehalf` first. because the nonce used to generate his signature is different than the onchain nonce his tx is reverted. He must wait for Alice's tx first.
Since there are 6 `onBehalf` functions things can get easily out of hand, forcing depositor to sign one intention at a time.  

An alternative to this nonce mechanismn  (incrementing nonce) would be to have a mapping and invalidate the nonce once it's used (eg. set a bit in a bitmap for each corresponding nonce used).

### L-02 Users who claim entire protocol fee accrued in the pool will have their tx reverted

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
- or subtract 1 wei when checking the requested amount and received amount
```solidity
    if (_amount0 < _amount0Requested - 1 || _amount1 < _amount1Requested - 1) {
      revert V3FactoryOwner__InsufficientFeesCollected();
    }
```

### L-03 Approvals are not cleared when a delegatee is changed
Deposit owner can choose to change the address to which he delegates his governance power by calling `alterDelegatee`. A surrogate contract is [deployed](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/491c7f63e5799d95a181be4a978b2f074dc219a5/src/UniStaker.sol#L609-L612) (or an existing one is fetched) for the new delegatee. The new surrogate contract [approve](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/491c7f63e5799d95a181be4a978b2f074dc219a5/src/DelegationSurrogate.sol#L27) UniStaker as spender. But the approval from old surrogate to UniStaker contract is not cleared. This may became problematic in case of using a different UniStaker in the future. 

Consider resetting approval when transferring stakeToken from one surrogate to another.

### L-04 ClaimFees can revert if payoutAmount < 2_592_000
`ClaimFees` can be called permisionless to claim protocol fees for `payoutAmount` of `PAYOUT_TOKEN`. 
First `payoutAmount` is transferred from caller to `REWARD_RECEIVER`.

Next, `notifyRewardAmount` is [called](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/491c7f63e5799d95a181be4a978b2f074dc219a5/src/V3FactoryOwner.sol#L188) :

- `scaledRewardRate` is [set](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/491c7f63e5799d95a181be4a978b2f074dc219a5/src/UniStaker.sol#L578) to `(_amount * SCALE_FACTOR) / REWARD_DURATION`
- a few lines below we have this [check](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/491c7f63e5799d95a181be4a978b2f074dc219a5/src/UniStaker.sol#L587) `if ((scaledRewardRate / SCALE_FACTOR) == 0) revert UniStaker__InvalidRewardRate()`
`REWARD_DURATION` [is](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/UniStaker.sol#L130) 30 days and is equal to 30 * 60 * 60 * 24 = 2_592_000

```solidity
    if (block.timestamp >= rewardEndTime) {
@>      scaledRewardRate = (_amount * SCALE_FACTOR) / REWARD_DURATION;
    } else {
      uint256 _remainingReward = scaledRewardRate * (rewardEndTime - block.timestamp);
      scaledRewardRate = (_remainingReward + _amount * SCALE_FACTOR) / REWARD_DURATION;
    }
    rewardEndTime = block.timestamp + REWARD_DURATION;
    lastCheckpointTime = block.timestamp;
@>    if ((scaledRewardRate / SCALE_FACTOR) == 0) revert UniStaker__InvalidRewardRate();
```
If received amount is smaller than 2_592_000, the division from if check will round down to 0 and reward claiming will revert.

For the low probability of setting a `payoutAmount` smaller than 2_592_000 (eg. in case a small decimal token is used), enforce `payoutAmount` to be bigger or equal to 2_592_000.

Note: Sponsor mentioned that `The proposed rewards token is WETH though that is not 100% guarranteed`. 

```solidity
  function setPayoutAmount(uint256 _newPayoutAmount) external {
    _revertIfNotAdmin();
-    if (_newPayoutAmount == 0 ) revert V3FactoryOwner__InvalidPayoutAmount();
+    if (_newPayoutAmount < 2_592_000) revert V3FactoryOwner__InvalidPayoutAmount();
    emit PayoutAmountSet(payoutAmount, _newPayoutAmount);
    payoutAmount = _newPayoutAmount;
  }
```