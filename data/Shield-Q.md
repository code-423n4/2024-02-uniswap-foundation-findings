**Overview**

Risk Rating | Number of issues
--- | ---
Low Risk | 10
Informational | 2

**Table of Contents**

- [1. Low Risk Concerns](#1-low-risk-concerns)
  - [1.1. ERC1271 signature verifications will not work due UNI's permit function](#11-erc1271-signature-verifications-will-not-work-due-unis-permit-function)
  - [1.2. In several functions, the signatures are valid forever (lack of `deadline`)](#12-in-several-functions-the-signatures-are-valid-forever-lack-of-deadline)
  - [1.3. Users can be tricked to use an arbitrary `address _pool` in `claimFees()`](#13-users-can-be-tricked-to-use-an-arbitrary-address-_pool-in-claimfees)
  - [1.4. `stakeOnBehalf()` can be called with `amount == 0` on any `_beneficiary` and `_delegatee`](#14-stakeonbehalf-can-be-called-with-amount--0-on-any-_beneficiary-and-_delegatee)
  - [1.5. `deadline == 0` not handled in permit functions](#15-deadline--0-not-handled-in-permit-functions)
  - [1.6. Discrepancy between `stakeOnBehalf()`'s Natspec comment and behavior](#16-discrepancy-between-stakeonbehalfs-natspec-comment-and-behavior)
  - [1.7. `setRewardNotifier()` isn't protected like `setAdmin`](#17-setrewardnotifier-isnt-protected-like-setadmin)
  - [1.8. `alterBeneficiary()` can be called with the old `deposit.beneficiary`](#18-alterbeneficiary-can-be-called-with-the-old-depositbeneficiary)
  - [1.9. `_alterDelegatee()` can be called with the old `deposit.delegatee`](#19-_alterdelegatee-can-be-called-with-the-old-depositdelegatee)
  - [1.10. `stakeMore()` with 0 is valid](#110-stakemore-with-0-is-valid)
- [2. Informational Concerns](#2-informational-concerns)
  - [2.1. Consider indexing the depositor (owner) in `event StakeDeposited`](#21-consider-indexing-the-depositor-owner-in-event-stakedeposited)
  - [2.2. Replace `_revertIfNotAdmin` with a modifier](#22-replace-_revertifnotadmin-with-a-modifier)

# 1. Low Risk Concerns

## 1.1. ERC1271 signature verifications will not work due UNI's permit function

The `permitAndStake` function and `permitAndStakeMore` functions will not work for ERC1271 smart contract signature verification since the UNI permit function only supports EOA signatures.

But the `stakeOnBehalf`, `stakeMoreOnBehalf`, `alterDelegateeOnBehalf`, `alterBeneficiaryOnBehalf`, `withdrawOnBehalf` and `claimRewardOnBehalf` support the ERC1271 signatures, thus enabling the smart contract wallets to work with the protocol.

Hence the users should be informed of this difference. Else a normal user with a smart contract wallet might try to call the `permitAndStake` with a valid signature but the transaction will revert.

## 1.2. In several functions, the signatures are valid forever (lack of `deadline`)

In the `stakeOnBehalf()` function, it's recommended to have a deadline to ensure that the signature hasn't expired for the transaction to execute successfully.

Currently there is no deadline and the **signature is valid forever**. Hence if the signer is a EOA and needs to revoke the transaction he is unable to do that until the signature is used. Hence recommended to include the deadline in the data hash to be signed.

The same issue exists in the `stakeMoreOnBehalf`, `alterDelegateeOnBehalf`, `withdrawOnBehalf` and `claimRewardOnBehalf` functions

## 1.3. Users can be tricked to use an arbitrary `address _pool` in `claimFees()`

It's possible to input any arbitrary `_pool` in `claimFees()`.
If it's a malicious one, the only effect seem to be losing an amount of `payoutAmount` of `PAYOUT_TOKEN` from the caller to the `REWARD_RECEIVER` (basically, a phishing attack without profit, only making the users lose funds).

Instead of letting any caller input an arbitrary pool, it'd be great to instead change the input to the pair of tokens (`token0` and `token1`) so that the real `pool`'s address could be fetched from the `Factory`

## 1.4. `stakeOnBehalf()` can be called with `amount == 0` on any `_beneficiary` and `_delegatee`

With a malicious `depositor` verifying any ERC1271 signatures, it's possible to call `stakeOnBehalf()` with `amount == 0` and any `_beneficiary` or `_delegatee`.
This could either pollute them or be used as a step in a phishing attack.
Consider validating against `amount == 0`

## 1.5. `deadline == 0` not handled in permit functions

In the `permitAndStake` and `permitAndStakeMore` functions, the `STAKE_TOKEN.permit()` function is called which would call the `permit` function of the `UNI` token.

But if the `deadline` is set to `0` for the above function call, the **transaction will revert** because the `UNI` token's deadline check is as follows:

```solidity
    require(now <= deadline, "Uni::permit: signature expired");
```

It should be noted that the `permit` function of some of the ERC20 Tokens (like [DAI](https://etherscan.io/token/0x6b175474e89094c44da98b954eedeac495271d0f#code)) support the zero deadline as follows:

```solidity
    require(deadline == 0 || now <= deadline, "Uni::permit: signature expired");
```

But this is not the case with the UNI token. Hence, the users may be it is recommended inform the users of this behaviour and request them to use a valid deadline when they are calling the `permitAndStake` and `permitAndStakeMore` functions.
If not, the transactions will revert for `deadline == 0` even if the signature is valid.

## 1.6. Discrepancy between `stakeOnBehalf()`'s Natspec comment and behavior

Please consider the following scenario:

`stakeOnBehalf()` function has the following natspec comment explaining the objective of the function.

```solidity
  /// @notice Stake tokens to a new deposit on behalf of a user, using a signature to validate the
  /// user's intent. The caller must pre-approve the staking contract to spend at least the
  /// would-be staked amount of the token.
```

Notice the `The caller must pre-approve` part. This indicates that the caller is depositing his tokens on behalf of the depositor address.

However, the actual `stakeOnBehalf()` function implementation uses a user-input `address _depositor`:

```solidity
    _depositId = _stake(_depositor, _amount, _delegatee, _beneficiary);
```

The `_stake()` function calls the `_stakeTokenSafeTransferFrom()` function where tokens are transferred from the `depositor` address and not from the `msg.sender`. If the `depositor` has not pre-approved the tokens then the **transaction reverts**.

```solidity
    _stakeTokenSafeTransferFrom(_depositor, address(_surrogate), _amount);
```

Hence **either the natspec comments is wrong or the function implementation is wrong**.

If the `caller` is depositing his tokens on behalf of the `depositor` then the `msg.sender` should also be passed into the `_stake()` function in addition to the `depositor` address (function overloading on `_stake()`) and the `_stakeTokenSafeTransferFrom()` should be called on the `msg.sender`.

Otherwise, the natspec comment needs to be changed to indicate that the `depositor` should approve the staking contract with the tokens. In which case caller is not transferring his own tokens on behalf of the depositor but only executing the deposit operation.

## 1.7. `setRewardNotifier()` isn't protected like `setAdmin`

`_setAdmin()` has a `_revertIfAddressZero()` function.
Therefore, we should have a similar `address(0)` protection on `setRewardNotifier()`.
Indeed, `isRewardNotifier[address(0)] = true;` is a reachable state right now.

## 1.8. `alterBeneficiary()` can be called with the old `deposit.beneficiary`

In the `_alterBeneficiary()` function, it should be checked that `_newBeneficiary != deposit.beneficiary`.

Notice that, in the fuzzed tests, the input is forbidden, as it is known to be implicitly invalid:

- [UniStaker.t.sol#L1801](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/test/UniStaker.t.sol#L1801)

```solidity
vm.assume(_newBeneficiary != address(0) && _newBeneficiary != _firstBeneficiary);
```

Consider explicitly validating against the invalid input so that the function can respect the specs (actually altering the beneficiary).

## 1.9. `_alterDelegatee()` can be called with the old `deposit.delegatee`

In the `_alterDelegatee()` function, it should be checked that `_newDelegatee != deposit.delegatee`.

Notice that, in the fuzzed tests, the input is forbidden, as it is known to be implicitly invalid:

- [UniStaker.t.sol#L1503](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/test/UniStaker.t.sol#L1503)

```solidity
vm.assume(_newDelegatee != address(0) && _newDelegatee != _firstDelegatee);
```

Consider explicitly validating against the invalid input so that the function can respect the specs (actually altering the delegatee).

## 1.10. `stakeMore()` with 0 is valid

We can call `stakeMore()` with `_amount == 0` (the UNI token doesn't revert on transferring 0).

Consider validating against `amount == 0` so that the function can respect the specs (actually staking more).

# 2. Informational Concerns

## 2.1. Consider indexing the depositor (owner) in `event StakeDeposited`

```diff
File: UniStaker.sol
36:   event StakeDeposited(
- 37:     address owner, DepositIdentifier indexed depositId, uint256 amount, uint256 depositBalance
+ 37:     address indexed owner, DepositIdentifier indexed depositId, uint256 amount, uint256 depositBalance
38:   );
```

## 2.2. Replace `_revertIfNotAdmin` with a modifier

`_revertIfNotAdmin()` is used in `setAdmin()` and `setRewardNotifier()` instead of a modifier.

Consider using a real modifier instead.
