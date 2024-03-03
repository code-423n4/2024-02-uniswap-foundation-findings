# Summary



# Low Risk Issues    
no | Issue |Instances||
|-|:-|:-:|:-:|
| [L-01] |  the function should revert if the amount to be staked is zero | 1 |
| [L-02] |  no zero address checks | 3 |
| [L-03] |  no logic to update important addresses | 3 |

# Non-critical
no | Issue |Instances||
|-|:-|:-:|:-:|
| [N-01] |  The comments of the function should say that the actual owner should pre approve the staked amount not the caller | 1 |
| [N-02] |  the comments should say that The caller must pre-approve the staking contract to spend at least the would-be staked amount of the token | 2 |
| [N-03] |  grammar error in the comment | 1 |


## [L-01] the function should revert if the amount to be staked is zero
The _stake() function has the argument _amount which represents the amount to be staked for the caller, what the function does not do is to check for zero amount, and if amount is zero the function will execute without any problem and certain events will be emitted while the user has not staked anything. The function should restrict such behavior by checking for if the amount to be staked is zero, and revert if it is. 

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L638-L664

## [L-02] no zero address checks 
the constructor of the contract V3FactoryOwner does not check for the zero address, for the addresses of the arguments _factory,_payoutToken, _rewardReceiver. Since the contract is immutable, and there are no functions in the contract to upgrade these addresses, this could be problematic if the addresses are unintentionally set to the zero address.

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L90
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L91
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L93

## [L-03] no logic to update important addresses
in the constructor of the contract V3FactoryOwner, the addresses for the _factory,_payoutToken, _rewardReceiver are set. These are crucial addresses, but the contract has no functions to update these addresses. If in case there is an issue in any of these addresses, or if the DAO wants to update one of these addresses such as the _payoutToken, there will be no logic to update these addresses. The contract should add methods to update these addresses which can only be called by the admin.

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L90
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L91
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L93

## [N-01]  The comments of the function should say that the actual owner should pre approve the staked amount not the caller
The comments of the function stakeOnBehalf() says that the caller must pre-approve the staking contract to spend at least the
would-be staked amount of the token, but in this function the caller is not the real owner of the tokens instead he is given the permission through a signture to stake on behalf of the owner, so he can not approve the contract for the token amount. The owner of the tokens should be the one who pre approves the contract not the caller, and the comments should point out to this. 

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L305-L334

## [N-02] the comments should say that The caller must pre-approve the staking contract to spend at least the would-be staked amount of the token
The other staking functions point to this context that the caller should pre-approve the staking contract in the comments, but the functions stakeMore(), and stakeMoreOnBehalf() does not point to this. These comments should be added for these functions as the other staking functions.

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L336-L346
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L375-L402

## [N-03] grammer error in the comment 
in the comment the word " select " should be changed to " selected " to be grammatically correct

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L27
