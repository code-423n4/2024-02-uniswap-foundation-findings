# 

| ID | Title | Severity |
| --- | --- | --- |
| [I-01](#I-01) | Eliminate permit grief risk | Informational |
| [I-02](#I-02) | Protocol usage will lock funds inside  | Informational |
| [I-03](#I-03) | No easy way to withdraw | Informational |

## **I-01**: Eliminate permit grief risk

## **Severity:**

- Informational

## **Summary:**
In the protocol, there are two permit convenience functions susceptible to front-running. It's important to note that addressing this vulnerability is currently out-of-scope for this audit. However, a suggested approach to mitigate this risk is to incorporate a straightforward allowance check. By implementing such a check, user transactions would be prevented from being reverted, ensuring they are executed as intended.


[PermitAndStake](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L292-L303)
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
-   STAKE_TOKEN.permit(msg.sender, address(this), _amount, _deadline, _v, _r, _s);
+   _permit(_amount, _deadline, _v, _r, _s);
    _depositId = _stake(msg.sender, _amount, _delegatee, _beneficiary);
  }

+  function _permit(uint256 _amount, uint256 _deadline, uint8 _v, bytes32 _r, bytes32 _s) internal {
+    if (STAKE_TOKEN.allowance(msg.sender, address(this)) != _amount) {
+      STAKE_TOKEN.permit(msg.sender, address(this), _amount, _deadline, _v, _r, _s);
+    }
+  }
```

[PermitAndStakeMore](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L360-L373)

```solidity
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

-   STAKE_TOKEN.permit(msg.sender, address(this), _amount, _deadline, _v, _r, _s);
+   _permit(_amount, _deadline, _v, _r, _s);
    _stakeMore(deposit, _depositId, _amount);
  }
```

## **Tools Used:**

- Manual analysis

## **Recommendation:**
Add the new utility func and forget about any griefing attacks. 


## **I-02**: Protocol usage will lock funds inside

## **Severity:**

- Informational

## **Summary:**

The usage of the contract will result in small amounts of tokens being accumulated in the contract because of operational imprecisions, without any means of distribution back to stakers.

Here is a coded POC which can be directly pasted in UniStaker.t.sol

```solidity
function testFuzz1_CalculatesCorrectEarningsForTwoUsersThatDepositEqualStakeForFullDuration(
    address _depositor1,
    address _depositor2,
    address _delegatee,
    uint256 _stakeAmount1,
    uint256 _stakeAmount2,
    uint256 _rewardAmount,
    uint256 _times
  ) public {
    vm.assume(_depositor1 != _depositor2);
    _rewardAmount = bound(_rewardAmount, 200e6, 10_000_000e18);
    _times = bound(_times, 10, 250);
    uint256 totalLeft = 0; // total left over
    for (uint256 i = 0; i < _times; i++) {
      // A user deposits staking tokens
      _boundMintAndStake(_depositor1, _stakeAmount1, _delegatee);
      // Another depositor deposits the same number of staking tokens
      _boundMintAndStake(_depositor2, _stakeAmount2, _delegatee);
      // The contract is notified of a reward
      _mintTransferAndNotifyReward(_rewardAmount);
      _jumpAheadByPercentOfRewardDuration(101); // full duration

      // users claim their rewards
      vm.prank(_depositor1);
      uniStaker.claimReward();
      vm.prank(_depositor2);
      uniStaker.claimReward();

      uint256 unistakerBalance = uniStaker.REWARD_TOKEN().balanceOf(address(uniStaker));
      //assert that unistakerBalance is increasing
      assertGe(unistakerBalance, totalLeft);
      totalLeft = unistakerBalance;
    }
    // assert that unistakerBalance is greater than 0 after all the iterations
    assertNotEq(totalLeft, 0);
  }
```

If a portion of these rewards remains unused, as illustrated in the example provided within the contract, this unused amount remains inactive within the contract.

## **Tools Used:**

- Manual analysis

## **Recommendation:**

There is no easy way to implement a redistribution mechanism for this token buildup. But the amount will be negligible because of the high precision but will accumulate nevertheless.  

## **I-03**: No easy way to withdraw

## **Severity:**

- Informational

## **Relevant GitHub Links:**

- https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L499

## **Summary:**

A depositor needs to keep his depositId store somewhere to be able to withdraw his deposit and there is no easy way for him to check his depositIds if he forgot to keep it safe, he will have to scour etherscan to find his deposit transactions to find the depositId or the transactions of an address he authorized to make deposits on his behalf. 

## **Impact:**

The above mentioned issue can result in a degraded customer experience. 

## **Tools Used:**

- Manual analysis

## **Recommendation:**

Consider implementing a public variable which will keep track of user’s deposits or even a unstakeAll() function.