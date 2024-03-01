# QA Report

| *Issue* | *Description*                                                                  |
|---------|--------------------------------------------------------------------------------|
| [L-01]  | Sum of governance token balances of all surrogate contracts can be greater than total staked amount                                                                                     |

## [L-01] Sum of governance token balances of all surrogate contracts can be greater than total staked amount

As stated in invariant #2, "the sum of the governance token balances of all the surrogate contracts is equal to the sum of all deposits less the sum of all withdrawals". This invariant can be broken by transferring governance tokens directly to a surrogate address. The delegated votes, however, are tracked correctly, as only the amount transferred through the staking process gets delegated. The directly transferred assets are stuck in the contract, as the withdrawal process in [UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol) prevents one from withdrawing more than their staked amount.

While, the broken invariant does not seem to present any immediate dangers, issues may arise from any potential future accounting with assumptions based on it being true.

### **Proof of concept**

- A user stakes 1000 governance tokens.
- Another 1000 governance tokens are to sent to the surrogate contract

```
function test_Sum_of_surrogate_balance_equals_total_stake() public {
    address delegatee = address(1);
    address beneficiary = address(2);

    // stake 1000 gov tokens, delegate to delegatee
    handler.stake(1000, delegatee, beneficiary);
    address surrogate = address(uniStaker.surrogates(delegatee));

    // mint and send 1000 gov tokens directly to surrogate address
    govToken.mint(address(3), 1000);
    vm.prank(address(3));
    govToken.transfer(surrogate, 1000);

    // sum of all staked gov tokens should be equal to sum of surrogate gov token balances
    assertEq(uniStaker.totalStaked(), handler.reduceDelegates(0, this.accumulateSurrogateBalance));
}
```

The expected results from this test fail, with surrogate balance being 2000, and the total staked amount remaining 1000.

```[FAIL. Reason: assertion failed] test_Sum_of_surrogate_balance_equals_total_stake() (gas: 948026)
Logs:
  Bound Result 1000
  Error: a == b not satisfied [uint]
        Left: 1000
       Right: 2000

```
