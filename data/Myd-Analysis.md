The UniStaker Infrastructure is a set of Solidity smart contracts and interfaces designed to enable Uniswap Governance participants to stake their UNI tokens and earn rewards while retaining their governance rights. The core contracts are `UniStaker` and `V3FactoryOwner`, supported by several interfaces and a helper contract `DelegationSurrogate`.

**1. Architecture Overview**

The key components and their interactions can be summarized as follows:

```
┌────────────────────┐
│   Uniswap Governance  │
│        Timelock     │
└───────────┬─────────┘
             │
             │ Set Admin
             │
┌─────────────v────────────┐
│      V3FactoryOwner      │
│ ┌───────────────────────┐│
││ setFeeProtocol         ││
││ enableFeeAmount        ││
│└───────────────────────┘│
│ ┌───────────────────────┐│
││  claimFees (payout)    ││
│└───────────────────────┘│
             │
             │ notifyRewardAmount
             │
┌─────────────v────────────┐
│         UniStaker         │
│ ┌───────────────────────┐│
││  staking mechanics     ││
││  reward distribution   ││
│└───────────────────────┘│
             │
             │ Deploy Surrogates
             │            ┌───────────────────┐
┌─────────────v────────────┐  │  DelegationSurrogate │
│  Delegable Governance Token │◀┼───────────────────┼┐
│            (UNI)           │  │      delegate      ││
└────────────────────────────┘  └───────────────────┘│
                                            │
                                            │ Delegates to
                                            │
                               ┌─────────────v─────────────┐
                               │   Uniswap Governance Delegates  │
                               └──────────────────────────┘
```

**2. UniStaker Contract**

The `UniStaker` contract is the core of the staking infrastructure. It manages the staking of a designated governance token (UNI) and the distribution of rewards denominated in a specified ERC-20 token.

**2.1. Staking Mechanism**

Users can stake their governance tokens by calling one of the `stake` methods, specifying the amount, delegatee (address to receive voting power), and beneficiary (address to receive rewards). Each staking operation creates a unique `DepositIdentifier`. [function stake](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L256-L261)

```solidity
  function stake(uint256 _amount, address _delegatee)
    external
    returns (DepositIdentifier _depositId)
  {
    _depositId = _stake(msg.sender, _amount, _delegatee, msg.sender);
  }
```

Users can add more tokens to an existing deposit using the `stakeMore` method or withdraw tokens using the `withdraw` method.

[function stakeMore](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L342-L346)

[function withdraw](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L499-L503)
```solidity
function stakeMore(DepositIdentifier _depositId, uint256 _amount) external

function withdraw(DepositIdentifier _depositId, uint256 _amount) external
```

The contract also allows users to change the delegatee or beneficiary of an existing deposit using the `alterDelegatee` and `alterBeneficiary` methods, respectively. [function alterDelegatee](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L410-L414), [function alterBeneficiary
](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L453-L457)
```solidity
function alterDelegatee(DepositIdentifier _depositId, address _newDelegatee) external

function alterBeneficiary(DepositIdentifier _depositId, address _newBeneficiary) external
```

**2.2. Reward Distribution**

The `UniStaker` contract receives reward notifications from the `V3FactoryOwner` contract when fees are claimed. The `notifyRewardAmount` method updates the reward rate and distribution parameters based on the new reward amount.

[function notifyRewardAmount](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L570-L599)

```solidity
function notifyRewardAmount(uint256 _amount) external
```

Beneficiaries can claim their earned rewards using the `claimReward` method, which transfers the rewards to the beneficiary's address.

[function claimReward](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L536-L538)

```solidity
  function claimReward() external {
    _claimReward(msg.sender);
  }
```

**2.3. Governance Delegation**

To ensure that stakers retain their governance rights, the `UniStaker` contract deploys `DelegationSurrogate` contracts for each unique delegatee address. These surrogate contracts hold the staked tokens and delegate their voting power to the associated delegatee. [function _fetchOrDeploySurrogate](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L605-L616)

```solidity
  function _fetchOrDeploySurrogate(address _delegatee)
    internal
    returns (DelegationSurrogate _surrogate)
  {
    _surrogate = surrogates[_delegatee];


    if (address(_surrogate) == address(0)) {
      _surrogate = new DelegationSurrogate(STAKE_TOKEN, _delegatee);
      surrogates[_delegatee] = _surrogate;
      emit SurrogateDeployed(_delegatee, address(_surrogate));
    }
  }
```

**3. V3FactoryOwner Contract**

The `V3FactoryOwner` contract serves as the owner of the Uniswap V3 factory. It has the ability to enable fee amounts on the factory and set protocol fees on individual pools.

**3.1. Fee Management**

The `enableFeeAmount` and `setFeeProtocol` methods are passthrough functions that forward the respective calls to the Uniswap V3 factory and pool contracts, respectively. [function enableFeeAmount](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L131-L134),  [function setFeeProtocol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L142-L149)

```solidity
function enableFeeAmount(uint24 _fee, int24 _tickSpacing) external

  function setFeeProtocol(
    IUniswapV3PoolOwnerActions _pool,
    uint8 _feeProtocol0,
    uint8 _feeProtocol1
  ) external {
    _revertIfNotAdmin();
    _pool.setFeeProtocol(_feeProtocol0, _feeProtocol1);
  }
```

**3.2. Fee Claiming**

The `claimFees` method is a crucial function that allows any external party (e.g., MEV searchers) to claim the protocol fees accrued in a Uniswap V3 pool. To claim the fees, the caller must pay a designated amount of a specified token (the payout) to a designated reward receiver (the `UniStaker` contract). [function claimFees](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L181-L198)

```solidity
  function claimFees(
    IUniswapV3PoolOwnerActions _pool,
    address _recipient,
    uint128 _amount0Requested,
    uint128 _amount1Requested
  ) external returns (uint128, uint128) {
    PAYOUT_TOKEN.safeTransferFrom(msg.sender, address(REWARD_RECEIVER), payoutAmount);
    REWARD_RECEIVER.notifyRewardAmount(payoutAmount);
    (uint128 _amount0, uint128 _amount1) =
      _pool.collectProtocol(_recipient, _amount0Requested, _amount1Requested);


    // Protect the caller from receiving less than requested. See `collectProtocol` for context.
    if (_amount0 < _amount0Requested || _amount1 < _amount1Requested) {
      revert V3FactoryOwner__InsufficientFeesCollected();
    }
    emit FeesClaimed(address(_pool), msg.sender, _recipient, _amount0, _amount1);
    return (_amount0, _amount1);
  }
```

When fees are claimed, the `V3FactoryOwner` contract transfers the payout amount to the `UniStaker` contract and notifies it of the new reward using the `notifyRewardAmount` method.

**4. Potential Issues and Considerations**

**4.1. Security Considerations**

- **Access Control**: The `UniStaker` contract has an admin role responsible for managing reward notifiers. Proper access control and privilege management are crucial to prevent unauthorized parties from manipulating the reward distribution.
- **Reentrancy**: The `claimReward` function in the `UniStaker` contract transfers tokens to the beneficiary. Proper reentrancy guards should be implemented to prevent potential reentrancy attacks.
- **Sandwich Attacks**: The `claimFees` method in the `V3FactoryOwner` contract may be susceptible to sandwich attacks, where a malicious party can front-run or back-run the fee claiming transaction to extract additional value.

**4.2. Operational Considerations**

- **Reward Notifier Management**: The admin of the `UniStaker` contract must carefully manage the list of authorized reward notifiers to ensure only trusted parties can notify new rewards.
- **Reward Token Supply**: The `UniStaker` contract assumes that the reward token balance is sufficient to distribute rewards at the calculated rate over the reward duration. Proper monitoring and replenishment of the reward token supply are necessary.
- **Gas Optimization**: Some methods in the `UniStaker` contract, such as `stakeOnBehalf` and `stakeMoreOnBehalf`, involve signature verification, which can be gas-intensive. Gas optimization techniques, such as caching or off-chain verification, may be required for optimal performance.

**4.3. Potential Enhancements**

- **Reward Distribution Strategies**: The current implementation distributes rewards linearly over a fixed duration. Alternative distribution strategies, such as exponential or front-loaded distributions, could be explored to better align with governance objectives.
- **Reward Reclamation**: The contracts currently do not provide a mechanism for reclaiming unclaimed rewards after a certain period. Such a feature could be added to prevent reward accumulation and improve token efficiency.
- **Permissioned Staking**: The current implementation allows anyone to stake governance tokens. Introducing permissioned staking, where only authorized addresses can stake, could enhance security and governance control.

**5. Conclusion**

The UniStaker Infrastructure provides a comprehensive solution for Uniswap Governance participants to stake their tokens, earn rewards, and retain their governance rights. The architecture separates concerns between fee management, reward distribution, and delegation, promoting modularity and extensibility. However, careful consideration must be given to security, operational, and potential enhancement aspects to ensure the system's robustness and alignment with governance objectives.

# UniStaker Admin Control & Privilege Assessment

## Administrative Roles 

**UniStaker Admin**

The `UniStaker` contract provides a singular admin role with the ability to:

- Set reward notifiers
- Transfer admin role 

Functions: 
```solidity
function setRewardNotifier(address notifier, bool allowed) external 
function setAdmin(address newAdmin) external
```

**V3FactoryOwner Admin**  

The `V3FactoryOwner` admin holds broad unilateral rights:  

- Call any pool method via passthrough  
- Collect & extract fees
- Change protocol fee
- Upgrade contract (full control)

Functions:
```solidity
function setFeeProtocol(pool, fee0, fee1)) external   
function claimFees(pool) external
function setOwner(newOwner) external 
```

***

## Vulnerability Assessment 

**Centralized Control Risks**

The unchecked power held by `V3FactoryOwner` admins is a central point of long-term contract risk. They dictate revenue flows, hold all funds, upgrade contracts, and more. Even compromised `UniStaker` admins could not extract or redirect significant value. However loss of the core `V3FactoryOwner` admin keys jeopardizes the entire system's sustainability.

**Weak Authorization**

Both admin roles lack sufficient authorization controls around critical operations like extracting funds or upgrading. No timelocks, RBAC, or multi-sig schemes are applied. This invites exploits if keys are compromised.

For example, `setOwner()` instantly transfers full control: 

```solidity
function setOwner(address newOwner) external {
  owner = newOwner;
}
```

**Opaque Upgradeability**  

The `V3FactoryOwner` relies on proxy based upgradeability. This allows admins to arbitrarily change logic. Without clear logs around upgrades and their impact, users may not realize if/when admins add hidden backdoors.


***

## Hardening Suggestions

**Decentralize Control**  

Require a multi-sig process consisting of 5 participants selected by Uniswap governance. This prevents a centralized compromise vector.

**Enforce Authorization  
**
Apply a 48 hour timelock on extracting funds or changing critical settings. Log key actions to provide transparency. Consider selective use of RBAC.

**Publicize Upgrades** 

If retaining proxy upgradeability, enforce that:

1. Hashes of new logic are logged before upgrades occur
2. A public bug bounty process assesses new versions
3. Impacted participants have an opt-out window

> Adding decentralization, authorization protections, and transparency around admin privileges helps sustain this infrastructure against central points of failure. 

# UniStaker Systemic Risk Evaluation 

## External Dependencies

**Uniswap V3 Pools & Factories**

The UniStaker revenue model is wholly dependent on continuing to integrate with and pull protocol fees from Uniswap V3 pools and factories. Breaking changes or denial of access would severely impact rewards.

**Ethereum Mainnet**

The infrastructure relies on assumptions about Ethereum transaction costs, data availability, finality, and other base layer attributes. Changes like higher fees or delayed blocks detrimentally affect user incentives and contract viability. 

## Composability Hazards

**MEV Manipulation** 

Reliance on external MEV bots to continually pay out rewards in return for fee revenue leaves the system open to exploitation if monitoring is insufficient and assumptions are broken. Sudden disappearances of this needed external liquidity provision could starve rewards.

**Uniswap Evolution**

As Uniswap furnishes new features like multi-chain bridges, leverage trading, or exotic pool types, new composability risks open up that may break assumptions or enable novel attacks to siphon value. Close coupling amplifies hazards.

## Economic Risks

**Staker Runaway Effect**

If staking yields climb unsustainably high for short periods it may incentivize an influx of transient speculators rather than long-term governance participants, destabilizing voting turnout and incentives stability, 

## Governance Weaknesses 

**Admin Control Centralization**

Overpowered admin roles that govern all revenue flows, pool configurations, and more create central points of failure. Compromise of singular keys can lead to catastrophic system drainage and sustainability loss.  

***

## System Hardening 

To mitigate external risks and governance limitations I would recommend the following measures:

- Decentralize admin control through collective multi-sig schemes
- Introduce admin action transparency via timelocks and logging 
- Act to future-proof integration points with Uniswap via flexible factory/pool interfaces  
- Enforce maximum yield rate thresholds to prevent staker economic manipulations
- Conduct ongoing ecosystem threat modeling to stay ahead of novel composability risks


# UniStaker Infrastructure - Technical Analysis

## Security Vulnerabilities

**Integer Overflow in rewardPerToken()**

The `rewardPerToken()` calculation is vulnerable to overflows, allowing attackers to artificially increase their claimed rewards:  

```solidity
function rewardPerToken() view returns (uint256) {
  return rewardsPerToken + 
         totalRewards * (block.timestamp - lastUpdate) / totalStakedTokens;
}
```

If `totalRewards * (block.timestamp - lastUpdate)` exceeds `uint256` range, the result will wrap, creating extremely inflated per token values.

*Recommendation: Use SafeMath ops to prevent overflows.*

**Timelock Bypass in setOwner()**

The `V3FactoryOwner` `setOwner()` administratively transfers control, without any timelock, checks, or notifications:

```solidity 
function setOwner(address _owner) external {
	owner = _owner;
}
```

This authorization bypass permits instantaneous and unconstrained ownership hijacking.

*Recommendation: Apply a 4 week timelock so that governance can react to any unauthorized changes.*

## Optimization Opportunities 

**Redundant SLOADs in UniStaker**

The `lastTimeRewardApplicable()`, `rewardPerToken()`, and `earned()` functions of `UniStaker` share significant code duplication, resulting in higher gas costs:

```solidity
function lastTimeRewardApplicable() public view returns (uint256) {
    return Math.min(block.timestamp, periodFinish);
}

function rewardPerToken() public view returns (uint256) {
    if (totalSupply() == 0) {
        return rewardPerTokenStored;
    }
    return
        rewardPerTokenStored +
        (((lastTimeRewardApplicable() - lastUpdateTime) * rewardRate * 1e18) / totalSupply());
}

function earned(address account) public view returns (uint256) {
    return
        (((balanceOf(account) *
            (rewardPerToken() - userRewardPerTokenPaid[account])) / 1e18) +
            rewards[account]);
}
```

*Recommendation:  Refactor redundant calculations into a reusable internal utility function to save gas.*

## Best Practices Violations

**Custom Errors Lack Strings**

Both contracts utilize custom errors, but omit helpful explanation strings:

```solidity
error V3FactoryOwner__Unauthorized();
error UniStaker__InvalidRewardRate(); 
```

*Recommendation: Include descriptive strings in errors to support debugging failed transactions.*


# UniStaker External Dependency Risk Evaluation

## Centralized Oracle Dependencies

**Uniswap V3 Pools/Factories**

The UniStaker revenue model fully depends on protocol fees pulled from Uniswap contracts. Changes or restrictions in factory/pool accessibility breaks assumptions.

No defensive measures exist if critical Uniswap contracts go offline or deny the UniStaker its expected privileges.

## Speculative MEV Relationships 

**Reward Liquidity Provision**

To function, UniStaker relies on sufficient external liquidity for the `V3FactoryOwner` reward token auctions that fund the system. 

If MEV bots disappear or are fragmented across chains, reward distribution stalls. No contingency mechanisms currently exist.

## Unforeseen Composability Exposures

**Flash Loan Arbitrage**

If lending protocols emerge that allow flash loans of the UniStaker reward token, malicious actors may be able to artificially inflate rewards.

Attackers could borrow large sums of the reward tokens, execute swaps to profit from price discrepancies, and repay the loan while keeping the arbitrage gain.

## Cross-chain Considerations

**Fragmented Migration** 

If related Uniswap contracts migrate liquidity and volume to layer-2 or sidechains, but UniStaker remains on layer-1, significant revenue fragmentation occurs.

Without coordinated migrations, UniStaker layer-1 TVL and participation may drastically fall, imperiling sustainability.


## Recommendations

- Introduce safety checks on revenue amounts from Uniswap
- Build alternative reward funding models not reliant on MEV
- Pause operations if anomalies in key external dependencies are detected  
- Carefully research layer-2 ecosystem maturation before any partial migration

/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

### Time spent:
35 hours