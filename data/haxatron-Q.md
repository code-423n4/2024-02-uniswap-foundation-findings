## [L-01]: Fee claiming can revert if user specifies the entire fee amount present in the pool for either `_amount0Requested` or `_amount1Requested`

When claiming pool fees, UniStaker uses an auction-like mechanism where the user can pay a fixed price to obtain a specified amount of the protocol fees.

[V3FactoryOwner.sol#L181-L198](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L181-L198)
```solidity
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

    // Protect the caller from receiving less than requested. See `collectProtocol` for context.
    if (_amount0 < _amount0Requested || _amount1 < _amount1Requested) {
      revert V3FactoryOwner__InsufficientFeesCollected();
    }
    emit FeesClaimed(address(_pool), msg.sender, _recipient, _amount0, _amount1);
    return (_amount0, _amount1);
  }
```
In the code above, there are checks to protect the caller from receiving less than the amount requested, which will lead to a revert. This function calls `collectProtocol` on the Uniswap V3 pool contract to collect the fees.

[UniswapV3Pool.sol#L848-L868](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#L848-L868)
```solidity
    function collectProtocol(
        address recipient,
        uint128 amount0Requested,
        uint128 amount1Requested
    ) external override lock onlyFactoryOwner returns (uint128 amount0, uint128 amount1) {
        amount0 = amount0Requested > protocolFees.token0 ? protocolFees.token0 : amount0Requested;
        amount1 = amount1Requested > protocolFees.token1 ? protocolFees.token1 : amount1Requested;

        if (amount0 > 0) {
            if (amount0 == protocolFees.token0) amount0--; // ensure that the slot is not cleared, for gas savings
            protocolFees.token0 -= amount0;
            TransferHelper.safeTransfer(token0, recipient, amount0);
        }
        if (amount1 > 0) {
            if (amount1 == protocolFees.token1) amount1--; // ensure that the slot is not cleared, for gas savings
            protocolFees.token1 -= amount1;
            TransferHelper.safeTransfer(token1, recipient, amount1);
        }

        emit CollectProtocol(msg.sender, recipient, amount0, amount1);
    }
```
In the fee claiming above, notice that to save gas, if the user specifies `amount0 == protocolFees.token0`, we deduct 1 from `amount0` and if `amount1 == protocolFees.token1` we also deduct 1 from `amount1`.

 However, when `amount0` and `amount1` is returned back to the original `claimFees` function it will result in `_amount0 < _amount0Requested` or `_amount1 < _amount1Requested`, which will cause the entire `claimFees` function to revert.

Therefore, if the user specifies either `_amount0Requested` or `_amount1Requested` to try and claim full protocol fees for that specific pool, they will never be able to do so because the `collectProtocol` function will always return 1 less for the `_amount0Requested` or `_amount1Requested` which will cause the following check to revert. This unexpected behaviour can cause users to waste gas and even lose the auction. 

It is recommended that this behaviour should be documented.

## [L-02]: User can specify `_pool` to an address of a contract they own.

In the `claimFees`, there is no validation whether the `_pool` specified is really a Uniswap V3 pool.

[V3FactoryOwner.sol#L181-L198](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L181-L198)
```solidity
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

    // Protect the caller from receiving less than requested. See `collectProtocol` for context.
    if (_amount0 < _amount0Requested || _amount1 < _amount1Requested) {
      revert V3FactoryOwner__InsufficientFeesCollected();
    }
    emit FeesClaimed(address(_pool), msg.sender, _recipient, _amount0, _amount1);
    return (_amount0, _amount1);
  }
```
A user can specify `_pool` and specify an arbitrary `_amount0Requested` and `_amount1Requested` to a contract they own, where the `collectProtocol` function doesn't do anything and returns an `_amount0` and `_amount1` lesser than `_amount0Requested` and `_amount1Requested` to pass the revert check. 

This can cause the `_amount0` and `_amount1` variables `FeesClaimed` event and the return value of the `claimFees` function to be an extremely large value that is fully controlled by the user and potentially cause errors in off-chain systems that log `FeesClaimed` event or integrators that call the `claimFees` function.

Consider either documenting this to prevent any integrators from making mistakes here or adding an allowlist of valid Uniswap V3 pools.

## [L-03]: No validation of `payoutAmount` against a minimum

In [V3FactoryOwner.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol), the `payoutAmount` variable is the amount paid by the user to call the `claimFees` function to claim fees from the Uniswap V3 pool. This `payoutAmount` is sent to the `UniStaker.sol` contract to be distributed to beneficiaries via `notifyRewardAmount`

A known issue in the `notifyRewardAmount` is that a misbehaving reward notifier can frequently call this function with tiny reward amounts.

[UniStaker.sol#L562-L565](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L562-L565)
```solidity
  /// 1. A misbehaving contract could grief stakers by frequently notifying this contract of tiny
  ///    rewards, thereby continuously stretching out the time duration over which real rewards are
  ///    distributed. It is required that reward notifiers supply reasonable rewards at reasonable
  ///    intervals.
```

However, when setting the `payoutAmount`, there is only a check for a zero value but there isn't a check against a minimum amount.

[V3FactoryOwner.sol#L119-L124](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L119-L124)
```solidity
  function setPayoutAmount(uint256 _newPayoutAmount) external {
    _revertIfNotAdmin();
    if (_newPayoutAmount == 0) revert V3FactoryOwner__InvalidPayoutAmount();
    emit PayoutAmountSet(payoutAmount, _newPayoutAmount);
    payoutAmount = _newPayoutAmount;
  }
```

In case the admin sets too low of a `payoutAmount`, it would be possible to face the problem outlined in the comment above.

Therefore, it is recommended to have a constant `MIN_PAYOUT_AMOUNT` in the contract and check against this value when setting the `payoutAmount`.

## [L-04]: Consider using two-step admin transfers similar to `Ownable2Step`

It is possible to change the admin / owner in one function `setAdmin` for both `V3FactoryOwner.sol` and `UniStaker.sol` which represents a single point of failure in case the admin is compromised

[V3FactoryOwner.sol#L110-L115](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L110-L115)
```solidity
  function setAdmin(address _newAdmin) external {
    _revertIfNotAdmin();
    if (_newAdmin == address(0)) revert V3FactoryOwner__InvalidAddress();
    emit AdminSet(admin, _newAdmin);
    admin = _newAdmin;
  }
```

[UniStaker.sol#L198-L205](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L198-L205)
```solidity
  function setAdmin(address _newAdmin) external {
    _revertIfNotAdmin();
    _setAdmin(_newAdmin);
  }
```

Consider using two-step admin transfers similar to how OpenZeppelin's `Ownable2Step` is implemented [here](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol). 

## [L-05]: Missing zero address check for certain variables in `V3FactoryOwner.sol` uncaught in bot report.

In `V3FactoryOwner.sol`, there are missing zero address checks not highlighted by the bot report in bot finding [L-03](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/bot-report.md#l-03-missing-zero-address-check-in-constructor)

[V3FactoryOwner.sol#L88-L106](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L88-L106)
```solidity
  constructor(
    address _admin,
    IUniswapV3FactoryOwnerActions _factory,
    IERC20 _payoutToken,
    uint256 _payoutAmount,
    INotifiableRewardReceiver _rewardReceiver
  ) {
    if (_admin == address(0)) revert V3FactoryOwner__InvalidAddress();
    if (_payoutAmount == 0) revert V3FactoryOwner__InvalidPayoutAmount();

    admin = _admin;
    FACTORY = _factory;
    PAYOUT_TOKEN = _payoutToken;
    payoutAmount = _payoutAmount;
    REWARD_RECEIVER = _rewardReceiver;

    emit AdminSet(address(0), _admin);
    emit PayoutAmountSet(0, _payoutAmount);
  }
```
In particular, the variables `_factory`, `_payoutToken` and `_rewardReceiver` are missing the zero address checks.

Consider adding zero address checks for these variables.

## [R-01]: Consider adding the amount of rewards claimed as a return value to `claimReward` and `_claimReward`

As per the TrailOfBits audit [report](https://docs.unistaker.io/UniStaker-Trail-of-Bits-Audit-Report.pdf), consider adding the amount of rewards claimed as a return value in the `claimReward` and `_claimReward` functions, so that the user and potential integrators knows how much rewards were claimed. 

[UniStaker.sol#L534-L538](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L534-L538)
```solidity
  /// @notice Claim reward tokens the message sender has earned as a stake beneficiary. Tokens are
  /// sent to the message sender.
  function claimReward() external {
    _claimReward(msg.sender);
  }
```

## [R-02]: No-op operations should return / revert early

It might make sense to return / revert early for certain no-op operations, before the events are emitted to prevent spamming of event emissions.

- Staking more 0 tokens

[UniStaker.sol#L669-L683](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L669-L683)
```diff solidity
  function _stakeMore(Deposit storage deposit, DepositIdentifier _depositId, uint256 _amount)
    internal
  {
+   if (amount == 0) return;
    _checkpointGlobalReward();
    _checkpointReward(deposit.beneficiary);

    DelegationSurrogate _surrogate = surrogates[deposit.delegatee];

    totalStaked += _amount;
    depositorTotalStaked[deposit.owner] += _amount;
    earningPower[deposit.beneficiary] += _amount;
    deposit.balance += _amount;
    _stakeTokenSafeTransferFrom(deposit.owner, address(_surrogate), _amount);
    emit StakeDeposited(deposit.owner, _depositId, _amount, deposit.balance);
  }
```

- Withdraw 0 tokens

[UniStaker.sol#L723-L735](UniStaker.sol#L723-L735)
```diff solidity
  function _withdraw(Deposit storage deposit, DepositIdentifier _depositId, uint256 _amount)
    internal
  {
+   if (amount == 0) return;
    _checkpointGlobalReward();
    _checkpointReward(deposit.beneficiary);

    deposit.balance -= _amount; // overflow prevents withdrawing more than balance
    totalStaked -= _amount;
    depositorTotalStaked[deposit.owner] -= _amount;
    earningPower[deposit.beneficiary] -= _amount;
    _stakeTokenSafeTransferFrom(address(surrogates[deposit.delegatee]), deposit.owner, _amount);
    emit StakeWithdrawn(_depositId, _amount, deposit.balance);
  }
```

- When the new delegatee is the same

[UniStaker.sol#L685-L699](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L685-L699)
```diff solidity
  function _alterDelegatee(
    Deposit storage deposit,
    DepositIdentifier _depositId,
    address _newDelegatee
  ) internal {
    _revertIfAddressZero(_newDelegatee);
+   if (deposit.delegatee == _newDelegatee) return;
    DelegationSurrogate _oldSurrogate = surrogates[deposit.delegatee];
    emit DelegateeAltered(_depositId, deposit.delegatee, _newDelegatee);
    deposit.delegatee = _newDelegatee;
    DelegationSurrogate _newSurrogate = _fetchOrDeploySurrogate(_newDelegatee);
    _stakeTokenSafeTransferFrom(address(_oldSurrogate), address(_newSurrogate), deposit.balance);
  } 
```

- When the new beneficiary is the same

[UniStaker.sol#L704-L718](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L704-L718)
```diff solidity
  function _alterBeneficiary(
    Deposit storage deposit,
    DepositIdentifier _depositId,
    address _newBeneficiary
  ) internal {
    _revertIfAddressZero(_newBeneficiary);
+   if (deposit.beneficiary == _newBeneficiary) return;
    _checkpointGlobalReward();
    _checkpointReward(deposit.beneficiary);
    earningPower[deposit.beneficiary] -= deposit.balance;

    _checkpointReward(_newBeneficiary);
    emit BeneficiaryAltered(_depositId, deposit.beneficiary, _newBeneficiary);
    deposit.beneficiary = _newBeneficiary;
    earningPower[_newBeneficiary] += deposit.balance;
  }
```

## [NC-01]: Part of NatSpec comment in `DelegationSurrogate.sol` contains multiple typos and is hard to read

There are some typos in one line of the NatSpec comment in `DelegationSurrogate.sol` which makes it difficult to read.

Firstly, there is a double spacing in between `appropriate` and `surrogate—or`.

Secondly, the introduction of random hyphens in the sentence such `surrogate—or` and `behalf—users` which makes it hard to read. These hyphenated words can be misinterpreted as compound words.

[DelegationSurrogate.sol#L15](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/DelegationSurrogate.sol#L15)
```solidity
/// depositor's tokens to the appropriate  surrogate—or deploys it on their behalf—users can retain
```
The line should be changed to:
```solidity
/// depositor's tokens to the appropriate surrogate or deploys it on their behalf, users can retain
```
Which makes it easier to read in the full sentence.
```solidity
/// If a pool contract deploys a DelegationSurrogate for each delegatee, and transfers each
/// depositor's tokens to the appropriate surrogate or deploys it on their behalf, users can retain
/// their governance rights.
```

## [NC-02]: Missing variable names in mappings decreases readability.

There are missing variable names for the value in the key-value pair in some mappings:

[UniStaker.sol#L175](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L175)
```solidity
  mapping(address account => uint256) public beneficiaryRewardPerTokenCheckpoint;
```
[UniStaker.sol#L185](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L185)
```solidity
  mapping(address rewardNotifier => bool) public isRewardNotifier;
```

To improve readability, it is recommended that variable names for the value in the key-value pair in these mappings should be present.


## [NC-03]: Misspelling of `oldAdmin` in `AdminSet` event variable name.

There is a misspelling in `AdminSet` event variable name, instead of `oldAmin` it should be `oldAdmin`.

[V3FactoryOwner.sol#L47-L48](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L47-L48)
```solidity
  /// @notice Emitted when the existing admin designates a new address as the admin.
  event AdminSet(address indexed oldAmin, address indexed newAdmin);
```

## [NC-04]: Misspelling of `param` in NatSpec comment

There is a misspelling in NatSpec comment, instead of `parm` it should be `param`.

[V3FactoryOwner.sol#L140](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L140)
```solidity
  /// @param _feeProtocol1 The fee protocol 1 parm to forward to the pool.
```