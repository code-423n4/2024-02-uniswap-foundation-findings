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

### [L-04] Note that collectProtocol collects everything but 1 wei. Fee collectors who aren't aware may get their transactions reverted

When calling collectProtocol, it is assumed by the fee collector that `amount0Requested` and `amount1Requested` should be everything in the pool to maximise their gains.

```
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
```

However, note that the fee amount actually collected is everything but 1 wei, because Uniswap doesn't want the slot to be cleared for gas savings

```
        if (amount0 > 0) {
>           if (amount0 == protocolFees.token0) amount0--; // ensure that the slot is not cleared, for gas savings
            protocolFees.token0 -= amount0;
            TransferHelper.safeTransfer(token0, recipient, amount0);
        }
        if (amount1 > 0) {
            if (amount1 == protocolFees.token1) amount1--; // ensure that the slot is not cleared, for gas savings
            protocolFees.token1 -= amount1;
            TransferHelper.safeTransfer(token1, recipient, amount1);
        }
```        

When calling `amount0Requested`, note that the amount must be totalAmount - 1 wei.

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L181-L190
https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#L856-L865


### [L-05] DelegationSurrogate is a convenient way to bypass the delegation requirement

A user can appoint anyone as the delegatee when staking uni tokens. When the delegatee is appointed, a surrogate contract will be created, and the staker will deposit the uni tokens into the surrogate. This surrogate contract will then delegate the uni tokens to the delegatee.

```
contract DelegationSurrogate {
  /// @param _token The governance token that will be held by this surrogate
  /// @param _delegatee The address of the would-be voter to which this surrogate will delegate its
  /// voting weight. 100% of all voting tokens held by this surrogate will be delegated to this
  /// address.
  constructor(IERC20Delegates _token, address _delegatee) {
    _token.delegate(_delegatee);
    _token.approve(msg.sender, type(uint256).max);
  }
```

This is a convenient way to bypass the normal delegation process. In the normal delegation process, the token holder can only delegate to one user through the delegate function.

```
    function delegate(address delegatee) public {
        return _delegate(msg.sender, delegatee);

    function _delegate(address delegator, address delegatee) internal {
        address currentDelegate = delegates[delegator];
        uint96 delegatorBalance = balances[delegator];
>       delegates[delegator] = delegatee;

        emit DelegateChanged(delegator, currentDelegate, delegate
        _moveDelegates(currentDelegate, delegatee, delegatorBalance);
    }
    }
```

Previously, if user A has 100 UNI tokens and wants to delegate 50 tokens to user B and 50 tokens user to C, he cannot do that. He can donate only 100 tokens to either user B or C. Now, the user can delegate to any delegatees that he wants using the staking contract. He can delegate 10 tokens to user B, 10 tokens to user C, and 80 tokens to user D, bypassing the original intention of only allowing the token holder to delegate to one user.

UNI token: https://etherscan.io/token/0x1f9840a85d5af5bf1d1762f925bdaddc4201f984#code
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/UniStaker.sol