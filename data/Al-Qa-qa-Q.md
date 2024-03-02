## Summary

|ID|Title|
|:-:|-----|
|[L-01](#l-01-unistakernotifyrewardamount-reward-transferred-check-can-get-down-permanently)|`UniStaker::notifyRewardAmount` reward transferred check can get down permanently|
|[L-02](#l-02-mev-searcher-will-not-claim-the-fees-for-the-swaps-that-occurred-after-initializing-claimfees-function)|MEV searcher will not claim the fees for the swaps that occurred after initializing `claimFees` function|
|[L-03](#l-03-signatures-do-not-have-a-deadline-even-the-significant-ones)|Signatures do not have a deadline, even the significant ones|
|[L-04](#l-04-stake-more-signature-do-not-check-beneficiary-address-changing)|Stake More signature do not check beneficiary address changing|
|||
|[NC&#8209;01](#nc-01-factoryowner-cannot-activate-more-than-one-pool-at-a-time)|FactoryOwner cannot activate more than one pool at a time|
|[NC&#8209;02](#nc-02-values-are-not-checked-that-they-differ-from-the-old-values-when-altering-them)|Values are not checked that they differ from the old values when altering them|
|[NC&#8209;03](#nc-03-unauthorized-error-has-no-parameters-in-v3factoryowner)|Unauthorized error has no parameters in `V3FactoryOwner`|
|[NC&#8209;04](#nc-04-type-definition-should-be-at-the-top)|Type definition should be at the top|
|[NC&#8209;05](#nc-05-time-weighted-contributions-staking-algorism-is-activated-only-after-the-first-reward-notified)|Time-weighted contributions staking algorism is activated only after the first reward notified|
|[NC&#8209;06](#nc-06-fixed-payoutamount-may-cause-some-pools-unprofitable-to-claim-their-fees)|Fixed `payoutAmount` may cause some pools unprofitable to claim their fees|

---

## [L-01] `UniStaker::notifyRewardAmount` reward transferred check can get down permanently

`Unistaker::notifyRewardAmount` is designed to be called once the award is distributed, but this function design can not guarantee that the rewards are distributed.

Trail Of Bits has mentioned in their report that users' unclaimed tokens are not checked with it, so the `notifyRewardAmount` can be fired without distributing the exact amount.

What we want to point out here is that this check can get permanently off (get passed all the time), if the contract receives donations.

The check is done to see that the amount the contract received + remaining rewards are not greater than the actual contract balance.

[UniStaker.sol#L594-L596](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L594-L596)
```solidity
  function notifyRewardAmount(uint256 _amount) external {
    ...

    if (
      (scaledRewardRate * REWARD_DURATION) > (REWARD_TOKEN.balanceOf(address(this)) * SCALE_FACTOR)
    ) revert UniStaker__InsufficientRewardBalance();

    ...
  }
```

If the contract receives a donation, let's say 1 WETH. An extra 1 WETH will be considered, whenever a reward is sent, and `notifyRewardAmount` is fired.

So the `_amount` value can be atmost 1 WETH less than the actual amount sent by the notifier (if there are no remaining rewards for example).

This issue differs from the case Trail Of Bits mentions, as the problem we mentioned is not because of unclaimed rewards by the users, but because of donations the contract received.

UniStaker devs mentioned that the notifier is required to send `_amount` before calling notify, so we preferred to make it a LOW issue.

### Recommendations
Mitigating this may be a little hard, as we will need to track the amount sent by notifiers, compare the real balance with the amount sent by notifiers, and take suitable actions. And tracking the number of tokens transferred is not an easy task, and will require some changes to `UniStaker`.

As Trail Of Bits said in their report, mitigating the issue will be hard, and UniStaker devs decided to document the check. So I provide adding in the comment that donations can get the check permanently off.

---

## [L-02] MEV searcher will not claim the fees for the swaps that occurred after initializing `claimFees` function
In the implementation of `V3FactoryOwner::claimFee` the caller must provide the amount of tokens (first and second pairs) he wants to withdraw, and there is a check that the amount taken is not less than the amount requested by the user

[V3FactoryOwner.sol#L189-L195](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L189-L195)
```solidity
  function claimFees(... , uint128 _amount0Requested, uint128 _amount1Requested) external returns (uint128, uint128) {
    ...

    (uint128 _amount0, uint128 _amount1) =
      _pool.collectProtocol(_recipient, _amount0Requested, _amount1Requested);

    // Protect the caller from receiving less than requested. See `collectProtocol` for context.
    // @audit check that the amount we received is not less than that we requested
    if (_amount0 < _amount0Requested || _amount1 < _amount1Requested) {
      revert V3FactoryOwner__InsufficientFeesCollected();
    }
    ...
  }
```

So the MEV will not be able to extract the maximum profit from the pool (all fees), if there are some swaps or flash loans done at the time the MEV searcher function was in the mempool.

- Let's say the swap fee is 1 token for each pair
- Fees now are (100,100)
- MEV searcher fired `claimFee` with amounts requested (100,100)
- Some swaps were in the mempool and occurred before `claimFee` and the total fees are (105,105) now
- MEV searcher will only claim 100 from each pair

If the amount requested by the called of the `claimFee` is greater than the protocol fees, the amount gets downed to the maximum in `uniswapv3-pools`

[UniswapV3Pool.sol#L853-L854](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L853-L854)
```solidity
    function collectProtocol(address recipient, uint128 amount0Requested, uint128 amount1Requested) ... ( ... ) {
        // @audit make the amount equals the fees collected if the amount requested is greater than collected
        amount0 = amount0Requested > protocolFees.token0 ? protocolFees.token0 : amount0Requested;
        amount1 = amount1Requested > protocolFees.token1 ? protocolFees.token1 : amount1Requested;

        ...
    }
```

So the amount can be set to a big value by the MEV searcher to guarantee the maximum profit, but because of the `V3OwnerFactory::claimFees` check, the amount received will be < requested and the function will get reverted.

### Recommendations
We can make a mechanism similar to the `minAmountOut` which prevents sandwich attacks.

The caller will provide an amount to request for claiming, and a minimum amount to receive parameters, and if the received amount is < min, the function gets reverted.

Here is a sample of how this can be implemented
> V3FactoryOwner::claimFee()
```diff
  function claimFees(
    IUniswapV3PoolOwnerActions _pool,
    address _recipient,
    uint128 _amount0Requested,
    uint128 _amount1Requested,
+   uint128 _amount0Min,
+   uint128 _amount1Min
  ) external returns (uint128, uint128) {
    PAYOUT_TOKEN.safeTransferFrom(msg.sender, address(REWARD_RECEIVER), payoutAmount);
    REWARD_RECEIVER.notifyRewardAmount(payoutAmount);
    (uint128 _amount0, uint128 _amount1) =
      _pool.collectProtocol(_recipient, _amount0Requested, _amount1Requested);

    // Protect the caller from receiving less than requested. See `collectProtocol` for context.
-   if (_amount0 < _amount0Requested || _amount1 < _amount1Requested) {
+   if (_amount0 < _amount0Min || _amount1 < _amount1Min) {
      revert V3FactoryOwner__InsufficientFeesCollected();
    }
    emit FeesClaimed(address(_pool), msg.sender, _recipient, _amount0, _amount1);
    return (_amount0, _amount1);
  }
```

So the caller (MEV searcher) can call the function providing the requested amounts with `type(uint128).max` for example, and set the minimum to the current fees the protocol collects, or the amount that makes him gain a profit.

---

## [L-03] Signatures do not have a deadline, even the significant ones

In `UniStaker.sol`, all functions like (stake and withdraw) can get fired on behalf of the user, where the user (deposited) signs the message and either a 3rth party fires it or he fires it himself.

Functions like (stake, stakeMore, and claim), which transfers funds, do not have a `deadline` parameter, All functions do not have a deadline parameter, but we want to point to the critical ones.

We can see in `permit` for example, that the function has a `deadline`, as it makes the user allow another party to spend his token, and the case is the same for staking, withdrawing, etc..., So having a `deadline` parameter in the signature, and a `deadline` check is useful in this case.

### Recommendations
Provide a `deadline` parameter for signatures of the critical functions like `stake*()` and `stakeMore*()`, and we can check this `deadline` in `UniStaker::_revertIfSignatureIsNotValidNow()` function.

> UniStaker::_revertIfSignatureIsNotValidNow
```diff
  function _revertIfSignatureIsNotValidNow(
    address _signer,
    bytes32 _hash,
    bytes memory _signature,
+   uint256 deadline
  )
    internal
    view
  {
    bool _isValid = SignatureChecker.isValidSignatureNow(_signer, _hash, _signature);
    if (!_isValid) revert UniStaker__InvalidSignature();

+    // deadline will be `type(uint256).max` for the functions that do not have deadline, or want to allow signature forever
+    if (deadline != type(uint256).max) {
+      require(block.timestamp <= deadline, "UniStaker: Signature expired");
+    }
  }
```

---

## [L-04] Stake More signature do not check beneficiary address changing

In `UniStaker` contract, `STAKE_MORE_TYPEHASH` does not contain the beneficiary address in consideration. So the beneficiary address can get changed after signing the message, and this will end up adding funds to the new beneficiary address, and not the one that existed when signing the message.

[src/UniStaker.sol#L106-L107](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L106-L107)
```solidity
  bytes32 public constant STAKE_MORE_TYPEHASH = // @audit beneficiary is not passed in the signature
    keccak256("StakeMore(uint256 depositId,uint256 amount,address depositor,uint256 nonce)");
```

- Let's say a deposit has a beneficiary address `0x01`
- Message signed by the deposit owner, for giving `0x01` the ability to stake
- depositor changed the beneficial address to `0x02`, to let him earn yields too
- The first beneficiary `0x01` fired `stakeMoreOnBehalf` with his signature signature
- Money goes to the `0x02` instead of `0x01`

Since the deposit owner is the one who can make the signatures, and the one who will alter the beneficiary, the problem is not critical, and the severity of it is low. But maybe for further protocol integrations, this can happen who knows?

### Recommendations
Add `beneficiary` parameter to `STAKE_MORE_TYPEHASH` signature, and provide it with the signature, so that if the beneficiary changes, the message will be invalid

---
---
---

## [NC-01] FactoryOwner cannot activate more than one pool at a time
In `V3FactoryOwner::setFeeProtocol`, the function accepts only one pool to activate the fees on it, and the function is restricted to admins.

Since it's planned to make Uni Governance contract the admin, adding a new pool, will need to make proposal, then vote, then agree, and execute in the last which is not a straight thing.

New ERC20 tokens will launch and pools for them created, so to accept these token pools for fees, you will have to make Governance proposals again and again for each pool separately.

- Let's say token `XYZ` is a new and powerful token launched
- The token has 5 pools (WETH, USDC, ...)
- If we want to accept fees from all pools for that token, we will have to make a proposal for each pole, separately.

### Recommendations
I recommend making the function to set pool fees in batches, where pools are passed to the function as an array, and the function goes to each pool and activates it, this will allow the governance to accept more than one pool in a single proposal.

And instead of modifying the function itself, we can add another function for adding more than one pool for verbosity and flexibility.

---

## [NC-02] Values are not checked that they differ from the old values when altering them
In `Unistaker::_alterDelegatee` and `Unistaker::_alterBeneficiary`, the new value provided is not checked if it is the same as the old value or not, changing by providing the same value will cause emitting events with no meaning and may cause confusion.

### Recommendations
Check that the new address provided either the new delegatee or the new beneficiary equals the old one or not.

---

## [NC-03] Unauthorized error has no parameters in `V3FactoryOwner`

In `UniStaker` contract, the unauthorized error has two parameters as it is used to authorize both the notifier and the admin.

[UniStaker.sol#L73](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L73)
```solidity
  error UniStaker__Unauthorized(bytes32 reason, address caller);
```

But in the case of `V3FactoryOwner`, the error has no parameters, and it is named unauthorized too.

[V3FactoryOwner.sol#L54](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L54)
```solidity
  error V3FactoryOwner__Unauthorized();
```

### Recommendations
Either provide a message in the `V3FactoryOwner__Unauthorized`, or we can change the error name to `V3FactoryOwner__NotAdmin`, so that it will be easy for frontend devs to handle the error in the UI.

---

## [NC-04] Type definition should be at the top
In `UniStaker` contract, the type definition of `DepositIdentifier` is presented inside the contract, this is not a good practice, and type definitions are preferred to be at the top of the file.

[UniStaker.sol#L31-L32](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L31-L32)
```solidity
contract UniStaker is INotifiableRewardReceiver, Multicall, EIP712, Nonces {
  type DepositIdentifier is uint256;

  ...
}
```

### Recommendations
Put the type declaration in the top of the file

 ---

## [NC-05] Time-weighted contributions staking algorism is activated only after the first reward notified

The staking algorism implemented by `synthetix` activates only after the first reward notified, what I mean by this is that as long as the first reward has not been notfyied yet, the one who staked on the early will gain the same as the one who staked before norifying the first reward by some minutes.

Since this is how the algorism work, solving this is not ideal, and can cause more issues. But I wanted to point out this here so that devs notice about this thing.

### Recommendations
Make the first `notify` as early as you can.

- Protocol launched
- Stakeholders staked their funds
- Notify rewards as early as you can

---

## [NC-06] Fixed `payoutAmount` may cause some pools unprofitable to claim their fees

In the current implementation of `V3FactoryOwner`, the `payoutAmount` is fixed for all pools activated. So the one who is going to claim fees and send rewards will have to pay that amount to collect pool actions fees.

Since pools are not always active, and some pools may have fewer actions (swaps and flash loans), the fees collected by the protocol may not reach this limit forever (fees collected be smaller than the amount needed to be paid).

### Recommendations
Since fixing `payoutAmount` is desired by the design, making a variety of payAmount for pools is not intended by the Dev team. So I recommend not adding pools that have fewer active swaps or interactions