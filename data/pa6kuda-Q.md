# Typography error in `V3FactoryOwner::claimFees()` natspec

## Summary
`V3FactoryOwner::claimFees()` natspec has redundant `the`.

## Relevant GitHub Links
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L170

## Recommended Mitigation Steps
Consider removing redundant `the`

```diff
      /// The same mechanic applies regardless of what the pool currencies, payout token, or payout
      /// amount are. Effectively, as each pool accrues fees, it eventually becomes possible to "buy"
-     /// the pool fees for less than they are valued by "paying" the the payout amount of the payout
+     /// the pool fees for less than they are valued by "paying" the payout amount of the payout
      /// token.
```