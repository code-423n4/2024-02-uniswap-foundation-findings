# QA Report

## Summary

| ID | Description | Severity |
| :-: | - | :-: |
| [L-01](#l-01) | Ambiguity among the actual fully claimable fee amount | Low |
| [L-02](#l-02) | Zero-Fee claiming in `V3FactoryOwner::claimFees()` | Low |
| [L-03](#l-03) | Pumping deposit identifiers due to zero-staking | Low |
| [L-04](#l-04) | Lack of ability to revoke the signatures | Low |
| [L-05](#l-05) | Missing signature deadline | Low |
| [N-01](#n-01) | Inconsistency between logic and code descriptions | Non-Critical |
</br>

## **[L-01] Ambiguity among the actual fully claimable fee amount**<a name="l-01"></a>
#### Location: [V3FactoryOwner::claimFees()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L181-L204)

---

In [V3FactoryOwner::claimFees()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L181-L204), when the claimer performs partial claims (where the requested amount is less than `UniswapV3Pool.protocolFees.token(0||1)`), they receive the exact specified amount.

However, for full claims, if the requested amount matches the queried protocol fee from [UniswapV3Pool.protocolFees](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#L87), the transaction reverts. This is because in [UniswapV3Pool::collectProtocol()](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#L848-L868), the maximum fee to collect is always `protocolFees.token(0||1) - 1`. Therefore, claimers must specify the precise amount (`protocolFees.token(0||1) - 1`) rather than the full protocol fee, which can be queried beforehand.

This may confuse claimers expecting to claim the entire protocol fee.

*There is 1 instance(s) of this issue:*

```solidity
File: V3FactoryOwner.sol
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
-->     if (_amount0 < _amount0Requested || _amount1 < _amount1Requested) {
            revert V3FactoryOwner__InsufficientFeesCollected();
        }
        emit FeesClaimed(address(_pool), msg.sender, _recipient, _amount0, _amount1);
        return (_amount0, _amount1);
        }

        /// @notice Ensures the msg.sender is the contract admin and reverts otherwise.
        /// @dev Place inside external methods to make them admin-only.
        function _revertIfNotAdmin() internal view {
        if (msg.sender != admin) revert V3FactoryOwner__Unauthorized();
    }
```

* [UniswapV3Pool::collectProtocol()](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#L848-L868) from [v3-core](https://github.com/Uniswap/v3-core)
```solidity
Sources: /v3-core/blob/main/contracts/UniswapV3Pool.sol
    /// @inheritdoc IUniswapV3PoolOwnerActions
    function collectProtocol(
        address recipient,
        uint128 amount0Requested,
        uint128 amount1Requested
    ) external override lock onlyFactoryOwner returns (uint128 amount0, uint128 amount1) {
        amount0 = amount0Requested > protocolFees.token0 ? protocolFees.token0 : amount0Requested;
        amount1 = amount1Requested > protocolFees.token1 ? protocolFees.token1 : amount1Requested;

        if (amount0 > 0) {
-->         if (amount0 == protocolFees.token0) amount0--; // ensure that the slot is not cleared, for gas savings
            protocolFees.token0 -= amount0;
            TransferHelper.safeTransfer(token0, recipient, amount0);
        }
        if (amount1 > 0) {
-->         if (amount1 == protocolFees.token1) amount1--; // ensure that the slot is not cleared, for gas savings
            protocolFees.token1 -= amount1;
            TransferHelper.safeTransfer(token1, recipient, amount1);
        }

        emit CollectProtocol(msg.sender, recipient, amount0, amount1);
    }
```

---

## **[L-02] Zero-Fee claiming in `V3FactoryOwner::claimFees()`**<a name="l-02"></a>
#### Location: [V3FactoryOwner::claimFees()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L181-L204)

---

The [V3FactoryOwner::claimFees()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L181-L204) **allows claimers to request both fees outputs as 0** (`_amount0Requested, _amount1Requested`).

This behavior triggers reward notifications `REWARD_RECEIVER.notifyRewardAmount(payoutAmount);` to the reward receiver without any fees being collected in return. 

*There is 1 instance(s) of this issue:*

```solidity
File: V3FactoryOwner.sol
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

        /// @notice Ensures the msg.sender is the contract admin and reverts otherwise.
        /// @dev Place inside external methods to make them admin-only.
        function _revertIfNotAdmin() internal view {
        if (msg.sender != admin) revert V3FactoryOwner__Unauthorized();
    }
```

---

## **[L-03] Pumping deposit identifiers due to zero-staking** <a name="l-03"></a>
#### Location: 
* [UniStaker::_stake()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L638-L664)
* [UniStaker::stake()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L256-L261)
* [UniStaker::stake()[overload]](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L271-L276)
* [UniStaker::permitAndStake()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L292-L303)
* [UniStaker::stakeOnBehalf()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L315-L334)

---

The staking functionalities allow depositors to stake 0 amounts, resulting in the generation of a deposit identifier for each new staking, even when no actual stake is made.

*There is 2 instance(s) of this issue:*

```solidity
File: UniStaker.sol
    function _stake(address _depositor, uint256 _amount, address _delegatee, address _beneficiary)
        internal
        returns (DepositIdentifier _depositId)
    {
        _revertIfAddressZero(_delegatee);
        _revertIfAddressZero(_beneficiary);

        _checkpointGlobalReward();
        _checkpointReward(_beneficiary);

        DelegationSurrogate _surrogate = _fetchOrDeploySurrogate(_delegatee);
-->      _depositId = _useDepositId();

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
```solidity
File: UniStaker.sol
    function _useDepositId() internal returns (DepositIdentifier _depositId) {
        _depositId = nextDepositId;
        nextDepositId = DepositIdentifier.wrap(DepositIdentifier.unwrap(_depositId) + 1);
    }
```

---

## **[L-04] Lack of ability to revoke the signatures** <a name="l-04"></a>
#### Location: 
* [OpenZeppelin::Nonces{}](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Nonces.sol)
* [UniStaker::stakeOnBehalf()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L315-L334)
* [UniStaker::stakeMoreOnBehalf()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L382-L402)
* [UniStaker::alterDelegateeOnBehalf()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L423-L445)
* [UniStaker::alterBeneficiaryOnBehalf()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L466-L492)
* [UniStaker::withdrawOnBehalf()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L512-L532)
* [UniStaker::claimRewardOnBehalf()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L544-L553)

---

**Signers lack the ability to revoke their signatures once signed.**

This means that even if a signer has a legitimate reason to cancel a signature (e.g., changed circumstances), the signed message remains valid and can be used within the [UniStaker](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol) contract.

*There is 1 instance(s) of this issue:*
```solidity
File: UniStaker.sol
    contract UniStaker is INotifiableRewardReceiver, Multicall, EIP712, Nonces {
        // SNIPPED
    }
```

```solidity
Source: OpenZeppelin::Nonces.sol
    abstract contract Nonces {
        /**
        * @dev The nonce used for an `account` is not the expected current nonce.
        */
        error InvalidAccountNonce(address account, uint256 currentNonce);

        mapping(address account => uint256) private _nonces;

        /**
        * @dev Returns the next unused nonce for an address.
        */
        function nonces(address owner) public view virtual returns (uint256) {
            return _nonces[owner];
        }

        /**
        * @dev Consumes a nonce.
        *
        * Returns the current value and increments nonce.
        */
        function _useNonce(address owner) internal virtual returns (uint256) {
            // For each account, the nonce has an initial value of 0, can only be incremented by one, and cannot be
            // decremented or reset. This guarantees that the nonce never overflows.
            unchecked {
                // It is important to do x++ and not ++x here.
                return _nonces[owner]++;
            }
        }

        /**
        * @dev Same as {_useNonce} but checking that `nonce` is the next valid for `owner`.
        */
        function _useCheckedNonce(address owner, uint256 nonce) internal virtual {
            uint256 current = _useNonce(owner);
            if (nonce != current) {
                revert InvalidAccountNonce(owner, current);
            }
        }
    }
```

---

## **[L-05] Missing signature deadline**<a name="l-05"></a>
#### Location: 
* [UniStaker::stakeOnBehalf()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L315-L334)
* [UniStaker::stakeMoreOnBehalf()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L382-L402)
* [UniStaker::alterDelegateeOnBehalf()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L423-L445)
* [UniStaker::alterBeneficiaryOnBehalf()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L466-L492)
* [UniStaker::withdrawOnBehalf()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L512-L532)
* [UniStaker::claimRewardOnBehalf()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L544-L553)

---

There is no specific deadline for signatures. Once a signature is authenticated by the signer/depositor, it can be supplied to any of the functions listed above in the [UniStaker](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol) contract at any subsequent time.

Consequently, signed signatures remain valid until the signer's nonce is increased, allowing them to be used in any future interaction.

*There is 6 instance(s) of this issue:*

```solidity
File: UniStaker.sol
    function stakeOnBehalf(
        uint256 _amount,
        address _delegatee,
        address _beneficiary,
        address _depositor,
        bytes memory _signature
    ) external returns (DepositIdentifier _depositId) {
        _revertIfSignatureIsNotValidNow(
        _depositor,
        _hashTypedDataV4(
            keccak256(
            abi.encode(
                STAKE_TYPEHASH, _amount, _delegatee, _beneficiary, _depositor, _useNonce(_depositor)
            )
            )
        ),
        _signature
        );
        _depositId = _stake(_depositor, _amount, _delegatee, _beneficiary);
    }
```
```solidity
File: UniStaker.sol
    function stakeMoreOnBehalf(
        DepositIdentifier _depositId,
        uint256 _amount,
        address _depositor,
        bytes memory _signature
    ) external {
        Deposit storage deposit = deposits[_depositId];
        _revertIfNotDepositOwner(deposit, _depositor);

        _revertIfSignatureIsNotValidNow(
        _depositor,
        _hashTypedDataV4(
            keccak256(
            abi.encode(STAKE_MORE_TYPEHASH, _depositId, _amount, _depositor, _useNonce(_depositor))
            )
        ),
        _signature
        );

        _stakeMore(deposit, _depositId, _amount);
    }
```
```solidity
File: UniStaker.sol
    function alterDelegateeOnBehalf(
        DepositIdentifier _depositId,
        address _newDelegatee,
        address _depositor,
        bytes memory _signature
    ) external {
        Deposit storage deposit = deposits[_depositId];
        _revertIfNotDepositOwner(deposit, _depositor);

        _revertIfSignatureIsNotValidNow(
        _depositor,
        _hashTypedDataV4(
            keccak256(
            abi.encode(
                ALTER_DELEGATEE_TYPEHASH, _depositId, _newDelegatee, _depositor, _useNonce(_depositor)
            )
            )
        ),
        _signature
        );

        _alterDelegatee(deposit, _depositId, _newDelegatee);
    }
```
```solidity
File: UniStaker.sol
    function alterBeneficiaryOnBehalf(
        DepositIdentifier _depositId,
        address _newBeneficiary,
        address _depositor,
        bytes memory _signature
    ) external {
        Deposit storage deposit = deposits[_depositId];
        _revertIfNotDepositOwner(deposit, _depositor);

        _revertIfSignatureIsNotValidNow(
        _depositor,
        _hashTypedDataV4(
            keccak256(
            abi.encode(
                ALTER_BENEFICIARY_TYPEHASH,
                _depositId,
                _newBeneficiary,
                _depositor,
                _useNonce(_depositor)
            )
            )
        ),
        _signature
        );

        _alterBeneficiary(deposit, _depositId, _newBeneficiary);
    }
```
```solidity
File: UniStaker.sol
    function withdrawOnBehalf(
        DepositIdentifier _depositId,
        uint256 _amount,
        address _depositor,
        bytes memory _signature
    ) external {
        Deposit storage deposit = deposits[_depositId];
        _revertIfNotDepositOwner(deposit, _depositor);

        _revertIfSignatureIsNotValidNow(
        _depositor,
        _hashTypedDataV4(
            keccak256(
            abi.encode(WITHDRAW_TYPEHASH, _depositId, _amount, _depositor, _useNonce(_depositor))
            )
        ),
        _signature
        );

        _withdraw(deposit, _depositId, _amount);
    }
```
```solidity
File: UniStaker.sol
    function claimRewardOnBehalf(address _beneficiary, bytes memory _signature) external {
        _revertIfSignatureIsNotValidNow(
        _beneficiary,
        _hashTypedDataV4(
            keccak256(abi.encode(CLAIM_REWARD_TYPEHASH, _beneficiary, _useNonce(_beneficiary)))
        ),
        _signature
        );
        _claimReward(_beneficiary);
    }
```

---

## **[N-01] Inconsistency between logic and code descriptions** <a name="n-01"></a>
#### Location: 
* [V3FactoryOwner::claimFees()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L192)
* [UniStaker::stakeOnBehalf()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L306)
* [UniStaker::stakeMoreOnBehalf()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L376-L377)
* [UniStaker::alterDelegateeOnBehalf()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L420-L421)
* [UniStaker::alterBeneficiaryOnBehalf()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L463-L464)
* [UniStaker::withdrawOnBehalf()](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L509-L510)

---

There are inconsistencies between the logic and code descriptions in the function listings above.

Specifically, the inconsistencies occur in each function shown below:

*There is 6 instance(s) of this issue:*

```diff
File: V3FactoryOwner.sol
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

-:192      // Protect the caller from receiving less than requested. See `collectProtocol` for context.
+:192     // Protect the recipient from receiving less than requested. See `collectProtocol` for context.
        if (_amount0 < _amount0Requested || _amount1 < _amount1Requested) {
        revert V3FactoryOwner__InsufficientFeesCollected();
        }
        emit FeesClaimed(address(_pool), msg.sender, _recipient, _amount0, _amount1);
        return (_amount0, _amount1);
    }

    /// @notice Ensures the msg.sender is the contract admin and reverts otherwise.
    /// @dev Place inside external methods to make them admin-only.
    function _revertIfNotAdmin() internal view {
        if (msg.sender != admin) revert V3FactoryOwner__Unauthorized();
    }
```
```diff
File: UniStaker.sol
    /// @notice Stake tokens to a new deposit on behalf of a user, using a signature to validate the
-:306    /// user's intent. The caller must pre-approve the staking contract to spend at least the
+:306    /// user's intent. The signer/depositor must pre-approve the staking contract to spend at least the
    /// would-be staked amount of the token.
    /// @param _amount Quantity of the staking token to stake.
    /// @param _delegatee Address to assign the governance voting weight of the staked tokens.
    /// @param _beneficiary Address that will accrue rewards for this stake.
    /// @param _depositor Address of the user on whose behalf this stake is being made.
    /// @param _signature Signature of the user authorizing this stake.
    /// @return _depositId Unique identifier for this deposit.
    /// @dev Neither the delegatee nor the beneficiary may be the zero address.
    function stakeOnBehalf(
        uint256 _amount,
        address _delegatee,
        address _beneficiary,
        address _depositor,
        bytes memory _signature
    ) external returns (DepositIdentifier _depositId) { .. }
```
```diff
File: UniStaker.sol
    /// @notice Add more staking tokens to an existing deposit on behalf of a user, using a signature
-:376    /// to validate the user's intent. A staker should call this method when they have an existing
+:376    /// to validate the user's intent. the user on whose behalf this stake should have an existing
    /// deposit, and wish to stake more while retaining the same delegatee and beneficiary.
    /// @param _depositId Unique identifier of the deposit to which stake will be added.
    /// @param _amount Quantity of stake to be added.
    /// @param _depositor Address of the user on whose behalf this stake is being made.
    /// @param _signature Signature of the user authorizing this stake.
    function stakeMoreOnBehalf(
        DepositIdentifier _depositId,
        uint256 _amount,
        address _depositor,
        bytes memory _signature
    ) external { .. }
```
```diff
File: UniStaker.sol
    /// @notice For an existing deposit, change the address to which governance voting power is
    /// assigned on behalf of a user, using a signature to validate the user's intent.
    /// @param _depositId Unique identifier of the deposit which will have its delegatee altered.
    /// @param _newDelegatee Address of the new governance delegate.
-:420    /// @param _depositor Address of the user on whose behalf this stake is being made.
+:420    /// @param _depositor Address of the user on whose behalf this delegate alteration is being made.
-:421    /// @param _signature Signature of the user authorizing this stake.
+:421    /// @param _signature Signature of the user authorizing this delegate alteration.
    /// @dev The new delegatee may not be the zero address.
    function alterDelegateeOnBehalf(
        DepositIdentifier _depositId,
        address _newDelegatee,
        address _depositor,
        bytes memory _signature
    ) external { .. }
```
```diff
File: UniStaker.sol
    /// @notice For an existing deposit, change the beneficiary to which staking rewards are
    /// accruing on behalf of a user, using a signature to validate the user's intent.
    /// @param _depositId Unique identifier of the deposit which will have its beneficiary altered.
    /// @param _newBeneficiary Address of the new rewards beneficiary.
-:463    /// @param _depositor Address of the user on whose behalf this stake is being made.
+:463    /// @param _depositor Address of the user on whose behalf this beneficiary alteration is being made.
-:464    /// @param _signature Signature of the user authorizing this stake.
+:464    /// @param _signature Signature of the user authorizing this beneficiary alteration.
    /// @dev The new beneficiary may not be the zero address.
    function alterBeneficiaryOnBehalf(
        DepositIdentifier _depositId,
        address _newBeneficiary,
        address _depositor,
        bytes memory _signature
    ) external { .. }
```
```diff
File: UniStaker.sol
    /// @notice Withdraw staked tokens from an existing deposit on behalf of a user, using a
    /// signature to validate the user's intent.
    /// @param _depositId Unique identifier of the deposit from which stake will be withdrawn.
    /// @param _amount Quantity of staked token to withdraw.
-:509    /// @param _depositor Address of the user on whose behalf this stake is being made.
+:509    /// @param _depositor Address of the user on whose behalf this withdrawal is being made.
-:510    /// @param _signature Signature of the user authorizing this stake.
+:510    /// @param _signature Signature of the user authorizing this withdrawal.
    /// @dev Stake is withdrawn to the deposit owner's account.
    function withdrawOnBehalf(
        DepositIdentifier _depositId,
        uint256 _amount,
        address _depositor,
        bytes memory _signature
    ) external { .. }
```

---