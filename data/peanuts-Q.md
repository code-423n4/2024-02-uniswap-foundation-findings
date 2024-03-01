### [L-01] PAYOUT_TOKEN cannot be changed

In V3FactoryOwner.sol, `PAYOUT_TOKEN` is currently set as WETH (stated in the documents). However, it is not sustainable to only have WETH as the `PAYOUT_TOKEN`, in case WETH experiences a flash crash. In that case, people will move away from interacting with WETH, which will destroy the whole unistaker architecture.

```
  /// @notice The ERC-20 token which must be used to pay for fees when claiming pool fees.
  IERC20 public immutable PAYOUT_TOKEN;
```

If possible, allow other `PAYOUT_TOKEN` to be used, like a stablecoin. Of course, the change is subjected to the Timelock contract.

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/V3FactoryOwner.sol#L99-L101

### [L-02] The owner of V3FactoryOwner cannot be changed anymore

In the normal UniswapV3Factory.sol, there is a function to set the owner.

```
    function setOwner(address _owner) external override {
        require(msg.sender == owner);
        emit OwnerChanged(owner, _owner);
        owner = _owner;
    }
```

Once the V3FactoryOwner takes over the UniswapV3Factory owner (Timelock contract) as the new owner, the owner cannot be changed anymore. This can be dangerous in the event that the V3FactoryOwner contract is compromised.

Consider adding a function in V3FactoryOwner to call `setOwner()` in UniswapV3Factory to allow for flexibility.

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/V3FactoryOwner.sol#L99

### [L-03] The surrogate contract is constantly having the approval set to max.

When staking UNI tokens into the UniStaker contract, the staker has to deposit tokens into a surrogate contract. The function fetchOrDeploySurrogate takes in the delegatee address (the person that the staker wants to pass the voting power to) and creates a new surrogate or skips the creation if the surrogate contract is created.

```
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

If many users decide to stake their voting rights to the same delegatee, then the approval will keep approving to max.

```
contract DelegationSurrogate {
  /// @param _token The governance token that will be held by this surrogate
  /// @param _delegatee The address of the would-be voter to which this surrogate will delegate its
  /// voting weight. 100% of all voting tokens held by this surrogate will be delegated to this
  /// address.
  constructor(IERC20Delegates _token, address _delegatee) {
    _token.delegate(_delegatee);
>   _token.approve(msg.sender, type(uint256).max);
  }
}
```

Best practice is for the contract to check if the approval is already set at max (note that for uni, uint96 is the max, not uint256), and if so, then skip the approval.
