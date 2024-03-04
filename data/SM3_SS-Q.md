
| Number | Issue | Instences |
|--------|-------|-----------|
|[L-01]| Constructor is missing zero address check | 1 |
|[L-02]| Missing checks for address(0x0) when updating address state variables | 2 |
|[L-03]| Checks-Effects-Interactions pattern not followed | 2 |
|[L-04]| Governance functions should be controlled by time locks | 6 |

## [G-01] Constructor is missing zero address check

In Solidity, constructors often take address parameters to initialize important components of a contract, such as owner or linked contracts. However, without a check, there's a risk that an address parameter could be mistakenly set to the zero address (0x0). This could occur due to a mistake or oversight during contract deployment. A zero address in a crucial role can cause serious issues, as it cannot perform actions like a normal address, and any funds sent to it are irretrievable. Therefore, it's crucial to include a zero address check in constructors to prevent such potential problems. If a zero address is detected, the constructor should revert the transaction.

```solidity
file: blob/main/src/V3FactoryOwner.sol

89   constructor:  address _admin,

```
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L89

## [L-02] Missing checks for address(0x0) when updating address state variables

```solidity
file: blob/main/src/UniStaker.sol

774  admin = _newAdmin;

```
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L774

```solidity
file: blob/main/src/V3FactoryOwner.sol

114  admin = _newAdmin;

```
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L114



## [L-03] Checks-Effects-Interactions pattern not followed

The [Checks-Effects-Interactions](https://fravoll.github.io/solidity-patterns/checks_effects_interactions.html) pattern (CEI) is a best practice that reduces the attack surface for reentrancy attacks.  To adhere to this pattern, place state variable updates before external calls within functions.

```solidity
file: blob/main/src/V3FactoryOwner.sol

181  function claimFees(
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
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L181-L198

```solidity
file: blob/main/src/UniStaker.sol

740   function _claimReward(address _beneficiary) internal {
    _checkpointGlobalReward();
    _checkpointReward(_beneficiary);

    uint256 _reward = unclaimedRewardCheckpoint[_beneficiary];
    if (_reward == 0) return;
    unclaimedRewardCheckpoint[_beneficiary] = 0;
    emit RewardClaimed(_beneficiary, _reward);

    SafeERC20.safeTransfer(REWARD_TOKEN, _beneficiary, _reward);
  }

```
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L740-L750 

## [G-04] Governance functions should be controlled by time locks

Governance functions (such as upgrading contracts, setting critical parameters) should be controlled using time locks to introduce a delay between a proposal and its execution. This gives users time to exit before a potentially dangerous or malicious operation is applied.

```solidity
file: blob/main/src/UniStaker.sol

201 function setAdmin(address _newAdmin) external {
    _revertIfNotAdmin();
    _setAdmin(_newAdmin);
  }

210 function setRewardNotifier(address _rewardNotifier, bool _isEnabled) external {
    _revertIfNotAdmin();
    isRewardNotifier[_rewardNotifier] = _isEnabled;
    emit RewardNotifierSet(_rewardNotifier, _isEnabled);
  }

771 function _setAdmin(address _newAdmin) internal {
    _revertIfAddressZero(_newAdmin);
    emit AdminSet(admin, _newAdmin);
    admin = _newAdmin;
  }
```
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L201-L204


```solidity
file: blob/main/src/V3FactoryOwner.sol

110 function setAdmin(address _newAdmin) external {
    _revertIfNotAdmin();
    if (_newAdmin == address(0)) revert V3FactoryOwner__InvalidAddress();
    emit AdminSet(admin, _newAdmin);
    admin = _newAdmin;
  }

119 function setPayoutAmount(uint256 _newPayoutAmount) external {
    _revertIfNotAdmin();
    if (_newPayoutAmount == 0) revert V3FactoryOwner__InvalidPayoutAmount();
    emit PayoutAmountSet(payoutAmount, _newPayoutAmount);
    payoutAmount = _newPayoutAmount;
  }

142 function setFeeProtocol(
    IUniswapV3PoolOwnerActions _pool,
    uint8 _feeProtocol0,
    uint8 _feeProtocol1
  ) external {
    _revertIfNotAdmin();
    _pool.setFeeProtocol(_feeProtocol0, _feeProtocol1);
  }

```
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L110-L115