## QA-01 Protocol supports WETH but doesn't consider all the nuances attached to it

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/UniStaker.sol#L292-L304

```solidity
  function permitAndStake(
    uint256 _amount,
    address _delegatee,
    address _beneficiary,
    uint256 _deadline,
    uint8 _v,
    bytes32 _r,
    bytes32 _s
  ) external returns (DepositIdentifier _depositId) {
    //@audit permit
    STAKE_TOKEN.permit(msg.sender, address(this), _amount, _deadline, _v, _r, _s);
    _depositId = _stake(msg.sender, _amount, _delegatee, _beneficiary);
  }
```

And https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/UniStaker.sol#L360-L373

```solidity
  function permitAndStakeMore(
    DepositIdentifier _depositId,
    uint256 _amount,
    uint256 _deadline,
    uint8 _v,
    bytes32 _r,
    bytes32 _s
  ) external {
    Deposit storage deposit = deposits[_depositId];
    _revertIfNotDepositOwner(deposit, msg.sender);

    STAKE_TOKEN.permit(msg.sender, address(this), _amount, _deadline, _v, _r, _s);
    _stakeMore(deposit, _depositId, _amount);
  }

```

Evidently one can see that in both instances, `permit()` is queried to verify that the caller has provided enough tokens for the new deposit to stake or to add to an existing deposit, case with this is that, protocol supports WETH and WETh is one of those tokesn that do not support this "permit" functionality, making bot h methods of depositing and staking in protocol faulty cause now an attempt to call `permi()` on this token would silently fail and not revert due to it's fall back implementation, now since it doesn't revert, it means anyone can pass any `_amount` for this asset and stake this amount while not sending/approving any tokens to the `UniStaker` contract.

### Impact

Protocol's internal accounting would get faulty cause now anyone can modify their total staked amount to any value for the WETH token and even the earning power attached of the beneficiary attached to the `depositId`.

### Recommended Mitigation Steps

Since there is no guarantee that all tokens would implement EIP-165, reconsider the protocol's support of WETH and even if it should ever be used as a STAKE_TOKEN

## QA-02 Document and integrate tests on the fact that `UNI` is also a weird token and has this faulty logic in regards to approvals/transfers

### Proof of Concept

Take a look at https://github.com/d-xo/weird-erc20?tab=readme-ov-file#revert-on-large-approvals--transfers.

One can see that `UNI` is weird token and has a faulty logic in regards to approvals/transfers, but no tests has been placed around this to ensure that protocol works as expected. This shouldn't be the case

Now see https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/DelegationSurrogate.sol#L25-L28

```solidity
  constructor(IERC20Delegates _token, address _delegatee) {
    _token.delegate(_delegatee);
    _token.approve(msg.sender, type(uint256).max);
  }
```

Evidently, protocol expects the surrogate to be approved of uint(256).max, but this wouldn't work due to UNI's limitation of approvals.

### Impact

More test cases have not being looked upon, limits protocols sight of contracts

### Recommended Mitigation Steps

Integrate more tests

## QA-03 Streamlining protocol fee management in uniswap V3 could be challenging

### Proof of Concept

Consider the implementation differences between Uniswap V2 and V3 regarding protocol fee management. In V3, the `V3FactoryOwner.setFeeProtocol` method could be adapted to manage a global protocol fee, simplifying the process of enabling or updating fees across pools significantly. This approach would alleviate the governance bottleneck and streamline fee adjustments.

Take a look at https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5298812a129f942555466ebaa6ea9a2af4be0ccc/src/V3FactoryOwner.sol#L136-L150

```solidity
  /// @notice Passthrough method that sets the protocol fee on a v3 pool. Must be called by the
  /// admin.
  /// @param _pool The Uniswap v3 pool on which the protocol fee is being set.
  /// @param _feeProtocol0 The fee protocol 0 param to forward to the pool.
  /// @param _feeProtocol1 The fee protocol 1 parm to forward to the pool.
  /// @dev See docs on IUniswapV3PoolOwnerActions for more information on forwarded params.
  function setFeeProtocol(
    IUniswapV3PoolOwnerActions _pool,
    uint8 _feeProtocol0,
    uint8 _feeProtocol1
  ) external {
    _revertIfNotAdmin();
    _pool.setFeeProtocol(_feeProtocol0, _feeProtocol1);
  }

```

### Impact

In Uniswap V2, the protocol fee is globally managed via the factory, presenting a streamlined approach. However, V3 introduces a more complex mechanism where setting the protocol fee for each pool requires governance approval. This complexity is magnified by the [governance proposal's limitation of actions](https://docs.uniswap.org/contracts/v3/reference/governance/overview#proposal-max-operations), making the activation of a global fee switch logistically challenging. Unlike V2's fixed protocol fee of 1/6th, V3 offers flexibility, allowing fees [to range from 1/10th to 1/4th or vary between pool constituents](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L836-L845). This flexibility, while advantageous, could lead to a community consensus on a universal fee strategy to avoid micromanagement of individual pools.

### Recommended Mitigation Steps

Introduce a global fee protocol feature within the `V3FactoryOwner` contract, allowing for broad fee adjustments without individual pool governance proposals.

Or maybe establish a manual override process, possibly through a trusted entity, to address edge cases or reduce fees after significant increases, ensuring flexibility and adaptability.
