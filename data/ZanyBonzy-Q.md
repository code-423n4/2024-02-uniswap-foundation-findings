# 1. Edge case from first depositor staking 1 wei

Links to affected code *

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/UniStaker.sol#L651
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/UniStaker.sol#L753
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/UniStaker.sol#L232-L233C

## Impact
There's no limit minimum or otherwise on the how much can be staked by users. A user(or his beneficiary) who stakes when `totalStaked` is 0, and stakes only 1 wei, will be given full rewards distributable for that period, which will be significantly more than 1 wei. This is because staking with 1 wei can inflate the `rewardPerTokenAccumulatedCheckpoint`. 

When the user stakes 1 wei into the empty pool, the total amount staked is updated from 0 to 1 wei.
```
  function _stake(address _depositor, uint256 _amount, address _delegatee, address _beneficiary)
    internal
    returns (DepositIdentifier _depositId)
  {
...
    totalStaked += _amount;
...
  }  https://github.com/sherlock-audit/2023-12-truflation-judging/issues/130
```
The GlobalReward is checkpointed through the `_checkpointGlobalReward` function which makes a call to the `rewardPerTokenAccumulated` function. The purpose of this is to calculate the `rewardPerTokenAccumulatedCheckpoint`
```
  function _checkpointGlobalReward() internal {
    rewardPerTokenAccumulatedCheckpoint = rewardPerTokenAccumulated();
    lastCheckpointTime = lastTimeRewardDistributed();
  }
```

After this is calculated, the beneificiary's rewards is then checkpointed through the `_checkpointReward` function which sets the `rewardPerTokenAccumulatedCheckpoint` to the beneficiary.

```
  function _checkpointReward(address _beneficiary) internal {
    unclaimedRewardCheckpoint[_beneficiary] = unclaimedReward(_beneficiary);
    beneficiaryRewardPerTokenCheckpoint[_beneficiary] = rewardPerTokenAccumulatedCheckpoint;
  }
  ```
The `rewardPerTokenAccumulated` function is calculated using the formula, which is can essentially be boiled down to rate / totalstaked, which by dividing by 1 wei, will result in a massive `rewardPerTokenAccumulated` being returned.

```
  function rewardPerTokenAccumulated() public view returns (uint256) {
    if (totalStaked == 0) return rewardPerTokenAccumulatedCheckpoint;

    return rewardPerTokenAccumulatedCheckpoint
      + (scaledRewardRate * (lastTimeRewardDistributed() - lastCheckpointTime)) / totalStaked;
  }

```

Upon checking the unclaimed rewards, the user's unclaimed rewards will be equal to rate * user balance / totalstaked which will be the total rate. 

```
  function unclaimedReward(address _beneficiary) public view returns (uint256) {
    return unclaimedRewardCheckpoint[_beneficiary]
      + (
        earningPower[_beneficiary]
          * (rewardPerTokenAccumulated() - beneficiaryRewardPerTokenCheckpoint[_beneficiary])
      ) / SCALE_FACTOR;
  }

```

This causes that for just 1 wei of UNI tokens, a the user can claim the entire payout amount. An edge case, but serious one at that.

## Recommended Mitigation Steps
Introduce a minimum amount of tokens that can be staked. 

***
# 2. Signers can't cancel their signatures before deadline

Lines of code* 

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/UniStaker.sol#L10

## Impact
After signing a signature, a user might want to cancel it for some reason. Ordinarily, this can be done by using the nonce, but the protocol inherits from OpenZeppelin's Nonces contract, which has the `_useNonce` function internal, hence there are no ways to cancel the signature before a deadline. Consequently, the user can't cancel the signature. 

## Recommended Mitigation Steps
Include a function callable only by the position owners to access the internal `_useNonce` function so that they can easily cancel the signature if they so desire.

***

# 3. Use safer CREATE methods with salt to deploy `DelegationSurrogate` contract 

Lines of code* 

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/UniStaker.sol#L612

## Impact

To deploy a DelegationSurrogate contract, the CREATE function is used. The issue is that its vulnerable to Ethereum network reorgs which are rare but still occur. The funds meant to be transferred to a surrogate would be deposited to another surrogate address if they get deployed really close to eachother during the reorg process.  
```
  function _fetchOrDeploySurrogate(address _delegatee)
    internal
    returns (DelegationSurrogate _surrogate)
  {
    _surrogate = surrogates[_delegatee];

    if (address(_surrogate) == address(0)) {
      _surrogate = new DelegationSurrogate(STAKE_TOKEN, _delegatee);
      surrogates[_delegatee] = _surrogate;
      emit SurrogateDeployed(_delegatee, address(_surrogate));
    }
  }
```

## Recommended Mitigation Steps

Use CREATE2/CREATE 3  with salt.
***

# 4.  Introduce a sweep/recover function into the contracts.

Lines of code* 
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/DelegationSurrogate.sol
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol

## Impact

Any tokens sent to the `V3FactoryOwner`, `Unistaker` and `DelegationSurrogate` can potentially be lost as there's no ways to retrieve them. This can also help to preserve the one of the main [invariant](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/README.md#main-invariants) in the delegate surrogate contract which can be broken by tokens including governance tokens being sent directly to the surrogate contract.
 
> The sum of the governance token balances of all the surrogate contracts is equal to the sum of all deposits less the sum of all withdrawals." 

## Recommended Mitigation Steps
Introduce an admin controlled `recoverERC20` function, that can be used to skim extra tokens.

***

# 5. Issue with permit frontrunning can be potentially mititgated by using trustless permit method

Lines of code* 

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/UniStaker.sol#L301
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/UniStaker.sol#L371

## Impact
The `permitAndStake` and `permitAndStakeMore` allows users to use the ERC20 permit functionality to approve and stake in one seamless transaction. This is subject to permit frontrunning which can be used to grief users. 
```
  function permitAndStake(
    uint256 _amount,
    address _delegatee,
    address _beneficiary,
    uint256 _deadline,
    uint8 _v,
    bytes32 _r,
    bytes32 _s
  ) external returns (DepositIdentifier _depositId) {
    STAKE_TOKEN.permit(msg.sender, address(this), _amount, _deadline, _v, _r, _s);
    _depositId = _stake(msg.sender, _amount, _delegatee, _beneficiary);
  }
```
```
  function permitAndStakeMore(
    DepositIdentifier _depositId,
    uint256 _amount,
    uint256 _deadline,
    uint8 _v,
    bytes32 _r,
    bytes32 _s
  ) external {
    Deposit storage deposit = deposits[_depositId];
    _revertIfNotDepositOwner(deposit, msg.sender);

    STAKE_TOKEN.permit(msg.sender, address(this), _amount, _deadline, _v, _r, _s);
    _stakeMore(deposit, _depositId, _amount);
  }
```
## Recommended Mitigation Steps
A potential workaround for this issue can be found [here](https://www.trust-security.xyz/post/permission-denied) and it involves wrapping the `permit` and `approve` functions in a try-catch block, to prevent failure. 

***

# 6. Certain tokens, including the `UNI` token doesn't consider `type(uint256).max` as an infinite approval

Lines of code* 
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/DelegationSurrogate.sol#L27
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/UniStaker.sol#L605C1-L616C4

## Impact

Upon `DelegationSurrogate` deployment by the `Unistaker` contract, the `type(uint256).max` approval is given to the `Unistaker`. 

```
  constructor(IERC20Delegates _token, address _delegatee) {
    _token.delegate(_delegatee);
    _token.approve(msg.sender, type(uint256).max);
  }
```
The issue is that some tokens including the protocol's own stake token, `UNI` downcast `type(uint256).max` approvals to `uint96` and use it as a raw value rather than interpreting it as an infinite approval. Due to the continuous stacks of calls and `safeTransferFrom`s made to the `DelegationSurrogate` contract to transfer these tokens upon staking, unstaking, delegation changes and so on, a time will eventually arrive when this allowance will become less than than the amount to transfer, or will reach 0. At, this point, token transfer from the `DelegationSurrogate` contract will no longer succeed. 

Considering that the `DelegationSurrogate` contract is a dead contract and tokens can't be accessed directly from it, the tokens left in it will be lost forever, inaccesible, leading to loss of funds for the user and inability to change delegates. An extreme edge case, but can be devastating.

## Recommended Mitigation Steps

Introduce an admin controlled `approve` function that can be used to continously renew the  `Unistaker`'s approval or a recover tokens function.

***

# 7. Add checks to prevent same parameter updates 

A check regarding whether the current value and the new value are the same should be added.

Lines of code* 
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/V3FactoryOwner.sol#L110-L124
```
  function setAdmin(address _newAdmin) external {
    _revertIfNotAdmin();
    if (_newAdmin == address(0)) revert V3FactoryOwner__InvalidAddress();
    emit AdminSet(admin, _newAdmin);
    admin = _newAdmin;
  }

  /// @notice Update the payout amount to a new value. Must be called by admin.
  /// @param _newPayoutAmount The value that will be the new payout amount.
  function setPayoutAmount(uint256 _newPayoutAmount) external {
    _revertIfNotAdmin();
    if (_newPayoutAmount == 0) revert V3FactoryOwner__InvalidPayoutAmount();
    emit PayoutAmountSet(payoutAmount, _newPayoutAmount);
    payoutAmount = _newPayoutAmount;
  }
```
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/UniStaker.sol#L688C1-L699C4
```
  function _alterDelegatee(
    Deposit storage deposit,
    DepositIdentifier _depositId,
    address _newDelegatee
  ) internal {
    _revertIfAddressZero(_newDelegatee);
    DelegationSurrogate _oldSurrogate = surrogates[deposit.delegatee];
    emit DelegateeAltered(_depositId, deposit.delegatee, _newDelegatee);
    deposit.delegatee = _newDelegatee;
    DelegationSurrogate _newSurrogate = _fetchOrDeploySurrogate(_newDelegatee);
    _stakeTokenSafeTransferFrom(address(_oldSurrogate), address(_newSurrogate), deposit.balance);
  }

  /// @notice Internal convenience method which alters the beneficiary of an existing deposit.
  /// @dev This method must only be called after proper authorization has been completed.
  /// @dev See public alterBeneficiary methods for additional documentation.
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
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/V3FactoryOwner.sol#L119C1-L125C1
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/V3FactoryOwner.sol#L110C1-L115C4
```
  function setAdmin(address _newAdmin) external {
    _revertIfNotAdmin();
    if (_newAdmin == address(0)) revert V3FactoryOwner__InvalidAddress();
    emit AdminSet(admin, _newAdmin);
    admin = _newAdmin;
  }

  /// @notice Update the payout amount to a new value. Must be called by admin.
  /// @param _newPayoutAmount The value that will be the new payout amount.
  function setPayoutAmount(uint256 _newPayoutAmount) external {
    _revertIfNotAdmin();
    if (_newPayoutAmount == 0) revert V3FactoryOwner__InvalidPayoutAmount();
    emit PayoutAmountSet(payoutAmount, _newPayoutAmount);
    payoutAmount = _newPayoutAmount;
  }

```

***


# 8. IERC20 delegates lacks the `name` function. it should include the name function as a basic erc20 implementation, and as a possible interface to the UNI token contract.

Lines of code* 

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/interfaces/IERC20Delegates.sol#L9

```
interface IERC20Delegates {
  // ERC20 related methods
  function allowance(address account, address spender) external view returns (uint256);
  function approve(address spender, uint256 rawAmount) external returns (bool);
  function balanceOf(address account) external view returns (uint256);
  function decimals() external view returns (uint8);
  function symbol() external view returns (string memory);
  function totalSupply() external view returns (uint256);
  function transfer(address dst, uint256 rawAmount) external returns (bool);
  function transferFrom(address src, address dst, uint256 rawAmount) external returns (bool);
  function permit(
    address owner,
    address spender,
    uint256 rawAmount,
    uint256 deadline,
    uint8 v,
    bytes32 r,
    bytes32 s
  ) external;

```

***

