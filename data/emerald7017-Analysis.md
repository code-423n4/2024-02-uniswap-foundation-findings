## Overview

The UniStaker contract allows UNI token holders to stake their tokens and earn additional token rewards over time. It is designed to incentivize long-term locking of UNI tokens to empower Uniswap governance participation.

**Contract Architecture**

```
┌──────────────────────────────────────────┐        
│                                          │        
│            UniStaker Contract            │        
│                                          │        
├──────────────────────────────────────────┤        
│  - Reward Distribution Logic             │        
│  - Deposit Management                    │        
│  - Delegation Handling                   │
│  - Admin Functions                       │        
│                                          │
└───────────────────┬──────────────────────┘
                     │
                     │ interfaces with
                     ▼  
┌──────────────────────────────────────────┐
│                                          │
│         External Reward Token           │
│                                          │
└──────────────────────────────────────────┘
```

The UniStaker contract consists of four key components:

1. **Reward Distribution Logic:** Calculates rewards earned by each depositor based on stake amount and duration. Manages streaming of rewards over fixed time period.

2. **Deposit Management:** Allow users to stake UNI tokens, withdraw stake, and manage associated deposit metadata (beneficiary, delegatee etc.)

3. **Delegation Handling:** Facilitates delegation of voting power of staked tokens to chosen delegates. Achieved via `DelegationSurrogate` contracts.

4. **Admin Functions:** Powerful control functions like setting admin, managing reward notifiers.

It interacts heavily with an external reward ERC20 token contract to distribute rewards.

**Vulnerability Analysis**

```
┌───────────────────────────────────────────────────────────┐
│                                                           │  
│                        Admin Risks                       │
│                                                           │
├───────────────────────────────────────────────────────────┤
│   - Admin has excessive privileges                        │ 
│   - Admin functions lack access controls                  │
│   - Lack of transparency around admin actions             │
│   - No timelocks or multisig protections                  │
│                                                           │
│     => Could lead to centralization of power, fund       │
│        drainage, malicious upgrades                       │
│                                                           │  
└───────────────────────────────────────────────────────────┘
```

The key risks stem from the unchecked admin privileges. The admin can unilaterally:

- Drain funds by setting invalid reward rate  
- Disable/enable reward notifiers
- Change critical parameters like reward duration
- Deploy malicious contract upgrades

This could lead to loss of funds or make the contract unusable. There are no mitigating controls like timelocks, multisig schemes, or transparency over admin actions.

**The key administrator privileges are concentrated in the [`V3FactoryOwner`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol) contract**

| Function              | Description                                                                                  | Access   |
|-----------------------|----------------------------------------------------------------------------------------------|----------|
| `setAdmin`              | Allows changing the admin address. Only callable by current admin.                           | Admin    |
| `setPayoutAmount`       | Allows changing the payout amount. Only callable by admin.                                   | Admin    |
| `enableFeeAmount`       | Enables a fee amount on the Uniswap V3 factory. Only callable by admin.                      | Admin    |
| `setFeeProtocol`        | Sets protocol fee percentage on a Uniswap V3 pool. Only callable by admin.                   | Admin    |
| `setAdmin`              | Allows changing the admin address. Only callable by current admin.                           | Admin    |
| `setRewardNotifier`     | Allows enabling/disabling reward notifier addresses. Only callable by admin.                | Admin    |

## Centralization Risks

The contracts are highly dependent on the admin roles. A malicious admin could:

- Set extremely high protocol fees in `V3FactoryOwner`, draining fees from users

- Disable legitimate reward notifiers or enable malicious ones in `UniStaker` 

- Change critical parameters like reward duration in `UniStaker`

This could lead to loss of funds or make the contracts unusable.

**Authorization Flaws** 

The admin functions lack further access control. Any admin can take critical actions unilaterally.

**Upgradeability Risks**

The contracts are not upgradeable, so this is not a concern.

**Lack of Transparency**

Admin actions like changing fees or reward duration are logged with events. However, disabling/enabling reward notifiers has no event.

**Mitigation Recommendations**

- Use a Timelock for admin functions. This prevents unilateral control.

- Implement 2-of-3 multisig scheme for admin changes. Promotes decentralization.

- Emit events for all admin actions, even enabling/disabling notifiers. Improves transparency. 

- Introduce role-based access control. For example, separate roles for changing fees, managing notifiers, etc. Reduces centralization of power in one admin.

- Consider making admin a DAO. Allows community oversight of changes.

- Simplify `V3FactoryOwner` design by removing ability to change pool fees. Reduces surface area for exploits.

_The unchecked admin privileges present a significant risk. Adding timelocks, multisig schemes, access roles and other mitigations can help reduce this risk and promote decentralization, transparency and accountability. The context that these contracts are meant for mainnet deployment necessitates thorough security review before launch._

**Recommendations**

1. Decentralize admin powers 

    - Implement DAO-governed admin using a governance token
    - Use multi-sig schemes
    - Assign roles to separate critical functions

2. Add timelocks for admin functions

3. Improve transparency

   - Emit events on all admin actions
   - Log changes to sensitive parameters

4. Conduct more testing

   - Increase test coverage of reward calculation logic
   - Audit math intensely

**Conclusion**  

The UniStaker contract provides a mechanism to incentivize UNI staking and governance participation. However, the key risks stem from the unchecked admin privileges, which could lead to centralization and fund loss over time. Implementing recommendations around decentralizing control, adding timelocks and transparency, and enhancing testing would significantly improve sustainability and security.

The contracts do not directly rely on external price or data feeds. However, their utility depends on the overall health of the Uniswap ecosystem. Issues like manipulation of Uniswap price oracles could have indirect effects.

The main dependencies are the Uniswap V3 factory and pool contracts. Any issues with those contracts will directly impact this system. For example, a bug that freezes trading in Uniswap pools would also freeze fee accrual in this system.

### Composability Risks

There is a risk of complex interactions with flash loans. For example, a malicious actor could take a flash loan to manipulate Uniswap prices, trigger this contract to think substantial fees have accrued, claim the fees cheaply, then profit from price manipulation with the stolen fees.

**Governance Risks**

**Centralization**

Highly centralized as admin holds total control. Lack of community governance or multisig schemes is a risk.

**Lack of Adaptability** 

No built-in way for the community to vote on or enact changes. This could prevent responding quickly to exploits or changing market conditions.

**Mitigation Recommendations**

- Implement DAO governance to decentralize control.

- Create a process for halting the contract under emergency scenarios.

- Conduct extensive composability testing with other DeFi protocols.

- Model impacts of different Uniswap fee settings and liquidity conditions. 

- Verify core invariant calculations using formal methods.

- Structure incentives so that honest behavior is most profitable for users.

## Contracts Overview

The codebase consists of:

1. The **UniStaker** contract, which is a **Uniswap token staking contract** that allows users to delegate their governance votes and earn rewards over time. It has logic for:

   - Tracking staked balances and associated delegatees/beneficiaries
   - Periodically distributing rewards to stakers
   - Managing deposit operations like staking/withdrawing
   - Admin functions for controlling parameters/settings

2. **DelegationSurrogate** contract, a **helper contract** to enable separating voting power while still allowing delegated governance for stakers.

3. Several **interface contracts** that define specific methods/structs used by the core staking and factory contracts.

4. The **V3FactoryOwner** contract, which **facilitates transferring protocol fees** accrued in Uniswap V3 pools. It:

   - Enables any user to claim a pool's fees by paying a pre-set ERC20 token amount as payment
   - Forwards this payment to the UniStaker contract as a reward
   - Has privileged functions like setting pool fees, only callable by admin

The core concept is a Uniswap token staking system to motivate governance participation, funded through a fee claim mechanism powered by the V3 Factory owner contract.

The key risks stem from the unchecked admin privileges in the staking and factory owner contracts, creating potential for centralization of power and fund loss over time.

## Architecture view (Diagram)

```mermaid

flowchart TD

    subgraph Exterior contracts

        Uni[Uniswap v3 Pools]-- Transfers accrued fees --> F[V3FactoryOwner]

    end

    subgraph UniStaker System

        F -->|Allows claiming fees|CF[ClaimFees<br>(Public function)]

        CF -->|Pays payoutToken amount <br>to notify rewards| ST1[StakingTokens<br>(ERC20)]

        ST1 -->|Notifies of <br> reward amount|SA1[StakingAndAdmin<br>(UniStaker Contract)]

        SA1 --> |Deposits rewards|STR[StakingTokens<br>(Reward ERC20)]

        SA1 --> |Distributes rewards to|SB1[Stakers/Beneficiaries]

        SA1 -. Delegates votes .-> DS1[DelegationSurrogate]

        SB1 -. Provides liquidity .-> Uni  

        DS1 -. Votes on behalf of .-> SB1

    end
    
```

**Key Points:**

- Uniswap pools accrue fees and transfer to V3FactoryOwner
- Public can claim fees from V3FactoryOwner by paying "payoutTokens" 
- These payoutTokens act as rewards for UniStaker contract
- UniStaker accepts stake deposits, delegates votes, and distributes reward tokens to stakers
- DelegationSurrogate allows vote delegation while keeping staking power separate

This allows Uniswap governance participation to be incentivized via staking rewards funded through fee revenue.

The systemic risk stems from the admin privileges in V3FactoryOwner and UniStaker, allowing unchecked control.

### Potential attacks that could arise from Uniswap governance decisions related to incentives, fees, and staking parameters

**Reducing staking rewards**

- If rewards are reduced substantially, it disincentivizes staking. This could be used by malicious actors staking a minority share to gain greater control over voting.

**Increasing payout amount**

- If the payout amount for claiming fees rises considerably, it may break the profitability for existing MEV bots arbitraging fees. This could reduce frequency of fee collection.

**Whitelisting malicious reward notifiers**

- If malicious reward notifier contracts are whitelisted, they could grief actual rewards or steal collected fees. Strict criteria should be used by governance when whitelisting.

**Enabling fees on manipulable pools** 

- Enabling fees on pools that are lower liquidity or have outlier pricing could lead bots to target those pools more frequently.

**Reducing reward duration**

- If the reward duration parameter is lowered, it concentrates rewards in shorter periods which could advantage certain stakers unfairly.  

**Changing the reward token**

- If the reward token is changed, it will impact the value of staking returns, potentially in unpredictable ways.

**Frequent parameter changes**

- In general, frequent changes to staking parameters like reward duration, payout amount etc creates unpredictability that encourages certain participants over others.

A major consideration is ensuring governance decisions balance incentives between ordinary stakers, MEV bots, governance voters, and protocol sustainability. Dramatic changes risk unintended consequences. Decisions should promote the long-term health of the protocol.


### Reward notifier contracts to ensure they don't disrupt the UniStaker system

**Adding Reward Notifiers**

- There should be strict criteria enforced by governance for whitelisting new reward notifiers
- The code they execute and rewards they provide should be scrutinized
- Initially only a few trusted notifiers should be added
- Too many notifiers could enable griefing attack vectors

**Removing Reward Notifiers** 

- There should never be an instant removal of a notifier contract
- They should be phased out over a period of time to allow accounting for pending rewards
- The rewards duration parameter can be adjusted to stop new rewards
- But existing reward commitments must be honored 

**Notified Reward Amounts**

- Reward amounts should be reasonable based on total staked assets
- Tiny amounts lead to griefing issues
- Should not be the sole source of rewards long term

**Reward Delivery**

- Require notifier contracts to transfer promised rewards to UniStaker before notifying
- Prevents inflating rewards owed without providing funds
- Solidity balance checks help but are not sufficient alone

By restricting this privileged functionality, establishing strict vetting, and phasing changes gradually, risk is mitigated. The code overall appears to enforce good constraints around notifier management. But governance discretion is key.

### How the UniStaker contracts handle potential changes in network conditions due to upgrades:

**Immutability**

- The contracts are marked as immutable in the overview comments
- The core contract state variables are initialized in the constructor only
- This prevents changes to key parameters like reward token, stake token, etc.

**Handling Upgrades**

- There does not appear to be explicit logic to handle network upgrades
- But the impact should be minimized by the immutable nature 

**Delegation Surrogates**

- These contracts delegate votes to other addresses
- If the vote delegation standard was changed, it could break
- The interface uses a trimmed down version of ERC20Votes standard
- This reduces the potential scope of breakage  

**Other Factors**

- The use of the SafeERC20 library provides some upgrade resilience 
- The interfaces also isolate external contract integrations
- No governance control allows full contract replacements

**Summary**

The immutable approach, interface abstractions, and minimal delegate contracts help mitigate upgrade risks. But if the vote delegation method changed, surrogates may need to revoted. No active logic handles this.

### Potential issues I identified related to account abstraction and smart contract wallets:

**Sig issues with smart contract accounts**

- The contract allows permit and sig-based operations 
- Signatures from smart contract accounts may act unexpectedly
- Could prevent permissioned operations depending on implementation

**Unexpected delegate call interactions** 

- Usage of smart contract wallets often relies on delegate calls
- The delegation surrogates delegate votes to other contracts
- Complex and untested delegate call chains could result
- May disrupt voting or reward accounting

**Obscured end user identification**

- Usage of account abstractions obscures the end user
- This could allow users to circumvent intended limitations
- For example deposit limits based on addresses

**Compromised smart contract wallets**

- If a smart contract wallet is compromised that is interacting with UniStaker
- This could allow an attacker to steal funds or manipulate governance

**Unintended token approvals**

- Smart contract wallets often approve contracts for maximum tokens
- If a malicious contract is approved it could steal tokens

To mitigate these issues, care should be taken around integration with smart contract accounts or wallets. Usage should be restricted until confidence is built via testing and audits. Account abstraction should not allow circumventing intended business logic. And governance decisions should consider the impacts to users relying on account abstraction solutions.

### Calculations involving token balances and numerical values like stakes, fees, and rewards could potentially overflow or underflow, leading to incorrect results.

- The `totalStaked` amount could overflow if new deposits are added without checking for overflows. This could incorrectly show more total stake than what is actually staked.

- Withdrawals could underflow `totalStaked` or `deposit.balance` if overflow checks are missing, allowing users to withdraw more than they deposited.

- Calculating `scaledRewardRate` involves multiplication and division, which could overflow. This could result in an excessively high reward rate.

- Accumulating rewards over time in `rewardPerTokenAccumulated` could overflow without overflow protection.

- Calculating a user's `unclaimedReward` involves subtraction. If the global reward amount has overflowed but the user's checkpoint hasn't, it could incorrectly show they are owed a large reward.

- Transferring staking tokens or reward tokens could underflow balances if checks are missing.

To prevent these, all arithmetic operations should be wrapped in SafeMath functions like `add`, `sub`, `mul`, `div` to prevent overflows and underflows. Explicit checks before state changes can also help.

Thoroughly review all math operations involving token balances, stakes, reward amounts, and accumulations over time to ensure overflows and underflows cannot occur in any scenario. This will help ensure token accounting remains accurate.

### The contract does not explicitly handle pausing or resuming rewards after a long pause. This could cause issues.

Specifically, if no `notifyRewardAmount` calls are made for a duration longer than `REWARD_DURATION` (30 days), the `rewardEndTime` would expire.

When a new notification is finally made after a long pause like this, here is what would happen:

```solidity
if (block.timestamp >= rewardEndTime) {
  // This code path would execute after long pause  

  scaledRewardRate = (_amount * SCALE_FACTOR) / REWARD_DURATION;

} else {

  // This code path expects rewards were continual  

  uint256 _remainingReward = scaledRewardRate * (rewardEndTime - block.timestamp);
  
  // _remainingReward will be 0 after long pause
  
  scaledRewardRate = (_remainingReward + _amount * SCALE_FACTOR) / REWARD_DURATION; 

}
```

The contract no longer expects pauses in notifications and does not properly restart rewards after a pause. This could mess up accounting.

To fix this, I would recommend explicitly handling pausing/resuming via new methods:

```solidity
function pauseRewards() external {
  // Pause distribution, snapshot state 
}

function resumeRewards() external {

  // Calculate total elapsed pause time
  // Set new rewardEndTime
  // Adjust rates if needed  
  // Restart distribution
}
```

Adding this logic would allow the contract to be resilient to gaps in reward notifications.





### When fees are calculated in percentage terms or fractional values and then translated into payouts, there is room for rounding errors.

**Fee Calculation**

Uniswap fees are calculated as .03% of the swap volume. Tracking fractional percentage values already introduces some rounding that could benefit users:

```
uint24 swapFee = .03% // Stored rounded down

uint feeForSwap = swapFee * swapAmount / 100;

// Rounding benefits swapper here
```

**Payout Calculation**

The payout amount may also round down fron a fractional native token value:  

```
uint payoutAmount = 1.123 WETH;

// Rounds down to 1 WETH 
// Benefits fee claimer
```

This happens again distributing rewards:

```
uint reward = calcFractionalReward(*);

// Rounding reduces rewards to stakers
```

**Mitigations**

- Use wrapper libraries that always round up fractional tokens 

- Introduce rounding buffers on top of exact calculations

- Dynamically adjust payouts to equalize roundings

The contract should account for rounding systematically rather than benefitting a single party.



### Time spent:
45 hours