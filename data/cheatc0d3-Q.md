#  QA Report: Unistaker

## Summary

- [L-01 Implement Role-Based Access Control](#l-01-implement-role-based-access-control)
- [L-02 Implement Deadline Checks in Time-sensitive Functions](#l-02-implement-deadline-checks-in-time-sensitive-functions)
- [L-03 Validate Non-zero Amounts](#l-03-validate-non-zero-amounts)
- [L-04 Scalability Concerns in Reward Calculation Mechanism of UniStaker Contract](#l-04-scalability-concerns-in-reward-calculation-mechanism-of-unistaker-contract)
- [L-05 Use `abi.encode` instead of `abi.encodePacked` when computing type hashes](#l-05-use-abiencode-instead-of-abiencodepacked-when-computing-type-hashes)
- [L-06 Lack of Slashing Conditions for Staked Assets](#l-06-lack-of-slashing-conditions-for-staked-assets)
- [L-07 Lack of Emergency Withdrawal Mechanisms](#l-07-lack-of-emergency-withdrawal-mechanisms)
- [L-08 Implement Nonce Expiry to better safeguard against replay attacks](#l-08-implement-nonce-expiry-to-better-safeguard-against-replay-attacks)
- [L-09 Missing Necessary Checks in notifyRewardAmount](#l-09-missing-necessary-checks-in-notifyrewardamount)
- [L-10 Explicitly checking that nextDepositId increment does not overflow is best practice](#l-10-explicitly-checking-that-nextdepositid-increment-does-not-overflow-is-best-practice)
- [L-11 Emit Events for Nonce Usage](#l-11-emit-events-for-nonce-usage)
- [L-12 Insufficient signature validation for sensitive operations](#l-12-insufficient-signature-validation-for-sensitive-operations)
- [L-13 Emit Events for Beneficiary Changes](#l-13-emit-events-for-beneficiary-changes)
- [L-14 Add more context to signed message to further prevent replay attacks](#l-14-add-more-context-to-signed-message-to-further-prevent-replay-attacks)
- [L-15 Insufficient Input Validations in stake function](#l-15-insufficient-input-validations-in-stake-function)
- [L-16 Limit the creation of new surrogates to authorized entities or under specific conditions](#l-16-limit-the-creation-of-new-surrogates-to-authorized-entities-or-under-specific-conditions)
- [L-17 Lack of check for the existence of the deposit in stakeMore function](#l-17-lack-of-check-for-the-existence-of-the-deposit-in-stakemore-function)
- [L-18 Employ Two-step confirmation process for Important Address changes](#l-18-employ-two-step-confirmation-process-for-important-address-changes)
- [L-19 Insufficient nonce validation](#l-19-insufficient-nonce-validation)
- [L-20 Explicitly reset the beneficiaryRewardPerTokenCheckpoint after rewards are claimed](#l-20-explicitly-reset-the-beneficiaryrewardpertokencheckpoint-after-rewards-are-claimed)
- [L-21 Enhance Event Logging](#l-21-enhance-event-logging)
- [NC-01 Use ternary operator for better readability in lastTimeRewardDistributed function](#nc-01-use-ternary-operator-for-better-readability-in-lasttimerewarddistributed-function)
- [NC-02 rewardPerTokenAccumulated can be rewritten for more clarity](#nc-02-rewardpertokenaccumulated-can-be-rewritten-for-more-clarity)
- [NC-03 Event Emission Before State Change](#nc-03-event-emission-before-state-change)

## 🛠️ Approach

I conducted a manual code review, meticulously examining each line against a checklist tailored for identifying nuanced and low-severity issues in Solidity contracts. This process involved a deep dive into the contract's logic, external calls, and handling of user inputs, allowing for the swift identification of common vulnerabilities and areas requiring further scrutiny.

## [L-01] Implement Role-Based Access Control

The UniStaker contract uses a simple admin check for critical functionalities like setting the admin and toggling reward notifiers. However, broader access control mechanisms, such as role-based access control (RBAC) for different levels of permissions, are not implemented. This can limit the flexibility and security of managing different administrative operations.

### LOC
- [function setAdmin](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L201)
- [function setRewardNotifier](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L210)
- [function _revertIfNotAdmin](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L779)


### Instances

1. function setAdmin

```solidity
function setAdmin(address _newAdmin) external {
  _revertIfNotAdmin();
  _setAdmin(_newAdmin);
}
```
2. function setRewardNotifier

```solidity
function setRewardNotifier(address _rewardNotifier, bool _isEnabled) external {
  _revertIfNotAdmin();
  isRewardNotifier[_rewardNotifier] = _isEnabled;
  emit RewardNotifierSet(_rewardNotifier, _isEnabled);
}
```
3. function _revertIfNotAdmin

```solidity
function _revertIfNotAdmin() internal view {
  if (msg.sender != admin) revert UniStaker__Unauthorized("not admin", msg.sender);
}
```

## [L-02] Implement Deadline Checks in Time-sensitive Functions

### LOC
- [function permitAndStake](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L292)

- [function permitAndStakeMore](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L360)


### Instances

#### 1. function permitAndStake

`permitAndStake` could benefit from ensuring the deadline is respected beyond the permit call itself.

```solidity
function permitAndStake(
    uint256 _amount,
    address _delegatee,
    address _beneficiary,
    uint256 _deadline,
    uint8 _v,
    bytes32 _r,
    bytes32 _s
) external {
    require(block.timestamp <= _deadline, "Transaction expired");
    // Existing permit and stake logic...
}

```

#### 2. function permitAndStakeMore

It's advisable to check that the current block timestamp is less than the deadline provided in the permitAndStakeMore function. This ensures the permit has not expired at the time of execution.

```solidity

function permitAndStakeMore(
    DepositIdentifier _depositId,
    uint256 _amount,
    uint256 _deadline,
    uint8 _v,
    bytes32 _r,
    bytes32 _s
) external {
    require(block.timestamp <= _deadline, "Permit: signature expired");

    Deposit storage deposit = deposits[_depositId];
    _revertIfNotDepositOwner(deposit, msg.sender);

    STAKE_TOKEN.permit(msg.sender, address(this), _amount, _deadline, _v, _r, _s);
    _stakeMore(deposit, _depositId, _amount);
}
```

## [L-03] Validate Non-zero Amounts

Ensure that functions dealing with value transfers or balances check for non-zero amounts.

### LOC

### Instances

#### 1. function stake

```solidity
function stake(uint256 _amount, address _delegatee) external {
    require(_amount > 0, "Amount must be greater than 0");
}
```

## [L-04] Scalability Concerns in Reward Calculation Mechanism of UniStaker Contract

### LOC

### Instances

In scenarios with high transaction volumes, significant fluctuations in staking behavior, or a large number of stakers, the existing mechanisms might not scale efficiently. This could lead to challenges in timely updating reward rates, calculating individual rewards, and maintaining the contract's performance and accuracy.


### Instances

**Global Reward Rate Updates**: 

The contract recalculates the global reward rate upon notification of new rewards, adjusting for the remaining reward duration or initiating a new duration. This calculation is critical for subsequent individual reward allocations.

    ```solidity
    if (block.timestamp >= rewardEndTime) {
        scaledRewardRate = (_amount * SCALE_FACTOR) / REWARD_DURATION;
    } else {
        uint256 _remainingReward = scaledRewardRate * (rewardEndTime - block.timestamp);
        scaledRewardRate = (_remainingReward + _amount * SCALE_FACTOR) / REWARD_DURATION;
    }
    ```
**Individual Reward Calculations**: 

The mechanism for recalculating each staker's reward entitlement based on their share of the total staked amount. This calculation relies on the updated global reward rate and individual staker's data, which could become less efficient under conditions of frequent large-scale changes.
    ```solidity
    function unclaimedReward(address _beneficiary) public view returns (uint256) {
        return unclaimedRewardCheckpoint[_beneficiary]
            + (earningPower[_beneficiary]
            * (rewardPerTokenAccumulated() - beneficiaryRewardPerTokenCheckpoint[_beneficiary])
            ) / SCALE_FACTOR;
    }
    ```

#### Impact
Inefficiencies in recalculating rewards could lead to delays in updating stakers' reward entitlements, affecting the timeliness of reward distribution.

#### Mitigation Strategies
Offload part of the calculation workload to off-chain systems, with on-chain verification to ensure the integrity and fairness of reward distributions.

## [L-05] Use `abi.encode` instead of `abi.encodePacked` when computing type hashes

The problem with using `abi.encode` in this context is that it adds padding to the encoded data, which can lead to different hash values being computed for the same type string across different clients or implementations.

### LOC

- [function claimRewardOnBehalf](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L544)
- [function stakeOnBehalf](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L315)
- [function stakeMoreOnBehalf](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L382)

### Instances
See links above.

#### 1. STAKE_TYPEHASH

```solidity
bytes32 public constant STAKE_TYPEHASH = keccak256(
    abi.encode(
        "Stake(uint256 amount,address delegatee,address beneficiary,address depositor,uint256 nonce)"
    )
);
```

use `abi.encodePacked` instead of `abi.encode` when computing the type hashes. `abi.encodePacked` does not add any padding to the encoded data, ensuring that the same type string will always result in the same hash value, regardless of the client or implementation.

## [L-06] Lack of Slashing Conditions for Staked Assets

The contract lacks a built-in mechanism to slash staked assets or withhold rewards in the case of staker misbehavior. This omission could potentially be exploited by participants who act maliciously or fail to meet predefined conditions essential for the protocol's integrity or operational efficiency, without any repercussions on their staked assets or earned rewards.

### LOC

- [Unistaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol)

### Instances

- Unistaker.sol

Introduce smart contract logic to slash a portion of the staked assets or withhold rewards for stakers found to be acting maliciously or failing to meet specific protocol responsibilities. These conditions should be clearly defined and understood by all participants.


## [L-07] Lack of Emergency Withdrawal Mechanisms

Failure to protect user assets in emergency situations can severely damage the reputation of the protocol and erode trust within the community.

### LOC

- [Unistaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol)

### Instances

- Unistaker.sol

Integrate an emergency withdrawal feature that allows users to retrieve their staked assets independently, without interacting with potentially compromised contract functions.

## [L-08] Implement Nonce Expiry to better safeguard against replay attacks 

Implementing a nonce expiry mechanism can further protect against replay attacks, where a nonce becomes invalid after a certain period or event, ensuring that old transactions cannot be replayed indefinitely.

### LOC
- [function stakeOnBehalf](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L315)
- [function stakeMoreOnBehalf](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L382)
- [function alterDelegateeOnBehalf](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L423)
- [function alterBeneficiaryOnBehalf](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L466)

### Instances
See links above


## [L-09] Missing Necessary Checks in notifyRewardAmount

The notifyRewardAmount function lacks sufficient checks to ensure that the _amount parameter matches the actual amount of tokens transferred to the contract before calling this function. This could potentially lead to discrepancies between the expected and actual reward amounts.

### LOC
- [function notifyRewardAmount](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L570)

### Instances

#### 1. function notifyRewardAmount

```solidity

function notifyRewardAmount(uint256 _amount) external {
    if (!isRewardNotifier[msg.sender]) revert UniStaker__Unauthorized("not notifier", msg.sender);
    // ...
    if (
      (scaledRewardRate * REWARD_DURATION) > (REWARD_TOKEN.balanceOf(address(this)) * SCALE_FACTOR)
    ) revert UniStaker__InsufficientRewardBalance();
    // Existing logic...
}
```

## [L-10] Explicitly checking that nextDepositId increment does not overflow is best practice

The require statement ensures that the underlying uint value of nextDepositId is less than the maximum value a uint256 can hold before incrementing. This prevents overflow errors. Given Solidity 0.8's inherent overflow checks, this might be more about explicit safety and clarity than a strict necessity.

### LOC
- [function _useDepositId](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L630)

### Instances

#### 1. function _useDepositId
```solidity
function _useDepositId() internal returns (DepositIdentifier _depositId) {
    require(DepositIdentifier.unwrap(nextDepositId) < type(uint256).max, "DepositId overflow");

    _depositId = nextDepositId;
    nextDepositId = DepositIdentifier.wrap(DepositIdentifier.unwrap(_depositId) + 1);
}

```

## [L-11] Emit Events for Nonce Usage

Emitting events upon the use of a nonce can provide transparency and allow users and off-chain services to track nonce usage, making unexpected behavior or attacks more visible.

### LOC
- [function stakeOnBehalf](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L315)
- [function stakeMoreOnBehalf](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L382)
- [function alterDelegateeOnBehalf](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L423)
- [function alterBeneficiaryOnBehalf](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L466)

### Instances
See links above

## [L-12] Insufficient signature validation for sensitive operations

Consider using additional layers of security like session keys or time-bound operations (where a signature is only valid within a certain timeframe), especially for sensitive operations.

### LOC
- [function stakeOnBehalf](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L315)
- [function stakeMoreOnBehalf](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L382)
- [function alterDelegateeOnBehalf](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L423)
- [function alterBeneficiaryOnBehalf](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L466)

### Instances
See links above (any function taking signature parameter can benefit)

## [L-13] Emit Events for Beneficiary Changes

Incorporating an event like BeneficiaryChanged enhances the contract's transparency by providing an auditable trail of beneficiary changes, which can be vital for governance and in scenarios where stakeholders need to track the history of such changes.

### LOC
- [function alterBeneficiary](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L453)

### Instances

#### 1. function alterBeneficiary

```solidity
event BeneficiaryChanged(
  DepositIdentifier indexed depositId,
  address indexed oldBeneficiary,
  address indexed newBeneficiary
);

function alterBeneficiary(DepositIdentifier _depositId, address _newBeneficiary) external {
  Deposit storage deposit = deposits[_depositId];
  _revertIfNotDepositOwner(deposit, msg.sender);

  // Emit an event before making the change
  emit BeneficiaryChanged(_depositId, deposit.beneficiary, _newBeneficiary);

  _alterBeneficiary(deposit, _depositId, _newBeneficiary);
}

```

## [L-14] Add more context to signed message to further prevent replay attacks

### LOC
- [function alterDelegateeOnBehalf](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L423)

### Instances

#### 1. function alterDelegateeOnBehalf
```solidity
_revertIfSignatureIsNotValidNow(
  _depositor,
  _hashTypedDataV4(
    keccak256(
      abi.encode(
        ALTER_DELEGATEE_TYPEHASH,
        _depositId,
        _newDelegatee,
        _depositor,
        _useNonce(_depositor),
        // Additional context like contract address and a unique operation identifier could be included here
        address(this),
        block.timestamp
      )
    )
  ),
  _signature
);

```
## [L-15] Insufficient Input Validations in stake function

### LOC
- [function stake](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L256)
- [function permitAndStake](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L292)
- [function stakeOnBehalf](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L315)

### Instances

#### 1. function stake

Explicitly checking for a non-zero stake amount at the function's entry point can enhance clarity and prevent unnecessary contract calls. This makes the function fail fast if the condition is not met, saving gas for the caller and preventing unnecessary state changes or event emissions.

```solidity
function stake(uint256 _amount, address _delegatee) external returns (DepositIdentifier _depositId) {
    require(_amount > 0, "Amount must be greater than 0");
    _depositId = _stake(msg.sender, _amount, _delegatee, msg.sender);
}

```

#### 2. function permitAndStake

Add a requirement at the beginning of the permitAndStake function to ensure that the amount being staked is greater than 0. This early check ensures that no further operations are performed if the stake amount does not meet this basic criterion.



```solidity
function permitAndStake(
    uint256 _amount,
    address _delegatee,
    address _beneficiary,
    uint256 _deadline,
    uint8 _v,
    bytes32 _r,
    bytes32 _s
) external returns (DepositIdentifier _depositId) {
    require(_amount > 0, "Amount must be greater than 0");
    STAKE_TOKEN.permit(msg.sender, address(this), _amount, _deadline, _v, _r, _s);
    _depositId = _stake(msg.sender, _amount, _delegatee, _beneficiary);
}

```

#### 3. function stakeOnBehalf

```solidity
function stakeOnBehalf(
    uint256 _amount,
    address _delegatee,
    address _beneficiary,
    address _depositor,
    bytes memory _signature
) external returns (DepositIdentifier _depositId) {
    // Validate the signature
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

    // Ensure the amount to be staked is greater than 0
    require(_amount > 0, "Amount must be greater than 0");

    // Proceed with staking
    _depositId = _stake(_depositor, _amount, _delegatee, _beneficiary);
}
```

## [L-16] Limit the creation of new surrogates to authorized entities or under specific conditions to avoid unnecessary deployments and centralization risks.

Add an additional require statement checks whether the caller (msg.sender) is authorized to deploy a new DelegationSurrogate contract.

### LOC
- [function _fetchOrDeploySurrogate](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L605)

### Instances

1. function _fetchOrDeploySurrogate

```solidity
// Add an authorization check or a condition to limit who can trigger the deployment of new surrogates
function _fetchOrDeploySurrogate(address _delegatee)
    internal
    returns (DelegationSurrogate _surrogate)
{
    _surrogate = surrogates[_delegatee];

    if (address(_surrogate) == address(0)) {
        // Ensure that the caller is authorized to deploy a new surrogate
        require(isAuthorizedDeployer(msg.sender), "Unauthorized to deploy surrogate");

        _surrogate = new DelegationSurrogate(STAKE_TOKEN, _delegatee);
        surrogates[_delegatee] = _surrogate;
        emit SurrogateDeployed(_delegatee, address(_surrogate));
    }
}

```

## [L-17] Lack of check for the existence of the deposit in stakeMore function

A check can be added to ensure that the deposit exists before allowing more stake to be added. This can prevent unintended or malicious interactions with non-existent deposits.

### LOC
- [function stakeMore](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L342)

### Instances

#### 1. function stakeMore

```solidity
function stakeMore(DepositIdentifier _depositId, uint256 _amount) external {
    require(DepositIdentifier.unwrap(_depositId) < DepositIdentifier.unwrap(nextDepositId), "Deposit does not exist");
    Deposit storage deposit = deposits[_depositId];
    _revertIfNotDepositOwner(deposit, msg.sender);
    _stakeMore(deposit, _depositId, _amount);
}

```
## [L-18] Employ Two-step confirmation process for Important Address changes

The _alterBeneficiary function changes the beneficiary address for a given deposit. It updates who receives the rewards without necessarily ensuring that the new beneficiary is aware or has consented to this change. This could lead to situations where rewards are directed to an unintended address due to user error or malicious activity. You could also use Openzeppelin's Ownable2step here.

### LOC
- [function _alterBeneficiary](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L704)

### Instances

#### 1. function _alterBeneficiary

```solidity
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


## [L-19] Insufficient nonce validation

If the contract doesn't ensure nonces are used sequentially without gaps, there might be vulnerabilities. For example, if a user can skip a nonce, it might allow an old signature to be replayed if the skipped nonce is used.

While the `_useNonce` function likely increments the nonce, explicitly checking that the provided nonce matches the expected nonce for the user adds clarity and ensures no gaps are allowed.

### LOC
- [function stakeOnBehalf](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L327)

### Instances

#### 1. function stakeOnBehalf

```solidity
// For staking on behalf of a user
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
          _useNonce(_depositor) // <-- Nonce usage here
        )
      )
    ),
    _signature
  );
  _depositId = _stake(_depositor, _amount, _delegatee, _beneficiary);
}
```
## [L-20] Explicitly reset the beneficiaryRewardPerTokenCheckpoint after rewards are claimed

There is no explicit mechanism in the contract to clear the `beneficiaryRewardPerTokenCheckpoint` for a beneficiary after they have claimed their rewards. This means that if a beneficiary claims their rewards and later earns more rewards by staking more tokens or due to a new reward notification, the calculation of their new `unclaimedReward` may be incorrect because the `beneficiaryRewardPerTokenCheckpoint` is not reset.

### LOC
- [function _claimReward](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L740)

### Instances

#### 1. function _claimReward

```solidity
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
The contract should reset the `beneficiaryRewardPerTokenCheckpoint` for a beneficiary after they have claimed their rewards. This can be done by adding a line in the `_claimReward` function to reset the checkpoint.

```solidity
function _claimReward(address _beneficiary) internal {
    _checkpointGlobalReward();
    _checkpointReward(_beneficiary);

    uint256 _reward = unclaimedRewardCheckpoint[_beneficiary];
    if (_reward == 0) return;
    unclaimedRewardCheckpoint[_beneficiary] = 0;
    beneficiaryRewardPerTokenCheckpoint[_beneficiary] = rewardPerTokenAccumulatedCheckpoint; // Reset checkpoint
    emit RewardClaimed(_beneficiary, _reward);

    SafeERC20.safeTransfer(REWARD_TOKEN, _beneficiary, _reward);
}
```

## [L-21] Enhance Event Logging

The RewardNotifierSet event should be extended to include the admin's address as an event parameter. And the function emitting this event would include the msg.sender in the emitted event:


### LOC
- [event RewardNotifierSet](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L213)

### Instances

#### 1. event RewardNotifierSet

```solidity
event RewardNotifierSet(address indexed account, bool isEnabled);
```

```solidity
emit RewardNotifierSet(msg.sender, _rewardNotifier, _isEnabled);
```

## [NC-01] Use ternary operator for better readability in lastTimeRewardDistributed function

The `lastTimeRewardDistributed` function can be optimized for better readability by using a ternary operator, which simplifies the code without changing its logic. This is more of a stylistic preference that enhances code readability and conciseness.

### LOC
- [function rewardPerTokenAccumulated](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L220)

### Instances

#### 1. function lastTimeRewardDistributed

```solidity

function lastTimeRewardDistributed() public view returns (uint256) {
    return rewardEndTime <= block.timestamp ? rewardEndTime : block.timestamp;
}

```

## [NC-02] rewardPerTokenAccumulated can be rewritten for more clarity

This version adds comments for clarity and breaks down the calculation into more steps, which can help with readability and debugging.

### LOC
- [function rewardPerTokenAccumulated](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L229)

### Instances

#### 1. function rewardPerTokenAccumulated

```solidity
function rewardPerTokenAccumulated() public view returns (uint256) {
    // If no tokens are staked, no new rewards have been accumulated.
    if (totalStaked == 0) {
        return rewardPerTokenAccumulatedCheckpoint;
    }

    // Calculate new rewards per token since last checkpoint.
    uint256 timeSinceLastCheckpoint = lastTimeRewardDistributed() - lastCheckpointTime;
    uint256 rewardAccumulation = scaledRewardRate * timeSinceLastCheckpoint;
    uint256 rewardsPerTokenSinceLastCheckpoint = rewardAccumulation / totalStaked;

    return rewardPerTokenAccumulatedCheckpoint + rewardsPerTokenSinceLastCheckpoint;
}
```

## [NC-03] Event Emission Before State Change

The emit RewardClaimed(_beneficiary, _reward); event is emitted before the actual transfer of rewards. In the standard practice of smart contract development, events are usually emitted after state changes to reflect the final state accurately. 

### LOC
- [function _claimReward](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L740C3-L750C4)

### Instances

#### 1. function _claimReward

```solidity
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

## Conclusion

In this report I gathered all low severity and informational bugs I could find in the code together with relevant code snippets, links and recommendations. The main focus of my QA analysis was the Unistaker.sol contract. As mentioned earlier I used an extensive custom-tailored checklist of common issues affecting solidity smart contracts to base my analysis. I also checked for compliance issues with the relevant EIPs the contract is implementing, which I was able to find some issues in light of this. For a high level analysis of my findings including high and medium severity issues refer to the analysis report and individual bug reports.
