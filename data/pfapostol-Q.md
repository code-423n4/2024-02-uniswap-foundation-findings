### 1. `payoutAmount` setter should be limited at adequate values.

NOTE: Although the contract will be used in the Uniswap ecosystem, and will be controlled by the governance timelock, ensuring that invalid values cannot be entered is a preferred security measure in case the timelock is hacked, or if the contract is later used in other contexts. (The presence of a null check in the current version suggests that the sponsor also cares about the security of the function, regardless of how it is integrated.)

- Setting `payoutAmount` to extremely low values will cause `claimFees` to be called constantly, which will result in rewards never being paid out, as every time a new reward is notified, the old ones are stretched out for the next 30 days distribution.
- Setting `payoutAmount` values very high will result in `claimFees` never being called and rewards never being distributed and fees never being collected.

```solidity
119 function setPayoutAmount(uint256 _newPayoutAmount) external {
120    _revertIfNotAdmin();
121    if (_newPayoutAmount == 0) revert V3FactoryOwner__InvalidPayoutAmount();
122    emit PayoutAmountSet(payoutAmount, _newPayoutAmount);
123    payoutAmount = _newPayoutAmount;
124  }
```

### 2. Usage of `SafeERC20` is not necessary in current context of the project

Readme and the sponsor confirm that only the WETH token will be used as a token to implement the reward logic, at least for the time being.
Because the WETH token is secure (transfer is IERC20 compliant and the token always returns true). Using SafeERC20 creates an additional external dependency that does not bring any benefit to the project, but increases the cost, and (potentially) increases the risk for the contract (in case a vulnerability is discovered in SafeERC20).

LoC:

- `src/UniStaker.sol`

```solidity
624     SafeERC20.safeTransferFrom(IERC20(address(STAKE_TOKEN)), _from, _to, _value);
```

- `src/V3FactoryOwner.sol`

```solidity
187    PAYOUT_TOKEN.safeTransferFrom(msg.sender, address(REWARD_RECEIVER), payoutAmount);
```

### 3. Variable not used in business logic

`depositorTotalStaked` only used to keep the amount of tokens deposited by particular user and has not other purposes  except for displaying this information. This increase complexity of code, while serving no practical purpose.

```solidity
146  mapping(address depositor => uint256 amount) public depositorTotalStaked;
```