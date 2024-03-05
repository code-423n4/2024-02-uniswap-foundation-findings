## [L-01] Token approvals for the `UniStaker` contract are not cleared post withdraw
When a staker delegates their token and stakes in the `UniStaker` contract, the `UniStaker` is approved for `type(uint256).max` tokens on behalf of the staker. This is a good approval limit because the staker can decide to increase their stakes at any time. However, since when you make a transfer, approvals for the max aren't cleared, the `Unistaker` will still have an enormous amount of tokens approved for it after a staker withdraws.

- (UniStaker.sol#L660)[https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L660]
- (DelegationSurrogate.sol#L27)[https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/DelegationSurrogate.sol#L27]
- (UniStaker.sol#L733)[https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L733]

```js
  FILE: Openzeppelin ERC20 Implementation
    function transferFrom(address from, address to, uint256 value) public virtual returns (bool) {
        address spender = _msgSender();
        _spendAllowance(from, spender, value); // @note if approval is for the max of uint256, skips this line of reducing the approval by the value being transferred and just run the transfer
        _transfer(from, to, value);
        return true;
    }
```

Since when a staker withdraws, their tokens are refunded, it's sensible to clear any pending approvals:

```diff
function _stakeTokenSafeTransferFrom(address _from, address _to, uint256 _value) internal {
    SafeERC20.safeTransferFrom(IERC20(address(STAKE_TOKEN)), _from, _to, _value);

    // check if the staker has no more positive value stakes

+    if () { // do check here
+      SafeERC20.safeDecreaseAllowance(IERC20(addres(STAKE_TOKEN)), address(this), 0);
+    }
}
```

## [L-02] Compromised owner address will lock-up governance tokens in the `DelegationSurrogate` contract
If a staker were to lose access to their wallet used for staking due to a compromisation of such wallet, there's no way to safely unwind their stakes in the `UniStaker` contract. There are a couple ways to mitigate this issue.

- Implement a `recoverToken` function in the UniStaker contract to pull the funds from the `DelegationSurrogate` contract. This function would be centralized.
- Implement the `recoverToken` function in the `DelegationSurrogate` contract directly. Also, the function would be centralized.
- Have a way to `burn` and then `mint` again to the staker whose address got compromised. This process can be done via two functions. 1. Staker initiates, gets put behind a timelock so they don't begin withdrawal. 2. Burn, mint and restake on behalf of the staker in the same transaction.


```js
function _withdraw(Deposit storage deposit, DepositIdentifier _depositId, uint256 _amount)
    internal
  {
    _checkpointGlobalReward();
    _checkpointReward(deposit.beneficiary);

    deposit.balance -= _amount; // overflow prevents withdrawing more than balance
    totalStaked -= _amount;
    depositorTotalStaked[deposit.owner] -= _amount;
    earningPower[deposit.beneficiary] -= _amount;
    _stakeTokenSafeTransferFrom(address(surrogates[deposit.delegatee]), deposit.owner, _amount); // @audit LOW compromised wallet loses stake tokens when withdrawn
    emit StakeWithdrawn(_depositId, _amount, deposit.balance);
  }
```

Supposed mitigation as described above in point 3:

```diff
+ function initiateRecovery() external {
+    // do logic to gather their deposit balances
+    // do logic to put msg sender behind the timelock for withdrawals
+ }

+ function completeRecover(address _toRecover) external onlyOwner { // defaulted to onlyOwner but the team can utilize admins so the process is faster as admins would likely be more active than owner
+    // do logic to gather `_toRecover` deposit balances
+    // do logic to burn their stake tokens
+    // do logic to transfer their pending rewards to `_toRecover`
+    // do logic to mint them new stake tokens
+    // do logic to restake them in the system
+ }
```

## [L-03] `StakeWithdrawn` should also index the user/owner who withdrew
As is, the `StakeWithdrawn` defined event emits an event that only indexes the `depositId` for the stake, `amount` being deposited and the leftover `depositBalance` of the deposit the user made. This event can use another topic field such as an `owner` address field for off-chain monitors tracking staking activities in the `UniStaker` contract.

- (UniStaker.sol#L660)[https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L41]

/// @notice Emitted when a depositor withdraws some portion of stake from a given deposit. @audit missing `owner` field

```diff
- event StakeWithdrawn(DepositIdentifier indexed depositId, uint256 amount, uint256 depositBalance);
+ event StakeWithdrawn(address owner, DepositIdentifier indexed depositId, uint256 amount, uint256 depositBalance);
```

## [L-04] Misbehaving claimant can notify rewards of small amounts
The `claimFees()` function of the V3FactoryOwner should set a limit of how much amount0 & amount1 can be requested to avoid scenarios where the UniStaker contract is notified for very minimal amounts e.g 0.005 ether.

- (V3FactoryOwner.sol#L181-L198)[https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L181]


```diff
function claimFees( // @audit info small amounts can still be notified by a misbehaving claimant
    IUniswapV3PoolOwnerActions _pool,
    address _recipient,
    uint128 _amount0Requested,
    uint128 _amount1Requested
  ) external returns (uint128, uint128) {
    PAYOUT_TOKEN.safeTransferFrom(msg.sender, address(REWARD_RECEIVER), payoutAmount);
    REWARD_RECEIVER.notifyRewardAmount(payoutAmount);
    (uint128 _amount0, uint128 _amount1) =
      _pool.collectProtocol(_recipient, _amount0Requested, _amount1Requested);

      // enforce a minimum amount to claim

+   if (_amount0Requested == 0 || _amount1Requested == 0) {
+    revert();
+  }
    // Protect the caller from receiving less than requested. See `collectProtocol` for context.
    if (_amount0 < _amount0Requested || _amount1 < _amount1Requested) {
      revert V3FactoryOwner__InsufficientFeesCollected();
    }
    emit FeesClaimed(address(_pool), msg.sender, _recipient, _amount0, _amount1);
    return (_amount0, _amount1);
  }
```

## [NC-01] `UniStaker__InvalidRewardRate` should also state a reason with the resulting value
The `error UniStaker__InvalidRewardRate()` is thrown when the resulting value of a reward notifier is 0. The error however lacks such reason for the revert and just throws that the rate is invalid. This will lead to confusion and can be better if the error included a reason such as that the resulting `scaledRewardRate` value from the `notifyRewardAmount` transaction is 0.

- (UniStaker.sol#L76)[https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L76]
- (UniStaker.sol#L587)[https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L587]

/// @notice Thrown if the new rate after a reward notification would be zero.

```diff
- error UniStaker__InvalidRewardRate();
+ error UniStaker__InvalidRewardRate(string reason, uint256 value);
```

```diff
function notifyRewardAmount(uint256 _amount) external {
    if (!isRewardNotifier[msg.sender]) revert UniStaker__Unauthorized("not notifier", msg.sender);

    // We checkpoint the accumulator without updating the timestamp at which it was updated, because
    // that second operation will be done after updating the reward rate.
    rewardPerTokenAccumulatedCheckpoint = rewardPerTokenAccumulated();

    if (block.timestamp >= rewardEndTime) {
      scaledRewardRate = (_amount * SCALE_FACTOR) / REWARD_DURATION;
    } else {
      uint256 _remainingReward = scaledRewardRate * (rewardEndTime - block.timestamp);
      scaledRewardRate = (_remainingReward + _amount * SCALE_FACTOR) / REWARD_DURATION;
    }

    rewardEndTime = block.timestamp + REWARD_DURATION;
    lastCheckpointTime = block.timestamp;

-   if ((scaledRewardRate / SCALE_FACTOR) == 0) revert UniStaker__InvalidRewardRate();
+   if ((scaledRewardRate / SCALE_FACTOR) == 0) revert UniStaker__InvalidRewardRate("Reward rate resulted in zero from this update: ", scaledRewardRate);

    // This check cannot _guarantee_ sufficient rewards have been transferred to the contract,
    // because it cannot isolate the unclaimed rewards owed to stakers left in the balance. While
    // this check is useful for preventing degenerate cases, it is not sufficient. Therefore, it is
    // critical that only safe reward notifier contracts are approved to call this method by the
    // admin.
    if (
      (scaledRewardRate * REWARD_DURATION) > (REWARD_TOKEN.balanceOf(address(this)) * SCALE_FACTOR)
    ) revert UniStaker__InsufficientRewardBalance();

    emit RewardNotified(_amount, msg.sender);
}
```

## [NC-02] Zero value stakes should be prohibited
As is, zero value stakes can be executed by depositors. Such stakes yield nothing for the protocol neither does it for the calling user. Such stake transactions are better off stamped out. Reverting such transactions at the internal call level as seen in the recommendation code below is better than having the check in all of the staking functions.

- (UniStaker.sol#L638-L664)[https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L638-L664]

```diff
function _stake(address _depositor, uint256 _amount, address _delegatee, address _beneficiary)
    internal
    returns (DepositIdentifier _depositId)
  {
    _revertIfAddressZero(_delegatee);
    _revertIfAddressZero(_beneficiary);

+   require(_amount !=  0);

    _checkpointGlobalReward();
    _checkpointReward(_beneficiary);

    DelegationSurrogate _surrogate = _fetchOrDeploySurrogate(_delegatee);
    _depositId = _useDepositId();

    totalStaked += _amount;
    depositorTotalStaked[_depositor] += _amount;
    earningPower[_beneficiary] += _amount;
    deposits[_depositId] = Deposit({
      balance: _amount,
      owner: _depositor,
      delegatee: _delegatee,
      beneficiary: _beneficiary
    });
    _stakeTokenSafeTransferFrom(_depositor, address(_surrogate), _amount);
    emit StakeDeposited(_depositor, _depositId, _amount, _amount);
    emit BeneficiaryAltered(_depositId, address(0), _beneficiary);
    emit DelegateeAltered(_depositId, address(0), _delegatee);
  }
```