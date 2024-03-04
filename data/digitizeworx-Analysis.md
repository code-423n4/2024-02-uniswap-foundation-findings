## Overview

The UniStaker system aims to incentivize long-term UNI token holders to stake their governance tokens and participate in protocol governance. It consists of two core components:

1. **UniStaker Contract:** Distributes staking rewards and manages voting power delegation. Users can stake UNI, assign voting power to chosen delegates, and earn designated reward tokens over time.

2. **V3FactoryOwner Contract:** Facilitates reward collection from Uniswap V3 pool protocol fees. Users can pay a set amount of tokens to claim fees, which are forwarded to the UniStaker as rewards.

**UniStaker Contract**

The UniStaker manages the core staking and incentive mechanics via key data structures tracking deposits, delegate mappings, accumulator values, and rewards:

## Scope
| Contract                  | Purpose                                                                                          |
|---------------------------|--------------------------------------------------------------------------------------------------|
| [UniStaker.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol)             | This contract manages the distribution of rewards to stakers.                                   |
| [V3FactoryOwner.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol)        | A contract that can serve as the owner of the Uniswap v3 factory and manages configuration and collection of protocol pool fees. |
| [DelegationSurrogate.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/DelegationSurrogate.sol)   | A dead-simple contract whose only purpose is to hold governance tokens on behalf of stakers while delegating voting power to a specific delegatee. |
| [IERC20Delegates.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/IERC20Delegates.sol)       | A subset of the ERC20Votes-style governance token to which UNI conforms.                         |
| [INotifiableRewardReceiver.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/INotifiableRewardReceiver.sol) | The communication interface between the V3FactoryOwner contract and the UniStaker contract.     |
| [IUniswapV3FactoryOwnerActions.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/IUniswapV3FactoryOwnerActions.sol) | Required subset of the interface for the Uniswap V3 Factory.                                    |
| [IUniswapV3PoolOwnerActions.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/IUniswapV3PoolOwnerActions.sol)   | Interface for pool methods that may only be called by the factory owner.                          |

[UniStaker.sol#struct Deposit](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L94-L99), [UniStaker.sol#156](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L156), [UniStaker.sol#159](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L159)

```solidity
// Metadata per deposit
struct Deposit {
  uint256 balance;
  address owner; 
  address delegatee;
  address beneficiary; 
}

// Surrogate contract per delegate  
mapping(address delegatee => DelegationSurrogate surrogate) public surrogates;

// Accrued rewards per beneficiary
mapping(address => uint256) public unclaimedRewardCheckpoint;  

// Global rewards distribution state
uint256 public rewardRate; 
uint256 public rewardEndTime;
uint256 public rewardPerTokenStored;
```

**Key Risks**

- **Centralized Administration:** The `admin` holds excessive control over critical functionality like reward distribution and delegate management: [UniStaker.sol#140](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L140),  [UniStaker.sol#setRewardNotifier](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L210-L214),  [UniStaker.sol#notifyRewardAmount](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L570-L599)  

    ```solidity
    // Administrator controls
    address public admin;

    // Admin privileged functions
    function setRewardNotifier(address _rewardNotifier, bool _isEnabled) external;
    function notifyRewardAmount(uint256 _amount) external; 
    ```

    - This could lead to mismanagement, abuse, or a governance coup.

-   **Mismanaged Reward Distribution:** Invalid or malicious reward notification: [UniStaker.sol#notifyRewardAmount](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L570-L599)

    ```solidity
    // Notifier simply invokes this 
    function notifyRewardAmount(uint256 _amount) external;
    ```
    
    - Reward rates are updated purely via external calls with no validation, controls, or transparency. Faulty data or exploitable logic here can break core functionality.
    
- **Compose-ability Risks:** 

    - Arbitrageurs could extract significant value by combining reward collection on V3 pools with DEX trades.
        
    - Flash loan attacks could rapidly drain rewards by manipulating staking token prices.
    
- **Economic Risk Factors:**

    - Insufficient staking token liquidity for withdrawals.
    
    - Market volatility impacting rewards model sustainability.
    
- **Upgradeability Issues:**

    - Lack of upgradeability could limit ability to fix issues. But malicious upgrades could also backdoor the contract.
    
**Mitigations**

- **Time Delayed Administration:** Require a timelock delay for admin functions to reduce impulsiveness and provide transparency.

- **DAO Voting Control:** Transition admin capabilities to a DAO governance process with decentralized voting.

- **Validation Scope Limiting on External Calls:** Apply tighter constraints on permissible parameters for things like `notifyRewardAmount`.

- **Composability Circuit Breakers:** Pause functionality if anomalous conditions are algorithmically detected.

- **Frequent Security Audits:** Promote regular formal verification to validate consistency of rewards model with monk markets.  

**V3FactoryOwner Contract**  

This contract owns the Uniswap V3 factory and pools, allowing it to configure and collect protocol fees. Third parties can call `claimFees` to receive fees in exchange for a payment sent to UniStaker as rewards.

**Key Risks**

- **Centralized Administration:** `admin` has total control again to add risky new functionality under a veil of legitimacy.

- **Manipulable Payout Oracle:** `setPayoutAmount` sets the reward payment without any validation logic: [UniStaker.sol#setPayoutAmount](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L119-L124)

    ```solidity
    // Admin sets payout amount without validation
    function setPayoutAmount(uint256 _newPayoutAmount) external; 
    ```

- **Asset Security Risk:** If the admin ever transfers factory or pool ownership, the entire fee collection process is compromised.

**Mitigations** 

- Apply mitigations from UniStaker section universally across all core contracts.

- **Trust-minimized Payout Oracles:** Use a manipulation-resistant price feed for determining payout amount.

- **Ownership Locked Asset Management:** One-way burn admin capabilities for factory ownership management preventing lateral transfer.  







## Admin Privileges

- `UniStaker.sol`

    - `admin` - Can set new admin, enable/disable reward notifiers

    - `notifyRewardAmount` - Called by reward notifiers to notify rewards

    - `setRewardNotifier` - Admin can enable/disable reward notifier addresses

- `V3FactoryOwner.sol`

    - `admin` - Can set new admin, enable fee amounts, set protocol fees

    - `setAdmin` - Sets new admin

    - `enableFeeAmount` - Enables fee amount on Uniswap V3 factory

    - `setFeeProtocol` - Sets protocol fee on V3 pools

    - `setPayoutAmount` - Sets payout amount for `claimFees`

## Centralization Risks

- `UniStaker` highly centralized around `admin` role - full control over rewards

- `V3FactoryOwner` also relies heavily on centralized `admin`

- No checks on admin actions besides basic address validation

**Authorization Flaws**

- No role-based access control in either contract

- `notifyRewardAmount` in `UniStaker` callable by any enabled notifier

- No re-entrancy guards on external calls like `notifyRewardAmount` 

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






## Systemic risk

**External Dependencies**

**Other Contracts** 

- `UniStaker` interacts with `REWARD_TOKEN` and `STAKE_TOKEN` which should be audited

- `V3FactoryOwner` interacts heavily with Uniswap V3 contracts like the factory and pools

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




## Dependency

**Other Contracts**

- `UniStaker` depends on `REWARD_TOKEN` and `STAKE_TOKEN` 

- `V3FactoryOwner` depends heavily on Uniswap V3 contracts

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




## Common Exploits

- No proper validation of reward amounts in `UniStaker`, risking integer overflow

- `claimFees` in `V3FactoryOwner` has no reentrancy guards

- Timestamp manipulation could extend or shorten reward streams in `UniStaker` 

- Setting `_rewardEndTime` explicitly could frontrun and steal rewards

**Logic Errors**

- Incorrect accumulator check in `notifyRewardAmount` can miss pending unclaimed rewards

- Rounding errors possible in reward rate calculation math

**Access Control** 

- No protection on `admin` functions besides basic address checks

- Any approved contract can drain rewards via `notifyRewardAmount`

**Code Optimization**  

**Gas Inefficiencies**

- `rewardPerTokenAccumulated` unnecessarily recalculates at every call

- Iterating through reward history in `lastTimeRewardApplicable` is inefficient

**Redundancies**

- `lastTimeRewardApplicable` duplicates logic of `rewardPerTokenAccumulated`

- Helper methods could reduce redundant address sanitization checks

**Best Practices and Standards**

- Follows Solidity naming conventions for most functions and variables

- Insufficient comments for understandability

- No ERC20 usage to standardize staking and reward tokens

- Limited test cases - lacks expected failure testing

## Mitigation Recommendations

- Use SafeMath libraries to prevent over/underflows

- Add reentrancy and anti-frontrunning guards 

- Introduce role-based access control schemes

- Cache accumulator values to optimize gas costs 

- Refactor redundant logic into reusable isolated functions

- Adhere to NatSpec and other commenting standards

- Expand test coverage to validate all edge cases

### Time spent:
25 hours