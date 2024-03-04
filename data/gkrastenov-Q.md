# [LOW-01] The payoutAmount should be different for every pool.
## Impact
The accrual of the fee depends on the trading volume of the pool. In some pools, during the low interest of exchanging, the payout amount can not be reached for a long time. The payout amount is the same for every pool, which can not be correct if the pools have different activity levels.

The payout amount should be different for all pools and the DAO should have the ability to increase or decrease it if some pairs are more active than others.

## Recommended Mitigation Steps
Allow the DAO to set the minimum payout amount for every pool.