## Conceptual Overview

The UniStaker Infrastructure project offers a unique integration with the Uniswap V3 ecosystem, focusing on the collection and distribution of protocol fees through UNI token staking. At its core, the project enables UNI token holders to stake their tokens in a trustless manner, facilitating participation in Uniswap's governance while earning rewards derived from the protocol's fee collection mechanisms. Upon depositing tokens, users can delegate their governance voting power, retaining their influence within the Uniswap ecosystem without directly participating in the voting process. This delegation is managed through the innovative use of Delegation Surrogates, ensuring each staker's governance rights are preserved. Rewards are distributed in a designated token, which is acquired through the auction-like mechanism of fee collection from Uniswap V3 pools, allowing stakers to benefit financially from their participation. From a user's perspective, engaging with the UniStaker Infrastructure means accessing a seamless blend of staking rewards, governance participation, and the opportunity to contribute to the liquidity and efficiency of the Uniswap protocol.

[![Conceptual.png](https://i.postimg.cc/JzVNc54F/Conceptual.png)](https://postimg.cc/KRf3mLT7)

## Technical Overview

From a technical standpoint, the UniStaker Infrastructure project is orchestrated through a series of smart contracts and interfaces that work in conjunction to enable UNI token staking, governance delegation, fee collection, and reward distribution. At the core, the `UniStaker.sol` contract manages the staking of UNI tokens, employing functions like `stake(uint256 _amount, address _delegatee)` for token deposits and enabling users to delegate their governance voting power to a chosen delegatee. This delegation leverages the `DelegationSurrogate.sol` contract, which holds the staked tokens and delegates voting power, ensuring that users' governance rights are preserved according to the `IERC20Delegates` interface that facilitates governance actions such as voting delegation.

The `V3FactoryOwner.sol` contract plays a pivotal role in protocol fee collection from the Uniswap V3 pools. It acts as the owner of the Uniswap V3 Factory, with the ability to set and collect protocol fees through functions like `claimFees(...)`, directly influencing the rewards pool available for distribution to stakers. This fee collection mechanism is designed to be dynamic, allowing fee adjustments and claiming operations to be performed in response to governance actions and market conditions, facilitated by the `IUniswapV3FactoryOwnerActions` and `IUniswapV3PoolOwnerActions` interfaces for factory and pool interactions, respectively.

Reward distribution is intricately handled by the `UniStaker.sol` contract, which calculates and distributes rewards based on the staked amounts and the fees collected. The `notifyRewardAmount(uint256 _amount)` function, as defined in the `INotifiableRewardReceiver` interface, is crucial for updating the reward pool in response to fee collection activities. Users interact with these functionalities primarily through the `UniStaker.sol` contract, which abstracts the complexities of governance delegation and reward calculations, presenting a streamlined interface for depositing tokens, earning rewards, and participating in governance indirectly.

From a user's perspective, interacting with the project involves engaging with these contracts through transactions and function calls that deposit tokens, delegate voting power, claim rewards, and, indirectly, participate in the fee collection process. The underlying logic, encapsulated in smart contracts and governed by the interfaces, ensures that users can seamlessly contribute to and benefit from the Uniswap V3 ecosystem while retaining their governance influence and earning rewards for their participation.


[![technical.png](https://i.postimg.cc/DZtLctPf/technical.png)](https://postimg.cc/KkD1GpWC)

## Architecture of the project

Diving deeper into the architecture and logic of the UniStaker Infrastructure project, we focus on the innovative implementations and how they achieve the project's goals. This discussion emphasizes the underlying mechanisms without redundant explanations of purpose.

[![tot.png](https://i.postimg.cc/wT0WHB6S/tot.png)](https://postimg.cc/pp5QJR5Z)

This diagram aims to simplify the structure and interactions within the UniStaker Infrastructure project, focusing on the relationships between the main contracts (UniStaker.sol, DelegationSurrogate.sol, V3FactoryOwner.sol) and their connections to interfaces and other contracts (IERC20.sol, IERC20Delegates.sol, INotifiableRewardReceiver.sol, IUniswapV3FactoryOwnerActions.sol, IUniswapV3PoolOwnerActions.sol). It abstracts away specific user actions to provide a clearer overview of the system's architecture and the flow of interactions among its components.

### Logic Behind Staking and Rewards Distribution

In `UniStaker.sol`, the staking mechanism is not just a simple token lock-up. The contract manages a complex interaction between staking, delegation, and rewards:

1. **Stake and Delegate**: The Stake and Delegate functionality within the UniStaker project leverages a sophisticated approach that allows UNI token holders to participate in staking rewards while preserving their governance rights through delegation. This dual functionality is achieved through a combination of smart contract interactions, primarily involving the `UniStaker` contract and the `DelegationSurrogate` contracts. Let's break down the key components and functions that facilitate this feature.

### The Stake Function

The entry point for users wishing to stake their UNI tokens and delegate governance rights is the `stake` function in the `UniStaker` contract. There are variations of this function to accommodate different user needs, but at its core, the function works as follows:

```solidity
function stake(uint256 _amount, address _delegatee, address _beneficiary) external returns (DepositIdentifier _depositId);
```

- **_amount**: The quantity of UNI tokens the user wishes to stake.
- **_delegatee**: The address to which the user wishes to delegate their governance rights.
- **_beneficiary**: The address that will receive staking rewards (often the staker themselves).


[![stake.png](https://i.postimg.cc/ZR23b19t/stake.png)](https://postimg.cc/R61qGpQX)




### Key Steps in Stake Function

1. **Token Transfer**: The function first ensures that the specified amount of UNI tokens are transferred from the staker's address to the `UniStaker` contract. This transfer is predicated on the staker having previously approved the `UniStaker` contract to spend the requisite amount of UNI tokens on their behalf.

2. **DelegationSurrogate Handling**: Upon successful transfer, the contract then either identifies an existing `DelegationSurrogate` associated with the specified delegatee or deploys a new one if none exists. This surrogate serves a crucial role by holding the staked UNI tokens and delegating the associated governance rights to the delegatee.

3. **Delegation Execution**: The `DelegationSurrogate` contract, upon deployment or selection, delegates the governance rights of the staked tokens to the specified delegatee. This is achieved through a call to the `delegate` function of the UNI token, which is an ERC20 token with extended functionality for delegation, as defined in the `IERC20Delegates` interface.

### DelegationSurrogate Contract

The `DelegationSurrogate` contract is a minimalist contract designed solely for the purpose of holding UNI tokens and delegating their voting power. Its constructor looks like this:

```solidity
constructor(IERC20Delegates _token, address _delegatee)
```

Upon deployment, the `DelegationSurrogate` contract immediately delegates the voting power of any UNI tokens it holds to the specified `_delegatee`. This ensures that as soon as UNI tokens are transferred to this contract by the `UniStaker` contract, their governance power is effectively delegated.

### Governance Power Retention

By using the `DelegationSurrogate`, the `UniStaker` infrastructure ensures that stakers do not lose their governance influence within the Uniswap protocol. This design choice underscores a commitment to preserving the decentralized governance model of Uniswap, even as it provides mechanisms for users to earn rewards through staking.

In summary, the "Stake and Delegate" feature is a testament to the thoughtful architecture of the UniStaker project, balancing the dual goals of incentivizing protocol participation through staking rewards while preserving the decentralized governance ethos of the Uniswap ecosystem. Through the strategic use of `UniStaker` and `DelegationSurrogate` contracts, it achieves a seamless integration of staking and governance delegation functionalities.

2. **Reward Calculation and Distribution**: The `notifyRewardAmount` function triggers the reward distribution logic. It recalculates the reward rate based on the new amount and the remaining distribution time, ensuring fair distribution among stakers. This dynamic adjustment is crucial for maintaining an equitable system, especially when new rewards are added periodically.

```solidity
if (block.timestamp >= rewardEndTime) {
    scaledRewardRate = (_amount * SCALE_FACTOR) / REWARD_DURATION;
} else {
    uint256 remaining = scaledRewardRate * (rewardEndTime - block.timestamp);
    scaledRewardRate = (remaining + _amount * SCALE_FACTOR) / REWARD_DURATION;
}
```

This snippet shows how the contract adjusts the reward rate based on new rewards and the time remaining in the distribution period. It's a clear demonstration of the project's commitment to fair and dynamic reward distribution.

### Fee Collection and Reward Notification Logic in V3FactoryOwner.sol

The `V3FactoryOwner.sol` contract introduces a novel approach to fee collection:

1. **Fee Collection Mechanism**: The `claimFees` function allows anyone to claim fees from a Uniswap V3 pool by paying a predetermined amount of a specific token. This mechanism creates a marketplace for fee collection, incentivizing users to monitor pools and claim fees when profitable.

2. **Integration with UniStaker**: Upon fee collection, the contract notifies the `UniStaker.sol` contract of the payout, seamlessly integrating fee collection with reward distribution.

```solidity
PAYOUT_TOKEN.safeTransferFrom(msg.sender, address(REWARD_RECEIVER), payoutAmount);
REWARD_RECEIVER.notifyRewardAmount(payoutAmount);
```

This code snippet demonstrates the direct linkage between fee collection and reward notification. By transferring the payout amount to the `UniStaker.sol` contract and notifying it, the system ensures that collected fees are immediately available for distribution as staking rewards.

### Surrogate Delegation Efficiency

The `DelegationSurrogate.sol` contract encapsulates the delegation logic efficiently:

1. **Instant Delegation and Approval**: Upon creation, the contract immediately delegates all voting power to the specified delegatee and approves the `UniStaker.sol` contract to move tokens as necessary. This design minimizes the steps required for delegation, streamlining the process.

The architecture cleverly separates concerns, allowing each contract to focus on a specific aspect of the system (staking, delegation, fee collection) while ensuring seamless interaction between them. This modularity and integration facilitate a robust and flexible system capable of adapting to the evolving DeFi landscape.

In summary, the UniStaker Infrastructure project leverages smart contract interactions to create a comprehensive system for staking, governance participation, and reward distribution within the Uniswap V3 ecosystem. The architecture not only addresses the immediate functionalities but also introduces mechanisms (like the marketplace for fee collection) that add depth to the ecosystem, encouraging active participation and engagement from the community.


## Some Calculations
In the UniStaker Infrastructure project, numerical calculations play a crucial role, especially in the context of rewards distribution and the delegation mechanism. Here’s a brief exploration of how these calculations are implemented:

### Reward Distribution Calculations

1. **Scaled Reward Rate**: The reward rate is a fundamental concept where the total amount of rewards to be distributed is divided by the rewards distribution period (in seconds), resulting in a reward rate per second. This rate is then used in further calculations to determine how much reward each staker earns over time. If the total rewards are 1000 tokens to be distributed over 30 days, with a day being 86400 seconds, the daily reward rate would be `1000 / (30 * 86400)` tokens per second.

2. **Reward Per Token**: This calculation is key to determining the amount of rewards each token is entitled to. It’s computed by integrating the reward rate over time against the total staked tokens. For example, if the total staked tokens are 10,000, and the reward rate is 0.000011574 tokens per second (from the previous example), the reward per token over one day would be `0.000011574 * 86400 / 10000`, assuming a constant total staked amount.

3. **Individual Staker Rewards**: A staker’s reward is calculated by the difference in the reward per token since their last update (either stake, unstake, or reward claim) multiplied by their staked amount. This mechanism ensures that rewards are distributed based on the duration and amount of staking, aligning incentives for long-term and significant participation.

### Precision Handling with Scale Factor

To mitigate precision loss from division operations in the reward calculation, a scale factor (e.g., 1e18 for Ethereum) is used. This involves multiplying values by the scale factor before division operations and adjusting back afterward. This approach ensures that rewards are calculated and distributed with minimal rounding errors, crucial for maintaining fairness in the rewards distribution process.

### Governance Power through Delegation

While not involving complex numerical calculations, the delegation mechanism is critical for maintaining governance power. The `DelegationSurrogate` contracts facilitate this by holding staked tokens and delegating the voting power to the chosen delegatee. This system ensures that the governance influence of stakers is preserved, allowing them to participate in protocol governance effectively while their tokens are staked.



### Time spent:
10 hours