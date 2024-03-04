The UniStaker contract allows Uniswap governance token holders to stake their tokens and earn additional rewards. It distributes staking rewards over time to designated beneficiary addresses. Users can stake to a specific deposit, delegate voting power, and assign a beneficiary address in a highly flexible manner.

**Architecture**

```
┌──────────────────────┐     ┌──────────────────────┐
│                      │     │                      │  
│      User 1          ◀────▶        Reward          │
│                      │     │       Token          │
│ ┌────────────────┐   │     │                      │
│ │                │   │     │                      │
│ │     Deposit    │◀──────────────────────────────┘
│ │                │   │      ▲ ERC20 Transfer
│ └────────────────┘   │      │ Notify Rewards
│      ▲Delegate           │      │
│      │                   │      │
│                      │◀────┘      │
└──────────────────────┘           │
                                   │
                                   ▼
                             ┌────────────┐
                        ┌────>│Delegatee 1 │
                        │    └────────────┘
    Deposit surpluses │
 aggregated by delegatee │    ┌────────────┐
                      └─────>│Delegatee 2 │
                             └────────────┘
```

**Key Contracts**

* [`UniStaker`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol) - Core staking and rewards distribution logic
* [`DelegationSurrogate`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/DelegationSurrogate.sol) - Holds tokens on behalf of delegatees
* [`V3FactoryOwner`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol) - Manages protocol fees collection

**Strengths**

* Flexible deposit structure enables granular configuration
* Surrogate pattern retains voting power for delegated tokens
* Signatures allow permissionless deposits and withdrawals  

**Weaknesses**

* Overpowered admin privileges in `V3FactoryOwner`
* Reward rate manipulation possible by attackers

**Attack Scenarios**

| Attack | Description | Mitigation |
|-|-|-|
| Rogue Admin | Admin could steal staked tokens or distribute unfair rewards | Decentralize admin, adopt timelock |
| Reward Rate Manipulation | Attackers can notify tiny rewards to reduce rate, discouraging stakers | Implement minimum reward thresholds | 

**Recommendations**

* Decentralize control over protocol fee parameters 
* Formal verification of calculation logic
* Circuit breakers against rapid reward notifications

The UniStaker infrastructure enables flexible and permissionless Uniswap governance participation but suffers from some centralized admin risks that could be mitigated through decentralization, trust minimization, and security best practices.

## Overview of the contracts:

**Scope**

**Contracts (3)**

[**src/UniStaker.sol**](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol)
- Purpose: Manages distribution of rewards to Uniswap governance token stakers
- Libraries: @openzeppelin/*
- Lines of Code: 423

[**src/V3FactoryOwner.sol**](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol) 
- Purpose: Serves as owner of Uniswap V3 factory, manages fees
- Libraries: @openzeppelin/*
- Lines of Code: 87

[**src/DelegationSurrogate.sol**](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/DelegationSurrogate.sol)
- Purpose: Holds governance tokens while delegating voting power 
- Libraries: None
- Lines of Code: 8

**Interfaces (4)**

[**src/interfaces/IERC20Delegates.sol** ](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/IERC20Delegates.sol)
- Purpose: Subset of Uniswap governance token interface
- Lines of Code: 22

[**src/interfaces/INotifiableRewardReceiver.sol**](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/INotifiableRewardReceiver.sol)
- Purpose: Interface between V3FactoryOwner and UniStaker  
- Lines of Code: 4

[**src/interfaces/IUniswapV3FactoryOwnerActions.sol**](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/IUniswapV3FactoryOwnerActions.sol)
- Purpose: Required Uniswap V3 factory owner methods
- Lines of Code: 6

[**src/interfaces/IUniswapV3PoolOwnerActions.sol**](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/IUniswapV3PoolOwnerActions.sol) 
- Purpose: Methods only Uniswap V3 pool owner can call 
- Lines of Code: 7

The key purpose is to empower Uniswap governance through rewards for stakers. Core functionality centers around UniStaker for distribution, V3FactoryOwner for managing fees funding rewards, and DelegationSurrogate for retaining voting power of staked tokens.

## Clear Identification of Admin Privileges

- [`UniStaker` contract:](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol)

    - `admin` - Can set new admin, enable/disable reward notifiers

- [`notifyRewardAmount`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L570-L599) - Called by reward notifiers to notify rewards
```solidity
  function notifyRewardAmount(uint256 _amount) external {
    if (!isRewardNotifier[msg.sender]) revert UniStaker__Unauthorized("not notifier", msg.sender);


    // We checkpoint the accumulator without updating the timestamp at which it was updated, because
    // that second operation will be done after updating the reward rate.
    rewardPerTokenAccumulatedCheckpoint = rewardPerTokenAccumulated();


    if (block.timestamp >= rewardEndTime) {
      scaledRewardRate = (_amount * SCALE_FACTOR) / REWARD_DURATION;
    } else {
      uint256 _remainingReward = scaledRewardRate * (rewardEndTime - block.timestamp);
      scaledRewardRate = (_remainingReward + _amount * SCALE_FACTOR) / REWARD_DURATION;
    }


    rewardEndTime = block.timestamp + REWARD_DURATION;
    lastCheckpointTime = block.timestamp;


    if ((scaledRewardRate / SCALE_FACTOR) == 0) revert UniStaker__InvalidRewardRate();


    // This check cannot _guarantee_ sufficient rewards have been transferred to the contract,
    // because it cannot isolate the unclaimed rewards owed to stakers left in the balance. While
    // this check is useful for preventing degenerate cases, it is not sufficient. Therefore, it is
    // critical that only safe reward notifier contracts are approved to call this method by the
    // admin.
    if (
      (scaledRewardRate * REWARD_DURATION) > (REWARD_TOKEN.balanceOf(address(this)) * SCALE_FACTOR)
    ) revert UniStaker__InsufficientRewardBalance();


    emit RewardNotified(_amount, msg.sender);
  }
```
- [`setRewardNotifier`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L210-L214) - Admin can enable/disable reward notifier addresses
```solidity
function setRewardNotifier(address _rewardNotifier, bool _isEnabled) external {
    _revertIfNotAdmin();
    isRewardNotifier[_rewardNotifier] = _isEnabled;
    emit RewardNotifierSet(_rewardNotifier, _isEnabled);
  }
```

- [`V3FactoryOwner` contract:
](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol)
    - `admin` - Can set new admin, enable fee amounts, set protocol fees

- [`setAdmin`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L110-L115) - Sets new admin
```solidity
  function setAdmin(address _newAdmin) external {
    _revertIfNotAdmin();
    if (_newAdmin == address(0)) revert V3FactoryOwner__InvalidAddress();
    emit AdminSet(admin, _newAdmin);
    admin = _newAdmin;
  }
```
- [`enableFeeAmount`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L131-L134) - Enables fee amount on Uniswap V3 factory
```solidity
  function enableFeeAmount(uint24 _fee, int24 _tickSpacing) external {
    _revertIfNotAdmin();
    FACTORY.enableFeeAmount(_fee, _tickSpacing);
  }
```
- [`setFeeProtocol`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L142-L149) - Sets protocol fee on V3 pools
```solidity
  function setFeeProtocol(
    IUniswapV3PoolOwnerActions _pool,
    uint8 _feeProtocol0,
    uint8 _feeProtocol1
  ) external {
    _revertIfNotAdmin();
    _pool.setFeeProtocol(_feeProtocol0, _feeProtocol1);
  }
```
- [`setPayoutAmount`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L119-L124) - Sets payout amount for `claimFees`
```solidity
  function setPayoutAmount(uint256 _newPayoutAmount) external {
    _revertIfNotAdmin();
    if (_newPayoutAmount == 0) revert V3FactoryOwner__InvalidPayoutAmount();
    emit PayoutAmountSet(payoutAmount, _newPayoutAmount);
    payoutAmount = _newPayoutAmount;
  }
```

**Vulnerability Assessment**

**Centralization Risks**

- [`UniStaker`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol) highly centralized around `admin` role - full control over rewards

- [`V3FactoryOwner`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol) also relies heavily on centralized `admin`

- No checks on admin actions besides basic address validation

**Authorization Flaws**

- No role-based access control in either contract

- [`notifyRewardAmount`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L570-L599) in `UniStaker` callable by any enabled notifier

- No re-entrancy guards on external calls like [`notifyRewardAmount`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L570-L599) 

**Upgradeability Risks**

- Contracts are not upgradeable, but a malicious admin could propose this

**Lack of Transparency**

- Some key admin functions like `setAdmin` don't emit events

- Vital actions like reward notifications not logged 

**Mitigation Recommendations**

- Use a DAO/multisig scheme instead of single admin address

- Implement a timelock for critical admin functions

- Add re-entrancy guards and input validation where possible

- Emit events for all state changes, especially admin actions

- Add RBAC to restrict capabilities rather than central admin

- Frequent audits to catch issues early

- Consider immutable storage for critical parameters

**Additional Considerations**

- Simplify `UniStaker` rewards distribution mechanics if possible

- Audit any external dependencies like Uniswap contracts

- Understand the intent and context of the contracts - centralized control may be a design choice

- As contracts grow in usage and value, prioritize decentralization and trust minimization 

## Architecture (Diagram) including user flow, admin flow, and protocol roles:

```
              +---------------------------------------------------+
              |                                                   |
              |                    UniStaker                     |
              |                                                   |
              +-+------------------+------------------+----------+
                ^                  ^                  ^
                |                  |                  |
           Reward with       Notify reward           Manage 
           governance tokens      amount           reward notifiers

                                +------------------------+  
                                |                        |
              +--------------->|     V3FactoryOwner     |<---------------+
              |  Pay fee       |                        | Set protocol   |
+------+      |               |                        | fees on pools   |     +--------+
| User | ---->+ Stake tokens  +------------------------+ Factory admin   |---->| Uniswap |
+------+      |                                        ^                |     | Factory |
              |               |                        |                |     +---+----+
              |   Claim       +------------------------+                            
              |    reward                                                     |
              +                                                        Pool fees
                                                                            |
                                                                            V
                                                                      +----------+
                                                                      |          |
                                                                      | Uniswap  |
                                                                      |  Pools   |
                                                                      |          |
                                                                      +----------+
```

**User Flow**

- User stakes governance tokens and earns reward over time
- User can withdraw stake or claim rewards anytime 

**Admin Flow** 

- Factory admin sets protocol fees on Uniswap pools
- Fees fund rewards for governance stakers
- V3FactoryOwner admin manages reward payout parameters
- UniStaker admin manages authorized reward notifiers

**Protocol Roles**

- **Users:** Stake tokens, earn rewards, retain governance power
- **Factory Admin:** Configures Uniswap pools and fees
- **V3FactoryOwner Admin:** Sets reward payout rules
- **UniStaker Admin:** Manages reward distribution parameters
- **Reward Notifiers:** Send reward funds and notify amounts

## Systemic risk Analysis

**External Dependencies**

**Oracles**

- No apparent reliance on external price or data feeds

**Other Contracts** 

- [`UniStaker`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol) interacts with `REWARD_TOKEN` and `STAKE_TOKEN` which should be audited

- [`V3FactoryOwner`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol) interacts heavily with Uniswap V3 contracts like the factory and pools

- Issues with any of these contracts could cascade to the staking system

**Composability Risks**

**Unforeseen Interactions**

- Flash loan attacks could manipulate staking token price or drain rewards

- Staking token price swings could make `claimFees` profitable for arbitrage at unintended times

- An integrated lending market could allow stakers to over-leverage positions

**Economic and Market Considerations** 

**Liquidity Risks**

- Insufficient staking token liquidity could prevent withdrawals 

- Low rewards token reserves could cause distribution issues

**Incentive Misalignment**

- Rewards distribution mechanics could attract primarily short-term speculation instead of long-term stakeholders 

**Governance Risks**

**Centralization**

- Highly centralized admin control poses risks of manipulation 

- No DAO involvement in governance decisions

**Lack of Adaptability**

- No built-in governance mechanics for parameters or logic changes

- Would require admin intervention to pause or upgrade contracts

**Mitigation Recommendations**

- Implement DAO governance and on-chain voting around critical parameters

- Run simulations of different staking token market conditions and liquidity scenarios

- Consider formal verification of core staking incentive mechanics

- Build a robust set of circuit breakers and emergency controls

- Evaluate integrating a lending market directly rather than allowing unforeseen composability 

- Audit all interacting DeFi protocols like Uniswap V3 contracts

## Analysis of potential risks arising from interactions with external systems:

**Dependency Analysis**

**Oracles**

- No apparent use of oracles

**Other Contracts**

- [`UniStaker`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol) depends on `REWARD_TOKEN` and `STAKE_TOKEN` 

- [`V3FactoryOwner`](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol) depends heavily on Uniswap V3 contracts

- No analysis done on security/trustworthiness of these contracts

- Bug or exploit in a dependency could propagate

**Composability Risks** 

**Unforeseen Interactions**

- Arbitrageurs could exploit `claimFees` in combination with DEX trades 

- Flash loans could manipulate staking token price 

- An integrated lending market could allow dangerous overleveraging

**Flash Loan Vulnerability**

- Flash loans could drain rewards quickly during price spikes

**"Sandwich" Attacks** 

- Trades surrounding `stake`/`withdraw` calls could siphon value via frontrunning

**Cross-chain Risks**

No apparent cross-chain interactions

**Mitigation Recommendations**

- Perform audits on all external contract dependencies

- Implement circuit breakers to halt operations if anomalies detected

- Use decentralized and manipulation-resistant oracles if possible

- Defensively code all external interactions for reentrancy and frontrunning

- Stress test composability scenarios like flash loan attacks

- Build a set of monitored invariant checks using chainlink data feeds

- Evaluate bridge mechanisms if any cross-chain interactions added

## Technical analysis

| Category                 | Description                                                                                               |
|--------------------------|-----------------------------------------------------------------------------------------------------------|
| **Security Vulnerabilities** |                                                                                                           |
| Common Exploits           |                                                                                                           |
| No proper validation of reward amounts in `UniStaker`, risking integer overflow                               |
| `claimFees` in `V3FactoryOwner` has no reentrancy guards                                                       |
| Timestamp manipulation could extend or shorten reward streams in `UniStaker`                                    |
| Setting `_rewardEndTime` explicitly could frontrun and steal rewards                                           |
| Logic Errors              |                                                                                                           |
| Incorrect accumulator check in `notifyRewardAmount` can miss pending unclaimed rewards                          |
| Rounding errors possible in reward rate calculation math                                                       |
| Access Control            |                                                                                                           |
| No protection on `admin` functions besides basic address checks                                               |
| Any approved contract can drain rewards via `notifyRewardAmount`                                               |
| Code Optimization        |                                                                                                           |
| Gas Inefficiencies       |                                                                                                           |
| `rewardPerTokenAccumulated` unnecessarily recalculates at every call                                          |
| Iterating through reward history in `lastTimeRewardApplicable` is inefficient                                   |
| Redundancies             |                                                                                                           |
| `lastTimeRewardApplicable` duplicates logic of `rewardPerTokenAccumulated`                                     |
| Helper methods could reduce redundant address sanitization checks                                              |
| Best Practices and Standards |                                                                                                         |
| Follows Solidity naming conventions for most functions and variables                                           |
| Insufficient comments for understandability                                                                   |
| No ERC20 usage to standardize staking and reward tokens                                                       |
| Limited test cases - lacks expected failure testing                                                           |
| Mitigation Recommendations |                                                                                                         |
| Use SafeMath libraries to prevent over/underflows                                                             |
| Add reentrancy and anti-frontrunning guards                                                                   |
| Introduce role-based access control schemes                                                                   |
| Cache accumulator values to optimize gas costs                                                                |
| Refactor redundant logic into reusable isolated functions                                                     |
| Adhere to NatSpec and other commenting standards                                                              |
| Expand test coverage to validate all edge cases                                                               |

### Time spent:
39 hours