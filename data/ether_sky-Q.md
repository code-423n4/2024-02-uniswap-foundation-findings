**[L-1] There may be unallocated `rewards` in the `UniStaker`, but there is no way to recover them.**

When users `claim fees` through `V3FactoryOwner`, it sends the `payout` token to the `UniStaker` without checking whether there are any `staked UNI` tokens.
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L187-L188
```
function claimFees(
    IUniswapV3PoolOwnerActions _pool,
    address _recipient,
    uint128 _amount0Requested,
    uint128 _amount1Requested
) external returns (uint128, uint128) {
    PAYOUT_TOKEN.safeTransferFrom(msg.sender, address(REWARD_RECEIVER), payoutAmount);
    REWARD_RECEIVER.notifyRewardAmount(payoutAmount);
}
```
So there is no guarantee that there will always be some `staked` tokens when the `rewards` are sent to the `UniStaker`.
Once the `rewards` are sent, they are distributed for a certain period. 
If there are `0 staked UNI` tokens for some days, the `rewards` for these days are not allocated to any `depositors`.
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L577-L585
```
function notifyRewardAmount(uint256 _amount) external {
    if (block.timestamp >= rewardEndTime) {
      scaledRewardRate = (_amount * SCALE_FACTOR) / REWARD_DURATION;
    } else {
      uint256 _remainingReward = scaledRewardRate * (rewardEndTime - block.timestamp);
      scaledRewardRate = (_remainingReward + _amount * SCALE_FACTOR) / REWARD_DURATION;
    }

    rewardEndTime = block.timestamp + REWARD_DURATION;
    lastCheckpointTime = block.timestamp;
}
```
However, there is currently no way to recover them.

We should allow the `owner` to recover the excess `payout` token.

**[L-2] The first user can benefit when the payoutAmount changes to a small value.**

Suppose the current `payoutAmount` is `10 WETH` and the available `fees` are worth `8 WETH`. 
Obviously, nobody will claim `fees`. 
At this point, the `admin` is going to change the `payoutAmount` to `5 WETH`.
Then the first user who claims `fees` after will benefit by more than `3 WETH`.
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/491c7f63e5799d95a181be4a978b2f074dc219a5/src/V3FactoryOwner.sol#L187-L188
```
function claimFees(
    IUniswapV3PoolOwnerActions _pool,
    address _recipient,
    uint128 _amount0Requested,
    uint128 _amount1Requested
) external returns (uint128, uint128) {
    PAYOUT_TOKEN.safeTransferFrom(msg.sender, address(REWARD_RECEIVER), payoutAmount);
    REWARD_RECEIVER.notifyRewardAmount(payoutAmount);
    (uint128 _amount0, uint128 _amount1) =
      _pool.collectProtocol(_recipient, _amount0Requested, _amount1Requested);
}
```

When the `admin` changes the `payoutAmount` to a smaller value, they ensure that there are fewer `fees` than the new amount, or they claim `fees` when there are excess `fees`.

**[L-3] In the `_alterBeneficiary` functions, if the `deposit.balance` is `0`, there is no need to update reward**

In the `_alterBeneficiary` function, if the `balance` is `0`, we could skip updating the `reward`. 
However, there is currently no check for this.

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/491c7f63e5799d95a181be4a978b2f074dc219a5/src/UniStaker.sol#L704C12-L718
```
function _alterBeneficiary(
    Deposit storage deposit,
    DepositIdentifier _depositId,
    address _newBeneficiary
) internal {
    _revertIfAddressZero(_newBeneficiary);
    _checkpointGlobalReward();
    _checkpointReward(deposit.beneficiary);
    earningPower[deposit.beneficiary] -= deposit.balance;

    _checkpointReward(_newBeneficiary);
    emit BeneficiaryAltered(_depositId, deposit.beneficiary, _newBeneficiary);
    deposit.beneficiary = _newBeneficiary;
    earningPower[_newBeneficiary] += deposit.balance;
}
```

We could change like below:
```
function _alterBeneficiary(
    Deposit storage deposit,
    DepositIdentifier _depositId,
    address _newBeneficiary
) internal {
    _revertIfAddressZero(_newBeneficiary);
+    if (deposit.balance != 0) {
        _checkpointGlobalReward();
        _checkpointReward(deposit.beneficiary);
        earningPower[deposit.beneficiary] -= deposit.balance;

        _checkpointReward(_newBeneficiary);
-        emit BeneficiaryAltered(_depositId, deposit.beneficiary, _newBeneficiary);
-        deposit.beneficiary = _newBeneficiary;
        earningPower[_newBeneficiary] += deposit.balance;
+    }

+    emit BeneficiaryAltered(_depositId, deposit.beneficiary, _newBeneficiary);
+    deposit.beneficiary = _newBeneficiary;
}
```