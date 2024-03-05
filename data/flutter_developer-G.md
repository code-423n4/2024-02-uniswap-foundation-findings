###### ----SUMARRY----### 
## SMART CONTRACTS THANKS CODE4RENA.COM 


NO |  ISSUE | INSTANCE 
|  |        |   

|[G-01] This contract manages the distribution of rewards to stakers.

|[G-02] A contract that can serve as the owner of the Uniswap v3 factory and manages configuration and collection of protocol pool fees.


|[G-03] A dead-simple contract whose only purpose is to hold governance tokens on behalf of stakers while delegating voting power to a specific delegatee.

|[G-04]	A subset of the ERC20Votes-style governance token to which UNI conforms.


|[G-05] The communication interface between the V3FactoryOwner contract and the UniStaker contract.

|[G-06] 	Required subset of the interface for the Uniswap V3 Factory.

| [G-07] Interface for pool methods that may only be called by the factory owner.|




##### Code description.....
#### Gas Optimization 


## |[G-01] This contract manages the distribution of rewards to stakers.|
```solidity
file: src/UniStaker.sol
```
755   lastCheckpointTime = lastTimeRewardDistributed();
```
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol#L755
```



## |[G-02] A contract that can serve as the owner of the Uniswap v3 factory and manages configuration and collection of protocol pool fees.|


```solidity
file: src/V3FactoryOwner.sol
```
148 _pool.setFeeProtocol(_feeProtocol0, _feeProtocol1);
```
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L148
```


## [G-03] A dead-simple contract whose only purpose is to hold governance tokens on behalf of stakers while delegating voting power to a specific delegatee.|


```solidity
file: src/DelegationSurrogate.sol
```
27    _token.approve(msg.sender, type(uint256).max);

```
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/DelegationSurrogate.sol#L27
```


## [G-04]	A subset of the ERC20Votes-style governance token to which UNI conforms.|
```solidity
file: src/interfaces/IERC20Delegates.sol
```
18 - 26
  function permit(
    address owner,
    address spender,
    uint256 rawAmount,
    uint256 deadline,
    uint8 v,
    bytes32 r,
    bytes32 s
  ) external;


```
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/IERC20Delegates.sol#L18-L26
```


## [G-05] The communication interface between the V3FactoryOwner contract and the UniStaker contract. |

```solidity
file: src/interfaces/INotifiableRewardReceiver.sol
```
10-14
interface INotifiableRewardReceiver {
  /// @notice Method called to notify a reward receiver it has received a reward.
  /// @param _amount The amount of reward.
  function notifyRewardAmount(uint256 _amount) external;
}
```
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/INotifiableRewardReceiver.sol#L10-L14
```



## [G-06] 	Required subset of the interface for the Uniswap V3 Factory.|

```solidity
file: src/interfaces/IUniswapV3FactoryOwnerActions.sol
```
 18 function setOwner(address _owner) external;

```
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/IUniswapV3FactoryOwnerActions.sol#L18
```



## [G-07] Interface for pool methods that may only be called by the factory owner.|

```solidity
file:  src/interfaces/IUniswapV3PoolOwnerActions.sol
```
12  function setFeeProtocol(uint8 feeProtocol0, uint8 feeProtocol1) external;

```
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/interfaces/IUniswapV3PoolOwnerActions.sol#L12
```