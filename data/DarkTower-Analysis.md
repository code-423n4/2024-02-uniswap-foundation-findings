[Preface](#1-preface)

[Review process](#2-review-process)

[Architecture and Recommendations](#3-architecture-and-recommendations)

[Call trace diagrams](#4-call-trace-diagrams)

[Oversight and mitigation](#5-oversight-and-mitigation)

[Conclusion](#6-conclusion)

***

## **1. Preface**

This Unistaker Infrastructure analysis report has been approached with the following key points and goals in mind:

- Devoid of redundant documentation the protocol is familiar with.
- Engineered towards the provision of valuable insights and edge case issues the protocol left unnoticed/overlooked.
- Simple to grasp call-trace diagrams for first-time users who stumble upon this report in the future.

## **2. Review process**

**D6-7:**

- Understanding of Uniswap's Fee concept and the `StakingRewards` contract from Synthetix.
- Mapping out security and attack areas relating to stakes, rewards, and delegations.
- Keeping track of the outlined areas with notes on contracts in scope.

**D7-9:**

- Brainstorm possible reachability of undesired states.
- Test the areas identified.
- Further review of current contract-level mitigations.

**D10:**

- Apply mitigations.
- Draft & submit report.

## **3. Architecture and Recommendations**

### Math:

For a staking contract and the usual Uniswap way of approaching their contracts, you would expect to see a ton of complex math all over the place but the Unistaker contract has rather disappointed some of us because there's no such overly-loaded math in the contracts that are in-scope. The math employed for the contracts in-scope for calculations is incredibly precise, modular, and easy to grasp leaving almost no room for tricky calldata.


### Withdrawals and Reward Claims:
As is, there is a `withdrawOnBehalf` function that we think would be useful for helping a depositor withdraw their stakes, hence delegated tokens. But, this function isn't effective in the sense that if a staker lost their private key/wallet due to compromise to another user who has access and can sweep any amount, the withdrawals made on behalf of such staker will be lost because the tokens are being transferred to the deposit owner which in this case is the staker whose wallet is compromised by any means as can be seen in the code block below:

```js
_stakeTokenSafeTransferFrom(address(surrogates[deposit.delegatee]), deposit.owner, _amount);
```

**Mitigation:**
The `_withdraw` function facilitates all withdrawals of the protocol. These withdraws can be for self or others and it sends the tokens to the `owner` tied to a deposit. In the case of a `withdrawOnBehalf` implementation, the Uniswap team can add another function such as the `_withdrawOnBehalf` function to allow a compromised staker specify an address they would like to recieve the tokens being withdrawn. 

What does this enforce? Added security. Last thing you want to be doing is sending more tokens to a compromised address. The `staker` already provides a signature for one such withdrawal. So, we can already be sure they initiated the process and trust the `address` they would specify as the `recipient` of the tokens being withdrawn which isn't themselves (compromised).

This implementation would look something like this for the token transfer block in the new `_withdrawOnBehalf` internal function implementation:

```js
function _withdrawOnBehalf(Deposit storage deposit, DepositIdentifier _depositId, uint256 _amount, address _recipient)
    internal
  {
    ...
    deposit.balance -= _amount;
    totalStaked -= _amount;
    depositorTotalStaked[deposit.owner] -= _amount;
    earningPower[deposit.beneficiary] -= _amount;
    _stakeTokenSafeTransferFrom(address(surrogates[deposit.delegatee]), _recipient, _amount);
    emit StakeWithdrawn(_depositId, _amount, deposit.balance);
  }
```

This recommendation can be implemented for the `claimRewardOnBehalf` function too. Also, since the Staking contract has no functionality to recover tokens from a `delegatee` to the `depositor` for any reason where such token withdrawals cannot be sent back to the owner, this recommendation is great because you would expect an admin to have a similar function which the Synthetix's `StakingRewards` contract has in the `recoverERC20` function.

### Comparison between `UniStaker` and Synthetix `StakingRewards`
 
| Feature  | UniStaker  | StakingRewards  |
|---|---|---|
| Accessibility  | Stakers can stake on their behalf as well as for others. The approvals for transfers just needs to be in place to facilitate token transfers. | Only the caller is staked. They cannot stake on behalf of others. |
| Rewards  | In some cases, a user may want to send their rewards to another address or have another address claim on their behalf. The UniStaker contract presents this functionality.  | Only the depositor claims the accrued rewards for their deposits.   |
| Recovery  | There's no functionality to recover tokens erroneously sent to the contract. Any token ever stuck in the contract cannot be pulled.  | There are some erroneous transfers or actions that can lead to stuck tokens in the contract. The Synthetix team has a function to recover such tokens from the contract. |
| Staking thresholds  | Zero value stakes go through.  | Zero value stakes are not allowed to flood the protocol.  |

## **4. Call trace diagrams**
Entry point:
![Entry](https://rexjoseph.github.io/images/unistaker-entry.png)

Reward claims:
![Entry](https://rexjoseph.github.io/images/unistaker-reward-claim.png)

Exit point:
![Entry](https://rexjoseph.github.io/images/unistaker-exit.png)

## **5. Oversight and mitigation**
- **Zero value stakes are worthless** It's best to keep these out right from the get go and save the caller some gas. For example, instead of paying 7.15 USD for a transaction that yields nothing, it's better to pay close to almost nothing for the same transaction by reverting it when you already know it would yield nothing. Such transactions are zero-value stakes. The user gets no the reward executing such transactions, it also isn't of use to the staking contract or Uniswap.

## **6. Conclusion**
The Unistaker Infrastructure Reward system for stakers has taken a good approach to rewarding users such as the linear ongoing rewards method which rewards users during the reward duration set as 30 days. What this has effectively done is, blockade users double claiming rewards which belong to other users within the system. With this feature, each user gets what they're owed and frontrun/race condition claims are shut out.

### Time spent:
10 hours