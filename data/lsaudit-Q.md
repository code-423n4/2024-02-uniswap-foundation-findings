
# [1] Setters should prevent re-setting of the same value

**Files:** `V3FactoryOwner.sol`, `UniStaker.sol`

[File: src/V3FactoryOwner.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L119)
```solidity
119:   function setPayoutAmount(uint256 _newPayoutAmount) external {
120:     _revertIfNotAdmin();
121:     if (_newPayoutAmount == 0) revert V3FactoryOwner__InvalidPayoutAmount();
122:     emit PayoutAmountSet(payoutAmount, _newPayoutAmount);
123:     payoutAmount = _newPayoutAmount;
124:   }
```

It's possible to call function which updates the state variable with the same value.
It's a good practice to make sure, that whenever we update some value, it's not being set to the same value which it was before the update.
E.g., let's consider a scenario where `payoutAmount = 123;`. Calling `setPayoutAmount(123)` is possible, even though it won't change the value of the variable: `payoutAmount` will remain `123`.
Moreover, after calling `setPayoutAmount(123)`, function will still emit an `PayoutAmountSet()` event, even though nothing had been upgraded (the `payoutAmount` would remain the same). Seeing that event might be confusing for the end-user.

Our recommendation is to implement additional check which verifies if value is really changed. This can be done in `setPayoutAmount()` function, by adding additional `require` check:

```
require(payoutAmount != _newPayoutAmount, "Nothing to update!");
```

The same issue has been observed in the following instances:


[File: src/V3FactoryOwner.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L110)
```solidity
110:   function setAdmin(address _newAdmin) external {
```

[File: src/V3FactoryOwner.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L142)
```solidity
142:   function setFeeProtocol(
```

[File: src/UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L201)
```solidity
201:   function setAdmin(address _newAdmin) external {
```

[File: src/UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L210)
```solidity
210:   function setRewardNotifier(address _rewardNotifier, bool _isEnabled) external {
```

[File: src/UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L771)
```solidity
771:   function _setAdmin(address _newAdmin) internal {
```


# [2] Incorrect comment in `_stakeTokenSafeTransferFrom()`

**File:** `UniStaker.sol`

[File: src/UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L618)
```solidity
618:   /// @notice Internal convenience method which calls the `transferFrom` method on the stake token
619:   /// contract and reverts on failure.
620:   /// @param _from Source account from which stake token is to be transferred.
621:   /// @param _to Destination account of the stake token which is to be transferred.
622:   /// @param _value Quantity of stake token which is to be transferred.
623:   function _stakeTokenSafeTransferFrom(address _from, address _to, uint256 _value) internal {
624:     SafeERC20.safeTransferFrom(IERC20(address(STAKE_TOKEN)), _from, _to, _value);
625:   }
```

The function's NatSpec (line 618) suggests, that `_stakeTokenSafeTransferFrom()` uses `transferFrom()`, although, it uses `safeTransferFrom()`. While functions `transferFrom()` and `safeTransferFrom()` do basically the same - their behavior is slightly different for some non-standard ERC-20 tokens (e.g., there are lots of ERC-20 tokens which don't return anything, thus their behavior won't be consistent with `transferFrom()`). The NatSpec comment should straightforwardly state that function `_stakeTokenSafeTransferFrom()` uses `safeTransferFrom()`. Stating that it uses `transferFrom()` is incorrect and should be changed.

# [3] `lastTimeRewardDistributed()` can be simplified

**File:** `UniStaker.sol`

[File: src/UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L220)
```solidity
220:   function lastTimeRewardDistributed() public view returns (uint256) {
221:     if (rewardEndTime <= block.timestamp) return rewardEndTime;
222:     else return block.timestamp; 
223:   }
```

The `else` statement at line 222 is redundant. When condition at line 221 won't be fulfilled, we will return with the value of `block.timestamp.` The code can be simplified by removing the unnecessary `else` branch:

```
 function lastTimeRewardDistributed() public view returns (uint256) {
    if (rewardEndTime <= block.timestamp) return rewardEndTime;
    return block.timestamp;
  }
```

Simplifying the code-base by removing redundant operations not only does reduce the gas cost, but it also increases the code quality and code readability.


# [4] Code in `notifyRewardAmount()` can be re-organized to be more clearer

**File:** `UniStaker.sol`

[File: src/UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L577)
```solidity
577:     if (block.timestamp >= rewardEndTime) {
578:       scaledRewardRate = (_amount * SCALE_FACTOR) / REWARD_DURATION;
579:     } else {
580:       uint256 _remainingReward = scaledRewardRate * (rewardEndTime - block.timestamp);
581:       scaledRewardRate = (_remainingReward + _amount * SCALE_FACTOR) / REWARD_DURATION;
582:     }
583: 
584:     rewardEndTime = block.timestamp + REWARD_DURATION;
585:     lastCheckpointTime = block.timestamp;
586: 
587:     if ((scaledRewardRate / SCALE_FACTOR) == 0) revert UniStaker__InvalidRewardRate(); 
[...]
594:     if (
595:       (scaledRewardRate * REWARD_DURATION) > (REWARD_TOKEN.balanceOf(address(this)) * SCALE_FACTOR)
596:     ) revert UniStaker__InsufficientRewardBalance();
```

Lines 577-582 are responsible for calculating the value of `scaledRewardRate`. It would be more intuitive to verify the `scaledRewardRate` value (lines 594-596) just after calculating it. 
Instead, the current code-base calculates the value of `rewardEndTime` and `lastCheckpointTime` (lines 584-585) - and then - later - it verifies the `scaledRewardRate` value.
The above issue could be summarized as:

```
577 - 582: calculates scaledRewardRate
584 - 585: calculates rewardEndTime, lastCheckpointTime
594 - 596: performs checks on scaledRewardRate
```

The more intuitive solution (which would increase the code readability), would be:

```
577 - 582: calculate scaledRewardRate
584 - 585: perform checks on scaledRewardRate
594 - 596 calculate rewardEndTime, lastCheckpointTime
```

This would be much more expected flow - thus the code readability will be increased. Moreover, it will increase the user's experience - since it will save some additional gas (function might revert with `UniStaker__InvalidRewardRate()` or  `UniStaker__InsufficientRewardBalance()` earlier, without wasting gas on `rewardEndTime` and `lastCheckpointTime` calculations).
Please notice that `rewardEndTime` and `lastCheckpointTime` calculations are not dependend on code at lines 594-596, thus we can easily move them to the bottom of the function:

```
    if (block.timestamp >= rewardEndTime) {
      scaledRewardRate = (_amount * SCALE_FACTOR) / REWARD_DURATION;
    } else {
      uint256 _remainingReward = scaledRewardRate * (rewardEndTime - block.timestamp);
      scaledRewardRate = (_remainingReward + _amount * SCALE_FACTOR) / REWARD_DURATION;
    }

    if ((scaledRewardRate / SCALE_FACTOR) == 0) revert UniStaker__InvalidRewardRate();

    if (
      (scaledRewardRate * REWARD_DURATION) > (REWARD_TOKEN.balanceOf(address(this)) * SCALE_FACTOR)
    ) revert UniStaker__InsufficientRewardBalance();

    rewardEndTime = block.timestamp + REWARD_DURATION;
    lastCheckpointTime = block.timestamp;
```


# [5] Comment describing `admin` state variable should be more detailed

**File:** `UniStaker.sol`

[File: src/UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L139)
```solidity
139:   /// @notice Permissioned actor that can enable/disable `rewardNotifier` addresses.
140:   address public admin;
```

According to the provided comment, we can assume that `admin` can only enable/disable `rewardNotifier` address. However, `admin` can perform additional operations which are not mentioned in the above comment.
The `admin` not only can enable/disable `rewardNotifier`. He can also change the address of the current `admin` by calling `setAdmin()` function:

[File: src/UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L201)
```solidity
201:   function setAdmin(address _newAdmin) external {
202:     _revertIfNotAdmin();
203:     _setAdmin(_newAdmin);
204:   }
```

Our recommendation is to update the comment from: `/// @notice Permissioned actor that can enable/disable rewardNotifier addresses.`, to:

```
/// @notice Permissioned actor that can enable/disable `rewardNotifier` addresses and set new admin address.
```

# [6] Standarize errors across the whole protocol

**Files:** `V3FactoryOwner.sol`, `UniStaker.sol`

[File: src/V3FactoryOwner.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L202)
```solidity
202:   function _revertIfNotAdmin() internal view {
203:     if (msg.sender != admin) revert V3FactoryOwner__Unauthorized();
204:   }
```

[File: src/UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L779)
```solidity
779:   function _revertIfNotAdmin() internal view {
780:     if (msg.sender != admin) revert UniStaker__Unauthorized("not admin", msg.sender);
781:   }
```

When non-admin address tries to call authorized function, `V3FactoryOwner.sol` reverts with `V3FactoryOwner__Unauthorized()`.
When non-admin address tries to call authorized function, `UniStaker.sol` reverts with `UniStaker__Unauthorized("not admin", msg.sender)`.

To increase code readability, those two errors could have similar parameters. Our recommendation is to change line 54 at `V3FactoryOwner.sol`, from `error V3FactoryOwner__Unauthorized();` to `error V3FactoryOwner__Unauthorized(bytes32 reason, address caller);` (similarly as it's defined in `UniStaker.sol`). Then, `_revertIfNotAdmin()` in `V3FactoryOwner.sol` should be defined as below:

```
  function _revertIfNotAdmin() internal view {
    if (msg.sender != admin) revert V3FactoryOwner__Unauthorized("not admin", msg.sender);
  }
```

Since both functions `_revertIfNotAdmin()` from `V3FactoryOwner.sol` and `_revertIfNotAdmin()` from `UniStaker.sol` do basically the same (verify if the caller is admin) - they should have a similar errors. Standardizing errors for the similar features across the whole code-base, increases the code readability, thus it's highly recommended.

# [7] `REWARD_DURATION` is constant, thus it cannot be updated

**File:** `UniStaker.sol`

[File: src/UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L129)
```solidity
129:   /// @notice Length of time over which rewards sent to this contract are distributed to stakers.
130:   uint256 public constant REWARD_DURATION = 30 days; 
```

`REWARD_DURATION` is constant, thus it cannot be updated. The protocol assumes that the `REWARD_DURATION` will always be 30 days. Protocol does not implement any additional function (called by `admin` only) which would allow to update `REWARD_DURATION`. Even thought this issue does not pose any security risk at all - the lack of ability to update protocol's configuration should be considered as refactoring issue.


# [8] `*__InvalidAddress()` error could be renamed to more descriptive name

**Files:** `V3FactoryOwner.sol`, `UniStaker.sol`

[File: src/V3FactoryOwner.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L56)
```solidity
56:   /// @notice Thrown if the proposed admin is the zero address.
57:   error V3FactoryOwner__InvalidAddress();
```

`V3FactoryOwner__InvalidAddress()` is only thrown when address is `address(0)`. This is confirmed both in the comment section: `Thrown if the proposed admin is the zero address` and in the code-base.

[File: src/V3FactoryOwner.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L95)
```solidity
95:     if (_admin == address(0)) revert V3FactoryOwner__InvalidAddress();
```

[File: src/V3FactoryOwner.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L112)
```solidity
112:     if (_newAdmin == address(0)) revert V3FactoryOwner__InvalidAddress();
```

This suggests, that the name of `V3FactoryOwner__InvalidAddress()` could be changed to: `V3FactoryOwner__AddressZeroProvided()`.
Having more descriptive error messages allows user to better understand why function revert. While `InvalidAddress` might be very vague (address can be invalid for multiple of reasons) - `AddressZeroProvided` is a clear, descriptive message.

The very same issue occurs in `UniStaker.sol`

[File: src/UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L83)
```solidity
83:   /// @notice Thrown if a caller attempts to specify address zero for certain designated addresses.
84:   error UniStaker__InvalidAddress();
```

Since `UniStaker__InvalidAddress()` is used only for `address(0)`-checks:

[File: src/UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L794)
```solidity
794:   function _revertIfAddressZero(address _account) internal pure {
795:     if (_account == address(0)) revert UniStaker__InvalidAddress();
796:   }
```

its name can be changed to: `UniStaker__AddressZeroProvided()`.


# [9] Style-guide: Stick to one way of `if`/`else` syntax

**File:** `UniStaker.sol`

When `if`/`else` branch executes only a single instruction, the curly braces could be omitted. This means, that both syntax are valid:

```
if (condition) functionA();
else functionB();

if (condition) {
    functionA();
} else {
    functionB();
}
```

During the code review, two instances which use both styles were detected:

[File: src/UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L221)
```solidity
221:     if (rewardEndTime <= block.timestamp) return rewardEndTime;
222:     else return block.timestamp;
```

[File: src/UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L577)
```solidity
577:     if (block.timestamp >= rewardEndTime) {
578:       scaledRewardRate = (_amount * SCALE_FACTOR) / REWARD_DURATION;
579:     } else {
580:       uint256 _remainingReward = scaledRewardRate * (rewardEndTime - block.timestamp);
581:       scaledRewardRate = (_remainingReward + _amount * SCALE_FACTOR) / REWARD_DURATION;
582:     }
```

Lines 221-222 do not use curly braces for single `if` instruction, while lines 577-578 use curly braces for a single `if` instruction. 

Sticking to one syntax increases the code readability, thus it's highly recommended. Our recommendation is to use curly braces at lines 221-222.

# [10] Style-guide: Standardize how constant TYPEHASH are being coded

**File:** `UniStaker.sol`

[File: src/UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L102)
```solidity
102:   bytes32 public constant STAKE_TYPEHASH = keccak256(
103:     "Stake(uint256 amount,address delegatee,address beneficiary,address depositor,uint256 nonce)"
```

[File: src/UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L109)
```solidity
109:   bytes32 public constant ALTER_DELEGATEE_TYPEHASH = keccak256(
110:     "AlterDelegatee(uint256 depositId,address newDelegatee,address depositor,uint256 nonce)"
```

[File: src/UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L113)
```solidity
113:   bytes32 public constant ALTER_BENEFICIARY_TYPEHASH = keccak256(
114:     "AlterBeneficiary(uint256 depositId,address newBeneficiary,address depositor,uint256 nonce)"
```

[File: src/UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L106)
```solidity
106:   bytes32 public constant STAKE_MORE_TYPEHASH =
107:     keccak256("StakeMore(uint256 depositId,uint256 amount,address depositor,uint256 nonce)");
```


[File: src/UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L117)
```solidity
117:   bytes32 public constant WITHDRAW_TYPEHASH =
118:     keccak256("Withdraw(uint256 depositId,uint256 amount,address depositor,uint256 nonce)");
```

[File: src/UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L120)
```solidity
120:   bytes32 public constant CLAIM_REWARD_TYPEHASH =
121:     keccak256("ClaimReward(address beneficiary,uint256 nonce)");
```

At lines 102-103, 109-110, 113-114, `keccak256` is in the same line as the constant:
```
 bytes32 public constant ***_TYPEHASH = keccak256(
```

while at lines 106-107, 117-118, 120-121, `keccak256` is moved to the next line:
```
bytes32 public constant ***_TYPEHASH =
    keccak256(...)
```

Using two different coding-styles might be a little confusing to the code-reader. Our recommendation is to stick to a one coding style and use it across the whole code-base.
For the demonstrated example, much better idea would be to always use `keccak256` in the same line (the same as it's done at lines 102-103, 109-110, 113-114).
The additional advantage of this style would be decreasing the lines' length of the TYPEHASH definitions. Here's the recommended fix:

```
106:   bytes32 public constant STAKE_MORE_TYPEHASH = keccak256(
107:     "StakeMore(uint256 depositId,uint256 amount,address depositor,uint256 nonce)"); // decreases the length of this line

[...]

117:   bytes32 public constant WITHDRAW_TYPEHASH = keccak256(
118:     "Withdraw(uint256 depositId,uint256 amount,address depositor,uint256 nonce)"); // decreases the length of this line

[...]

120:   bytes32 public constant CLAIM_REWARD_TYPEHASH = keccak256(
121:     "ClaimReward(address beneficiary,uint256 nonce)"); // decreases the length of this line
```

Sticking to one style increases the code readability, thus it's highly recommended.


# [11] Typos

**File:** `V3FactoryOwner.sol`

[File: src/V3FactoryOwner.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L139)
```solidity
139:   /// @param _feeProtocol0 The fee protocol 0 param to forward to the pool.
140:   /// @param _feeProtocol1 The fee protocol 1 parm to forward to the pool.
```

` The fee protocol 1 parm` should be changed to ` The fee protocol 1 param`.

# [12] Incorrect punctuation

**Files:** `DelegationSurrogate.sol`, `UniStaker.sol`, `IUniswapV3FactoryOwnerActions.sol`

It some places, there was detected that character `—` instead of white-space is being used:

[File: src/UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L601)
```solidity
601:   /// @notice Internal method which finds the existing surrogate contract—or deploys a new one if
602:   /// none exists—for a given delegatee.
```

`surrogate contract—or deploys` should be changed to `surrogate contract or deploys`.
`none exists—for a given delegatee` should be changed to `none exists for a given delegatee`.

[File: src/DelegationSurrogate.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/DelegationSurrogate.sol#L15)
```solidity
15: /// depositor's tokens to the appropriate  surrogate—or deploys it on their behalf—users can retain
16: /// their governance rights.
```

`surrogate—or deploys it on their behalf—users` should be changed to `surrogate or deploys it on their behalf users`.

Moreover, double white-space has been detected before word `surrogate`:
`appropriate  surrogate` should be changed to `appropriate surrogate`.


[File: src/interfaces/IUniswapV3FactoryOwnerActions.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/IUniswapV3FactoryOwnerActions.sol#L22)
```solidity
22:   /// @param fee The fee amount to enable, denominated in hundredths of a bip (i.e. 1e-6)
```

According to American style guides, `i.e.` should be followed by comma, thus `i.e. 1e-6` should be changed to `i.e., 1e-6`
