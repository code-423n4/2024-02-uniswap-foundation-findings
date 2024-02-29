# Findings Overview

| Severity | Title                                                                                                                           |
| -------- | ------------------------------------------------------------------------------------------------------------------------------- |
| L-1      | WETH rewards will be permanently stuck in the UniStaker contract if `notifyRewardAmount` is called before any tokens are staked |
| L-2      | No checks on amounts requested in `V3FactoryOwner::claimFees` could lead to costly mistakes by callers                          |
| NC-1     | Impossible for a user to find the deposit ids associated with their address on chain                                        |
| NC-2     | Incorrect variable name in NATSPEC comments for `_checkpointReward`                                                             |

## [L-1] WETH rewards will be permanently stuck in the UniStaker contract if `notifyRewardAmount` is called before any tokens are staked

One Instance: [notifyRewardAmount function](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L570])

**Impact:**
In the event that there is zero `STAKE_TOKEN` staked when rewards are added to the contract via `UniStaker::notifyRewardAmount`, the rewards will start being emitted per second until `rewardEndTime` despite there being no stakers to receive them. These missing rewards will never be added back in future rounds leaving them permanently locked in the contract.

**Proof of Concept:**
Consider the following steps:

1. `notifyRewardAmount(uint256 _amount)` called while there are no stakers
2. Alice stakes shortly afterwards
3. Alice remains the only staker for 30 days
4. Alice's rewards to claim is less than the `_amount` added to the contract
5. `notifyRewardAmount(uint256 _amount)` is called again.
6. After this 30 days Alice has received the entire `_amount` but the original missing funds are still stuck in the contract.

Add the following test to `UniStaker.t.sol` to confirm this:

```solidity
  function test_stuckRewardsDontGetUnstuck() public {
    // Set alice up with tokens
    address alice = makeAddr("alice");
    _mintGovToken(alice, 100 ether);

    // Add rewards to protocol
    _mintTransferAndNotifyReward(10 ether);

    // Time moves forward
    _jumpAhead(10);

    // Alice stakes tokens
    _stake(alice, 100 ether, alice);

    // Jump ahead past end of rewards
    _jumpAhead(30 days);

    // Alice checks her rewards amount
    uint256 aliceRewards = uniStaker.unclaimedReward(alice);
    assert(aliceRewards < 10 ether);
    console2.log("Alice rewards after round one", aliceRewards);
    // Add new rewards
    _mintTransferAndNotifyReward(10 ether);

    // Jump ahead past end of new rewards
    _jumpAhead(30 days + 1);

    // Check rewards have ended
    assert(block.timestamp > uniStaker.rewardEndTime());

    // Check how much claimable rewards alice has
    uint256 aliceRewardsEnd = uniStaker.unclaimedReward(alice);
    // Allow for some normal precision loss in rewards dispersal
    assertEq(aliceRewardsEnd, aliceRewards + 10 ether);
    console2.log("Alice rewards after round two", aliceRewardsEnd);
  }
```

This produced the following result. As you can see, in the second round alice got the full 10 ether of rewards, but the missing amount from the first round remains untouched:
```solidity
[PASS] test_stuckRewardsDontGetUnstuck() (gas: 563644)
Logs:
  Alice rewards after round one 9999961419753086419
  Alice rewards after round two 19999961419753086419
```

**Recommended Mitigation:**
Consider adding a check to `UniStaker::notifyRewardAmount` to confirm there is some amount staked in the contract before rewards can be added.

Example:

```diff
  // -- snip --
+ error UniStaker__ZeroStaked();

  // -- snip --

  function notifyRewardAmount(uint256 _amount) external {
    if (!isRewardNotifier[msg.sender]) revert UniStaker__Unauthorized("not notifier", msg.sender);

+   if (totalStaked == 0) revert UniStaker__ZeroStaked();

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

    if ((scaledRewardRate / SCALE_FACTOR) == 0) revert UniStaker__InvalidRewardRate();

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

## [L-2] No checks on amounts requested in `V3FactoryOwner::claimFees` could lead to costly mistakes by callers

OneInstance: [claimFees function](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L181)

**Impact**
If `claimFees` is called without inputs to `_amount0Requested` and `_amount1Requested` the function will complete without reverting and the caller will send `payoutAmount` of `PAYOUT_TOKEN` to `REWARD_RECEIVER` for potentially nothing in return.

Additionally as `V3FactoryOwner::claimFees` when called with zero requested amounts does not revert even in the case that fees are yet to be turned on for the given pool, this error could end up being a cause the aforementioned L-1 issue.

**Proof of Concept**
Add the following test to `V3FactoryOwner.t.sol` to confirm this:

```solidity
  function test_ClaimFeesNoRequest() public {
    // Set alice up with payout tokens
    address alice = makeAddr("alice");
    payoutToken.mint(alice, 5 ether);

    // Set up factory owner
    _deployFactoryOwnerWithPayoutAmount(5 ether);

    // Alice mistakenly calls the function with no requested amounts
    vm.startPrank(alice);
    payoutToken.approve(address(factoryOwner), 5 ether);
    (uint128 amount0, uint128 amount1) = factoryOwner.claimFees(pool, alice, 0 , 0);

    // Alice loses her 5 ether
    uint256 aliceBalanceAfter = payoutToken.balanceOf(alice);
    assertEq(aliceBalanceAfter, 0);
    // Alice doesn't receive any fees either
    assertEq(amount0, 0);
    assertEq(amount1, 0);
  }
```

**Recommended Mitigation**
Consider adding zero amount checks for `_amount0Requested` and `_amount1Requested` to help users avoid very costly errors.

Example:
```diff
  // --snip--

+ error V3FactoryOwner__InputError();

  // --snip--

  function claimFees(
    IUniswapV3PoolOwnerActions _pool,
    address _recipient,
    uint128 _amount0Requested,
    uint128 _amount1Requested
  ) external returns (uint128, uint128) {
+   if (_amount0Requested == 0 || _amount1Requested == 0) revert V3FactoryOwner__InputError();
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

## [NC-1] Impossible for a user to find the deposit ids associated with their address on chain

One Instance: [UniStaker contract](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L31)

**Description**
In the current implementation of the `UniStaker` contract there is no way (on chain) for a user to find the deposit id's associated with their address. This means the user either has to keep track of their deposit ids manually when first calling `stake` or rely on an offchain provider to have stored the details for them offchain.

**Recommended Mitigation**
To increase the trustlessness of the protocol and remove a potential point of failure (the frontend going down) it's recommended that the project add the following mapping in `UniStaker.sol`:

```solidity
    mapping(address depositor => DepositIdentifier[] depositIds) public addressToDepositIds;
```

## [NC-2] Incorrect variable name in NATSPEC comments for `_checkpointReward`

One Instance: [UniStaker ln762](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L762)
See the following NATSPEC comment:

```solidity
  /// @notice Checkpoints the unclaimed rewards and reward per token accumulator of a given
  /// beneficiary account.
  /// @param _beneficiary The account for which reward parameters will be checkpointed.
  /// @dev This is a sensitive internal helper method that must only be called after global rewards
  /// accumulator has been checkpointed. It assumes the global `rewardPerTokenCheckpoint` is up to
  /// date.
  function _checkpointReward(address _beneficiary) internal {
```

The comment references a variable `rewardPerTokenCheckpoint` which should be `rewardPerTokenCheckpointAccumulated`. It's recommended this typo is fixed to improve the codes readability.
