# UniStaker QA Report

| ID            | Title                                                                                       | Severity |
| ------------- | ------------------------------------------------------------------------------------------- | -------- |
| [L-01](#l-01) | A signer can't cancel his signature                                                         | Low      |
| [L-02](#l-02) | `rewardPerTokenAccumulated()` function can be manipulated by manipulating the `totalStaked` | Low      |
| [L-03](#l-03) | A flaw in the signature verification design                                                 | Low      |
| [L-04](#l-04) | `_claimReward` function should have a minimum claimed amount parameter                      | Low      |

## [L-01] A signer can't cancel his signature <a id="l-01" ></a>

## Details

The protocol has functionalities to enable other users to do some actions like staking,withdrawing and rewards claiming onbehalf of other users by providing a valid signature, but there's no functionality to cancel signatures if a signer wants to invalidate his signatures.

## Recommendation

Add a function to cancel signatures by increasing the signer nonce.

## [L-02] `rewardPerTokenAccumulated()` function can be manipulated by manipulating the `totalStaked` <a id="l-02" ></a>

## Details

`rewardPerTokenAccumulated()` function is used to update the global reward per token accumulator (`rewardPerTokenAccumulatedCheckpoint`):

```javascript
    function rewardPerTokenAccumulated() public view returns (uint256) {
        if (totalStaked == 0) return rewardPerTokenAccumulatedCheckpoint;
        return
            rewardPerTokenAccumulatedCheckpoint +
            (scaledRewardRate *
                (lastTimeRewardDistributed() - lastCheckpointTime)) /
            totalStaked;
    }
```

and as can be noticed; the `totalStaked` variable is being used to calculate this value, where `totalStaked` represents the current amount of UNI tokens being staked/deposited, and this value is updated whenever there's a stake or withdraw made, so any malicious actor can manipulate the calculation of `rewardPerTokenAccumulatedCheckpoint` for his favor/or against others favors by depositing to increase the `totalStaked` ; hence decreasing the updated amount of `rewardPerTokenAccumulatedCheckpoint`, or by withdrawing his deposit and decreasing the `totalStaked` so the updated amount of `rewardPerTokenAccumulatedCheckpoint` will be increased.

## Recommendation

Implement a mechanism to update the `rewardPerTokenAccumulatedCheckpoint` based on the `totalStaked` of the previous block.

## [L-03] A flaw in the signature verification design <a id="l-03" ></a>

## Details

- The protocol has functionalities to enable other users to do some actions like staking,withdrawing and rewards claiming onbehalf of other users by providing a valid signature, where the signature is validated/invalidate based on a hash created with the signer nonce included, for example:

  ```javascript
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
                          STAKE_TYPEHASH,
                          _amount,
                          _delegatee,
                          _beneficiary,
                          _depositor,
                          _useNonce(_depositor)
                      )
                  )
              ),
              _signature
          );
          _depositId = _stake(_depositor, _amount, _delegatee, _beneficiary);
      }
  ```

  where the signature is validated by `_revertIfSignatureIsNotValidNow()`:

  ```javascript
      function _revertIfSignatureIsNotValidNow(
          address _signer,
          bytes32 _hash,
          bytes memory _signature
      ) internal view {
          bool _isValid = SignatureChecker.isValidSignatureNow(
              _signer,
              _hash,
              _signature
          );
          if (!_isValid) revert UniStaker__InvalidSignature();
      }
  ```

- As can be noticed, the signer nonce is being used in the signature hash after being increased, and all onbehalf functionalities are using the signer nonce to validate the signature : `stakeOnBehalf()`, `stakeMoreOnBehalf()`, `alterDelegateeOnBehalf()`, `alterBeneficiaryOnBehalf()`, `withdrawOnBehalf()` and `claimRewardOnBehalf()` functions.

- So for example: if a signer signs a signature for `stakeOnBehalf()` function with a nonce of 2, and this function hasn't been called by anyone; the signer nonce will be stuck at 2, and no other signatures can be used after then unless this signature is used.

## Recommendation

Implement another mechanism that invalidate the signature after a deadline is passed instead of using the signer nonce.

## [L-04] `_claimReward` function should have a minimum claimed amount parameter <a id="l-04" ></a>

## Details

`_claimReward` function will be called when beneficiary claims his entitled rewrds,where the global rewards parameters (`rewardPerTokenAccumulatedCheckpoint` & `lastCheckpointTime`) will be updated via `_checkpointGlobalReward()`, then the beneficiary rewards paramteres will be updated next via `_checkpointReward(_beneficiary)`:

    ```javascript
    function _claimReward(address _beneficiary) internal {
            _checkpointGlobalReward();
            _checkpointReward(_beneficiary);

            uint256 _reward = unclaimedRewardCheckpoint[_beneficiary];
            if (_reward == 0) return;

            unclaimedRewardCheckpoint[_beneficiary] = 0;
            emit RewardClaimed(_beneficiary, _reward);

            SafeERC20.safeTransfer(REWARD_TOKEN, _beneficiary, _reward);
        }
    ```

    ```javascript
        function _checkpointReward(address _beneficiary) internal {
            unclaimedRewardCheckpoint[_beneficiary] = unclaimedReward(_beneficiary);

            beneficiaryRewardPerTokenCheckpoint[
                _beneficiary
            ] = rewardPerTokenAccumulatedCheckpoint;
        }
    ```

where:

    ```javascript
        function unclaimedReward(
            address _beneficiary
        ) public view returns (uint256) {
            return
                unclaimedRewardCheckpoint[_beneficiary] +
                (earningPower[_beneficiary] *
                    (rewardPerTokenAccumulated() -
                        beneficiaryRewardPerTokenCheckpoint[_beneficiary])) /
                SCALE_FACTOR;
        }
    ```

- So as can be noticed; the unclaime rewards are heavily influenced by `rewardPerTokenAccumulated() - beneficiaryRewardPerTokenCheckpoint[_beneficiary]`, and the `rewardPerTokenAccumulated()` depends on the `totalStaked`:

  ```javascript
      function rewardPerTokenAccumulated() public view returns (uint256) {
          if (totalStaked == 0) return rewardPerTokenAccumulatedCheckpoint;

          return
              rewardPerTokenAccumulatedCheckpoint +
              (scaledRewardRate *
                  (lastTimeRewardDistributed() - lastCheckpointTime)) /
              totalStaked;
      }
  ```

  so the benficiary claimed rewards would be highly affected by the increase of `totalStaked`, as it will result in lower `rewardPerTokenAccumulatedCheckpoint` increment, and the diiference between `rewardPerTokenAccumulated() - beneficiaryRewardPerTokenCheckpoint[_beneficiary]` would be less, which will result in less rewards claimed by the beneficiary.

## Recommendation

Update `_claimReward` function to have a minimum claimed rewards determined by the beneficiary:

```diff
- function _claimReward(address _beneficiary) internal {
+ function _claimReward(address _beneficiary, uint256 minRewards) internal {
        _checkpointGlobalReward();
        _checkpointReward(_beneficiary);

        uint256 _reward = unclaimedRewardCheckpoint[_beneficiary];
        if (_reward == 0) return;
+       if (_reward < minRewards) revert();

        unclaimedRewardCheckpoint[_beneficiary] = 0;
        emit RewardClaimed(_beneficiary, _reward);

        SafeERC20.safeTransfer(REWARD_TOKEN, _beneficiary, _reward);
    }
```
