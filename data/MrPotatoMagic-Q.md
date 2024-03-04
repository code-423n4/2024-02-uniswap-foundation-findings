# Quality Assurance Report

| ID     | Issues                                                                                                                                                                                             | Instances |
|--------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------|
| [N-01] | Typo error in comment                                                                                                                                                                              | 2         |
| [N-02] | Missing address(0) checks in constructor [(missed by bot report)](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/bot-report.md#l-03-missing-zero-address-check-in-constructor) | 3         |
| [N-03] | 0 value stakes, withdrawals and claims are allowed                                                                                                                                                 | 3         |
| [N-04] | Self beneficiary and delegatee alterations are allowed                                                                                                                                             | 2         |
| [R-01] | Consider using modifiers for checks instead of functions                                                                                                                                           | 4         |
| [R-02] | Consider providing reminders/warnings on frontend for stale depositIds                                                                                                                             | 1         |
| [R-03] | Consider reverting with a reason instead of underflowing                                                                                                                                           | 1         |

## [N-01] Typo error in comment

[Link](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L170)

The word "the" is used twice in the below statement.
```solidity
File: V3FactoryOwner.sol
172:   /// the pool fees for less than they are valued by "paying" the the payout amount of the payout
173:   /// token.
```

[Link](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L140)

`parm` should be corrected to `param`
```solidity
File: V3FactoryOwner.sol
142:   /// @param _feeProtocol1 The fee protocol 1 parm to forward to the pool.
```

## [N-02] Missing address(0) checks in constructor [(missed by bot report)](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/bot-report.md#l-03-missing-zero-address-check-in-constructor)

[Link to instances](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L98C1-L102C39)

The constructor below is missing zero address checks for `_factory`, `_payoutToken` and `_rewardReceiver`. These instances and contract was not covered by the [bot report [L-03] finding](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/bot-report.md#l-03-missing-zero-address-check-in-constructor).
```solidity
File: V3FactoryOwner.sol
088:   constructor(
089:     address _admin,
090:     IUniswapV3FactoryOwnerActions _factory,
091:     IERC20 _payoutToken,
092:     uint256 _payoutAmount,
093:     INotifiableRewardReceiver _rewardReceiver
094:   ) {
095:     if (_admin == address(0)) revert V3FactoryOwner__InvalidAddress();
096:     if (_payoutAmount == 0) revert V3FactoryOwner__InvalidPayoutAmount();
097: 
099:     admin = _admin;
100:     FACTORY = _factory;
101:     PAYOUT_TOKEN = _payoutToken;
102:     payoutAmount = _payoutAmount;
103:     REWARD_RECEIVER = _rewardReceiver;
104: 
105:     emit AdminSet(address(0), _admin);
106:     emit PayoutAmountSet(0, _payoutAmount);
107:   }
```

## [N-03] 0 value stakes, withdrawals and claims are allowed

There does not seem to be direct risk by allowing 0 value stakes, withdrawals and claims but it would be a good practice to add checks for them as the [SNX Staking Contract does here](https://github.com/Synthetixio/synthetix/blob/13f1011eb9879df7c2f2f3713a95be565daed715/contracts/StakingRewards.sol#L81C1-L95C6).

```solidity
File: UniStaker.sol
746:     function _stake(
747:         address _depositor,
748:         uint256 _amount,
749:         address _delegatee,
750:         address _beneficiary
751:     ) internal returns (DepositIdentifier _depositId) {
752:         _revertIfAddressZero(_delegatee);
753:         _revertIfAddressZero(_beneficiary);
754: 
755:         _checkpointGlobalReward();
756:         _checkpointReward(_beneficiary);
757: 
758:         DelegationSurrogate _surrogate = _fetchOrDeploySurrogate(_delegatee);
759:         _depositId = _useDepositId();
760: 
761:         totalStaked += _amount;
762:         depositorTotalStaked[_depositor] += _amount;
763:         earningPower[_beneficiary] += _amount;
764:         deposits[_depositId] = Deposit({
765:             balance: _amount,
766:             owner: _depositor,
767:             delegatee: _delegatee,
768:             beneficiary: _beneficiary
769:         });
770:         _stakeTokenSafeTransferFrom(_depositor, address(_surrogate), _amount);
771:         emit StakeDeposited(_depositor, _depositId, _amount, _amount);
772:         emit BeneficiaryAltered(_depositId, address(0), _beneficiary);
773:         emit DelegateeAltered(_depositId, address(0), _delegatee);
774:     }

856:     function _withdraw(
857:         Deposit storage deposit,
858:         DepositIdentifier _depositId,
859:         uint256 _amount
860:     ) internal {
861:         _checkpointGlobalReward();
862:         _checkpointReward(deposit.beneficiary);
863: 
864:         deposit.balance -= _amount; // overflow prevents withdrawing more than balance
865:         totalStaked -= _amount;
866:         depositorTotalStaked[deposit.owner] -= _amount;
867:         earningPower[deposit.beneficiary] -= _amount;
868:         _stakeTokenSafeTransferFrom(
869:             address(surrogates[deposit.delegatee]),
870:             deposit.owner,
871:             _amount
872:         );
873:         
875:         emit StakeWithdrawn(_depositId, _amount, deposit.balance);
876:     }

883:     function _claimReward(address _beneficiary) internal {
884:         _checkpointGlobalReward();
885:         _checkpointReward(_beneficiary);
886: 
887:         uint256 _reward = unclaimedRewardCheckpoint[_beneficiary];
888:         if (_reward == 0) return;
889:         unclaimedRewardCheckpoint[_beneficiary] = 0;
890:         emit RewardClaimed(_beneficiary, _reward);
891: 
892:         SafeERC20.safeTransfer(REWARD_TOKEN, _beneficiary, _reward);
```

## [N-04] Self beneficiary and delegatee alterations are allowed

[Link](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L704)
[Link](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L688)

The function below does not check if the new beneficiary is already the old beneficiary and revert with a helper message. This does not pose any risk but it could save user's some gas by reverting early and would also provide them with a helper message in case they think the beneficiary was not correctly set previously, This would particularly be useful for depositIds that might've gone stale previously and now the user tries to set the beneficiary to the same address again. The same issue applies for the _alterDelegatee() function which has been linked above.
```solidity
File: UniStaker.sol
832:     function _alterBeneficiary(
833:         Deposit storage deposit,
834:         DepositIdentifier _depositId,
835:         address _newBeneficiary
836:     ) internal {
837:       
838:         _revertIfAddressZero(_newBeneficiary);
839:         _checkpointGlobalReward();
840:         _checkpointReward(deposit.beneficiary);
841:         earningPower[deposit.beneficiary] -= deposit.balance;
842: 
843:         _checkpointReward(_newBeneficiary);
844:         emit BeneficiaryAltered(
845:             _depositId,
846:             deposit.beneficiary,
847:             _newBeneficiary
848:         );
849:         deposit.beneficiary = _newBeneficiary;
850:         earningPower[_newBeneficiary] += deposit.balance;
851:     }
```

## [R-01] Consider using modifiers for checks instead of functions

[Link to instances](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L776C1-L809C4)

Instead of using functions, consider using modifiers when there are common checks involved across functions. This would improve the code readability. maintainability. 
```solidity
File: UniStaker.sol
922:     /// @notice Internal helper method which reverts UniStaker__Unauthorized if the message sender is
923:     /// not the admin.
924:     function _revertIfNotAdmin() internal view {
925:         if (msg.sender != admin)
926:             revert UniStaker__Unauthorized("not admin", msg.sender);
927:     }
928: 
929:     /// @notice Internal helper method which reverts UniStaker__Unauthorized if the alleged owner is
930:     /// not the true owner of the deposit.
931:     /// @param deposit Deposit to validate.
932:     /// @param owner Alleged owner of deposit.
933:     function _revertIfNotDepositOwner(
934:         Deposit storage deposit,
935:         address owner
936:     ) internal view {
937:         if (owner != deposit.owner)
938:             revert UniStaker__Unauthorized("not owner", owner);
939:     }
940: 
941:     /// @notice Internal helper method which reverts with UniStaker__InvalidAddress if the account in
942:     /// question is address zero.
943:     /// @param _account Account to verify.
944:     function _revertIfAddressZero(address _account) internal pure {
945:         if (_account == address(0)) revert UniStaker__InvalidAddress();
946:     }
947: 
948:     /// @notice Internal helper method which reverts with UniStaker__InvalidSignature if the signature
949:     /// is invalid.
950:     /// @param _signer Address of the signer.
951:     /// @param _hash Hash of the message.
952:     /// @param _signature Signature to validate.
953:     function _revertIfSignatureIsNotValidNow(
954:         address _signer,
955:         bytes32 _hash,
956:         bytes memory _signature
957:     ) internal view {
958:         bool _isValid = SignatureChecker.isValidSignatureNow(
959:             _signer,
960:             _hash,
961:             _signature
962:         );
963:         if (!_isValid) revert UniStaker__InvalidSignature();
964:     }
```

## [R-02] Consider providing reminders/warnings on frontend for stale depositIds

[Link](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L723)

In the withdraw() function below, when a user fully withdraws their stake, the delegatee and beneficiary in the `Deposit` struct is not cleared. The issue is that when the user comes back to stake to a stale depositId after some time in the future, the user might not be knowing the depositId is using the old addresses configured for delegatee and beneficiary. Not clearing these addresses seems like a design choice (i.e. only the user can alter these), but this does pose a security risk if users are unaware of the old addresses.

**Solution:**
If this is not an intended design choice, consider clearing the delegatee and beneficiary addresses on full withdrawals. If this is intended desgin, as a safety mechanism, consider providing reminders/warnings on depositIds that go stale after certain period of time. 
```solidity
File: UniStaker.sol
855:     function _withdraw(
856:         Deposit storage deposit,
857:         DepositIdentifier _depositId,
858:         uint256 _amount
859:     ) internal {
860:         _checkpointGlobalReward();
861:         _checkpointReward(deposit.beneficiary);
862: 
863:         deposit.balance -= _amount; // overflow prevents withdrawing more than balance 
864:         totalStaked -= _amount;
865:         depositorTotalStaked[deposit.owner] -= _amount;
866:         earningPower[deposit.beneficiary] -= _amount;
867:         _stakeTokenSafeTransferFrom(
868:             address(surrogates[deposit.delegatee]),
869:             deposit.owner,
870:             _amount
871:         );
875:         emit StakeWithdrawn(_depositId, _amount, deposit.balance);
876:     }
```

## [R-03] Consider reverting with a reason instead of underflowing

[Link](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L729)

Currently the withdraw() function reverts with an underflow when a user tries to withdraw more than what they had staked. This is not a good practice and it is recommended to revert with a reason (i.e. not enough staked).
```solidity
File: UniStaker.sol
856:     function _withdraw(
857:         Deposit storage deposit,
858:         DepositIdentifier _depositId,
859:         uint256 _amount
860:     ) internal {
861:         _checkpointGlobalReward();
862:         _checkpointReward(deposit.beneficiary);
863: 
864:         deposit.balance -= _amount; // overflow prevents withdrawing more than balance 
```