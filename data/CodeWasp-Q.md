#  Adapting UniStaker test infrastructure to UNI token

Current testing infrastructure for UniStaker includes fuzz and integration tests which employ mocks for the governance token, in particular [test/mocks/MockERC20Votes.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/test/mocks/MockERC20Votes.sol). The sponsors have confirmed in the Discord contest channel though that exclusively the [currently deployed UNI token](https://etherscan.io/token/0x1f9840a85d5af5bf1d1762f925bdaddc4201f984#code) will be used as the governance token. In light of that information, it should be noted that `MockERC20Votes.sol` is a very crude approximation of the functionality contained in `Uni.sol`. In particular, the latter:

- allows token holders to delegate their voting power directly, via the `delegate()` method;
- employs a non-trivial accounting scheme for delegated votes, indexed according to block numbers;
- is written using Solidity 0.5.16 compiler, and moreover restricts many of its underlying datatypes to `uint96` / `uint32`.

Moreover, the current UniStaker testing infrastructure doesn't try to test for the correct votes accounting at all, although it's a crucial aspect of integrating `UniStaker` with the currently deployed `UNI` token. Taking this into account, we've undertaken the steps to integrate `UNI` token into the `UniStaker` testing, of which activity we report below. In particular, we:

- ported `Uni.sol` from Solidity 0.5.16 to Solidity 0.8.23
- adjusted the tests in `UniStaker.t.sol` such that they pass when used with `Uni.sol` instead of `MockERC20Votes.sol`
- added some assertions to `UniStaker.t.sol` to track for voting power in tests
- wrote a handler around `Uni.sol`, `Uni.handler.sol`, which allows to call for its most important user-facing methods from Foundry fuzz/invariant tester
- performed necessary adaptations to `UniStaker.handler.sol`, to integrate `UNI` and avoid failing tests due to a low-level foundry function
- extended `UniStaker.invariants.t.sol` with an additional invariant, `invariant_Total_stake_plus_direct_delegations_equals_current_votes`, which captures the relation between the voting power delegated directly through users and via UniStaker surrogates
- extended the helper library `AddressSet.sol`, to be able to track external user delegations
- made necessary changes to `foundry.toml` to make the project compile, and run a reasonable amount of fuzz/invariant tests

While these activities have not allowed us to catch any critical vulnerabilities, they did allow us to identify and fix many implicit assumptions in the testing infrastructure that made it incompatible with the real `UNI` token, and not the mock. We also have been able to identify and fix a few false positives, i.e. the tests that were failing due to the deficiencies in the tests themselves. We hope that our efforts will help the UniSwap developers in seamlessly integrating their new staking contracts with the currently deployed ones.

All of the added/modified files are available in [this gist](https://gist.github.com/kuprumion/b7b0e03ea52ff925d0f9a9a4dcd7116f).


## A port of `Uni.sol` from Solidity 0.5.16 to Solidity 0.8.23

This is the simplest of undertaken activities, which amounted in fixing a couple of incompatibilities between the compiler versions, disabling some checks which were not compatible with the current test suite (like minting restrictions), and adding the `DOMAIN_SEPARATOR()` function required by tests. The changes between the [deployed UNI token](https://etherscan.io/token/0x1f9840a85d5af5bf1d1762f925bdaddc4201f984#code) and the adaptation are summarized in the diff below.

```diff
--- test/mocks/Uni.sol.orig     2024-03-04 13:51:22.540178698 +0100
+++ test/mocks/Uni.sol  2024-03-04 14:22:43.058757812 +0100
@@ -1,4 +1,8 @@
-pragma solidity ^0.5.16;
+// Adaptation of the UNI code from https://etherscan.io/token/0x1f9840a85d5af5bf1d1762f925bdaddc4201f984#code
+// To make the tests pass. For the original version see "Uni.sol.orig"
+pragma solidity 0.8.23;
 pragma experimental ABIEncoderV2;
 
+import {IERC20Delegates} from "src/interfaces/IERC20Delegates.sol";
+
 // From https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/math/Math.sol
@@ -188,3 +192,3 @@
 
-contract Uni {
+contract Uni  is IERC20Delegates {
     /// @notice EIP-20 token name for this token
@@ -293,4 +297,4 @@
     function mint(address dst, uint rawAmount) external {
-        require(msg.sender == minter, "Uni::mint: only the minter can mint");
-        require(block.timestamp >= mintingAllowedAfter, "Uni::mint: minting not allowed yet");
+        // require(msg.sender == minter, "Uni::mint: only the minter can mint");
+        // require(block.timestamp >= mintingAllowedAfter, "Uni::mint: minting not allowed yet");
         require(dst != address(0), "Uni::mint: cannot transfer to the zero address");
@@ -302,3 +306,3 @@
         uint96 amount = safe96(rawAmount, "Uni::mint: amount exceeds 96 bits");
-        require(amount <= SafeMath.div(SafeMath.mul(totalSupply, mintCap), 100), "Uni::mint: exceeded mint cap");
+        // require(amount <= SafeMath.div(SafeMath.mul(totalSupply, mintCap), 100), "Uni::mint: exceeded mint cap");
         totalSupply = safe96(SafeMath.add(totalSupply, amount), "Uni::mint: totalSupply exceeds 96 bits");
@@ -333,4 +337,4 @@
         uint96 amount;
-        if (rawAmount == uint(-1)) {
-            amount = uint96(-1);
+        if (rawAmount == type(uint).max) {
+            amount = type(uint96).max;
         } else {
@@ -345,2 +349,6 @@
 
+    function DOMAIN_SEPARATOR() external view virtual returns (bytes32) {
+        return keccak256(abi.encode(DOMAIN_TYPEHASH, keccak256(bytes(name)), getChainId(), address(this)));
+    }
+
     /**
@@ -357,4 +365,4 @@
         uint96 amount;
-        if (rawAmount == uint(-1)) {
-            amount = uint96(-1);
+        if (rawAmount == type(uint).max) {
+            amount = type(uint96).max;
         } else {
@@ -369,3 +377,3 @@
         require(signatory == owner, "Uni::permit: unauthorized");
-        require(now <= deadline, "Uni::permit: signature expired");
+        require(block.timestamp <= deadline, "Uni::permit: signature expired");
 
@@ -409,3 +417,3 @@
 
-        if (spender != src && spenderAllowance != uint96(-1)) {
+        if (spender != src && spenderAllowance != type(uint96).max) {
             uint96 newAllowance = sub96(spenderAllowance, amount, "Uni::transferFrom: transfer amount exceeds spender allowance");
@@ -444,3 +452,3 @@
         require(nonce == nonces[signatory]++, "Uni::delegateBySig: invalid nonce");
-        require(now <= expiry, "Uni::delegateBySig: signature expired");
+        require(block.timestamp <= expiry, "Uni::delegateBySig: signature expired");
         return _delegate(signatory, delegatee);
@@ -572,3 +580,3 @@
 
-    function getChainId() internal pure returns (uint) {
+    function getChainId() internal view returns (uint) {
         uint256 chainId;
```

## Adjustment of the tests in `UniStaker.t.sol` to use `Uni.sol` instead of `MockERC20Votes.sol`

We don't list here the whole diff, only the most important parts of it; also omitting duplicate changes in multiple places.

**Preamble and set up: replace `ERC20VotesMock` with `Uni`**

```diff 
diff --git a/test/UniStaker.t.sol b/test/UniStaker.t.sol
index 89124f8..22e0534 100644
--- a/test/UniStaker.t.sol
+++ b/test/UniStaker.t.sol
@@ -9,2 +9,3 @@ import {ERC20Fake} from "test/fakes/ERC20Fake.sol";
 import {PercentAssertions} from "test/helpers/PercentAssertions.sol";
+import {Uni} from "test/mocks/Uni.sol";
 
@@ -12,3 +13,3 @@ contract UniStakerTest is Test, PercentAssertions {
   ERC20Fake rewardToken;
-  ERC20VotesMock govToken;
+  Uni govToken;
   address admin;
@@ -38,4 +39,6 @@ contract UniStakerTest is Test, PercentAssertions {
 
-    govToken = new ERC20VotesMock();
+    admin = makeAddr("admin");
+    govToken = new Uni(admin, admin, 2000);
     vm.label(address(govToken), "Governance Token");
+    _jumpAhead(1234);
 
@@ -44,4 +47,2 @@ contract UniStakerTest is Test, PercentAssertions {
 
-    admin = makeAddr("admin");
-
     uniStaker = new UniStakerHarness(rewardToken, govToken, admin);
@@ -61,3 +62,3 @@ contract UniStakerTest is Test, PercentAssertions {
   function _boundMintAmount(uint256 _amount) internal pure returns (uint256) {
-    return bound(_amount, 0, 100_000_000_000e18);
+    return bound(_amount, 0, 100_000_000_000e12); // reduced for tests to pass with UNI
   }
@@ -66,2 +67,4 @@ contract UniStakerTest is Test, PercentAssertions {
     vm.assume(_to != address(0));
+    vm.assume(_to != admin); // needed to avoid using admin's address in tests
+    vm.prank(admin);
     govToken.mint(_to, _amount);
```

**Reduce the maximum constants used to be compatible with `uint96` used in `Uni`**

```diff
@@ -74,3 +77,3 @@ contract UniStakerTest is Test, PercentAssertions {
   {
-    _boundedStakeAmount = bound(_stakeAmount, 0.1e18, 25_000_000e18);
+    _boundedStakeAmount = bound(_stakeAmount, 0.1e18, 25_000_000e12);  // reduced for tests to pass with UNI
   }
@@ -194,3 +197,3 @@ contract Stake is UniStakerTest {
   ) public {
-    _amount = bound(_amount, 1, type(uint224).max);
+    _amount = bound(_amount, 1, type(uint88).max);
     _mintGovToken(_depositor, _amount);
@@ -721,3 +733,3 @@ contract PermitAndStake is UniStakerTest {
     uint256 _deadline,
-    uint256 _currentNonce
+    uint248 _currentNonce
   ) public {
@@ -2371,3 +2384,3 @@ contract Withdraw is UniStakerTest {
     (_amount, _depositId) = _boundMintAndStake(_depositor, _amount, _delegatee);
-    _amountOver = bound(_amountOver, 1, type(uint128).max);
+    _amountOver = bound(_amountOver, 1, type(uint88).max);

```

**Miscellaneous changes**

```diff
@@ -793,3 +805,3 @@ contract PermitAndStake is UniStakerTest {
     vm.expectRevert(
-      abi.encodeWithSelector(ERC20Permit.ERC2612InvalidSigner.selector, _depositor, _notDepositor)
+      "Uni::permit: unauthorized"
     );
@@ -4670,5 +4682,5 @@ contract _FetchOrDeploySurrogate is UniStakerRewardsTest {
 
-    assertEq(logs[1].topics[0], keccak256("SurrogateDeployed(address,address)"));
-    assertEq(logs[1].topics[1], bytes32(uint256(uint160(_delegatee))));
-    assertEq(logs[1].topics[2], bytes32(uint256(uint160(address(_surrogate)))));
+    assertEq(logs[2].topics[0], keccak256("SurrogateDeployed(address,address)"));
+    assertEq(logs[2].topics[1], bytes32(uint256(uint160(_delegatee))));
+    assertEq(logs[2].topics[2], bytes32(uint256(uint160(address(_surrogate)))));
   }
```

## Additional assertions to track voting power changes in `Uni`

As already explained above, voting power is a very important aspect of `UNI` token, which, on the one hand, is influenced by the introduction of `UniStaker` (via surrogate delegations), and on the other hand voting power changes are not tracked at all in the current test suite. We have added corresponding assertions to a few of the current tests; the rest of the test suite needs to be examined, and assertions added as well; we leave this to UniSwap developers.

An example of one of the modified tests is below:

```diff
@@ -189,15 +191,16 @@ contract Constructor is UniStakerTest {
 contract Stake is UniStakerTest {
   function testFuzz_DeploysAndTransfersTokensToANewSurrogateWhenAnAccountStakes(
     address _depositor,
     uint256 _amount,
     address _delegatee
   ) public {
-    _amount = bound(_amount, 1, type(uint224).max);
+    _amount = bound(_amount, 1, type(uint88).max);
     _mintGovToken(_depositor, _amount);
     _stake(_depositor, _amount, _delegatee);
 
     DelegationSurrogate _surrogate = uniStaker.surrogates(_delegatee);
 
     assertEq(govToken.balanceOf(address(_surrogate)), _amount);
     assertEq(govToken.delegates(address(_surrogate)), _delegatee);
     assertEq(govToken.balanceOf(_depositor), 0);
+    assertEq(govToken.getCurrentVotes(_delegatee), _amount);
   }
 ```

 ##  Add `Uni.handler.sol`, the wrapper around `Uni`, allowing to call its functions from fuzz/invariant tests

Similar to the already present [test/helpers/UniStaker.handler.sol](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/test/helpers/UniStaker.handler.sol), we have implemented the lightweight `test/helpers/Uni.handler.sol`, which allows to call most crucial for testing user-facing functions of `UNI`.

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity ^0.8.13;

import {CommonBase} from "forge-std/Base.sol";
import {StdCheats} from "forge-std/StdCheats.sol";
import {StdUtils} from "forge-std/StdUtils.sol";
import {AddressSet, LibAddressSet} from "../helpers/AddressSet.sol";
import {Uni} from "test/mocks/Uni.sol";

contract UniHandler is CommonBase, StdCheats, StdUtils {
  using LibAddressSet for AddressSet;

  Uni public uni;

  // delegator -> delegatee
  mapping(address => address) private _delegatee;
  
  // delegatee -> delegators
  mapping(address => AddressSet) private _delegators;

  constructor(Uni _uni) {
    uni= _uni;
  }

  function approve(address spender, uint _amount) external returns (bool)
  {
    _amount = bound(_amount, 0, type(uint96).max);
    vm.startPrank(msg.sender);
    uni.approve(spender, _amount);
    vm.stopPrank();
    return true;
  }

  // Track delegations performed by the users directly via the UNI token
  function transfer(address dst, uint _amount) external returns (bool)
  {
    // bound to the max available amount
    vm.startPrank(msg.sender);
    uint256 balance =  uni.balanceOf(msg.sender);
    _amount = bound(_amount, 0, balance);
    uni.transfer(dst, _amount);
    vm.stopPrank();
    return true;
  }

  // Track delegations performed by users directly via the UNI token
  function delegate(address delegatee)  public  
  {
    address prev_delegatee = _delegatee[msg.sender];
    _delegators[prev_delegatee].remove(msg.sender);
    _delegators[delegatee].add(msg.sender);
    _delegatee[msg.sender] = delegatee;

    vm.startPrank(msg.sender);
    uni.delegate(delegatee);
    vm.stopPrank();
  }

  // Advance the specified number of blocks. 
  // Needed to trigger UNI's block-numbers-based votes accounting
  function roll(uint16 advance) public  
  {
    vm.roll(block.number + advance);
  }

  function addDelegator(uint256 acc, address delegator) external view returns (uint256) {
    return acc + uni.balanceOf(delegator);
  }

  function sumDelegatorVotes(address delegatee)
    public view
    returns (uint256)
  {
    return _delegators[delegatee].reduce(0, this.addDelegator);
  }  
}
```

## Necessary adaptations to `UniStaker.handler.sol`

We had to perform necessary adaptations to `UniStaker.handler.sol`, to integrate `UNI` and avoid failing tests due to the usage of a low-level foundry function; the changes are outlined below.

```diff
diff --git a/test/helpers/UniStaker.handler.sol b/test/helpers/UniStaker.handler.sol
index f8fe335..9622571 100644
--- a/test/helpers/UniStaker.handler.sol
+++ b/test/helpers/UniStaker.handler.sol
@@ -10,2 +10,3 @@ import {UniStaker} from "src/UniStaker.sol";
 import {IERC20} from "openzeppelin/token/ERC20/IERC20.sol";
+import {Uni} from "test/mocks/Uni.sol";
-  IERC20 public stakeToken;
+  Uni public stakeToken;
   IERC20 public rewardToken;
@@ -50,3 +51,3 @@ contract UniStakerHandler is CommonBase, StdCheats, StdUtils {
     uniStaker = _uniStaker;
-    stakeToken = IERC20(address(_uniStaker.STAKE_TOKEN()));
+    stakeToken = Uni(address(_uniStaker.STAKE_TOKEN()));
     rewardToken = IERC20(address(_uniStaker.REWARD_TOKEN()));
@@ -57,3 +58,5 @@ contract UniStakerHandler is CommonBase, StdCheats, StdUtils {
     vm.assume(_to != address(0));
-    deal(address(stakeToken), _to, _amount, true);
+    vm.prank(admin);
+    stakeToken.mint(_to, _amount);
+    vm.stopPrank();
   }
@@ -98,2 +101,4 @@ contract UniStakerHandler is CommonBase, StdCheats, StdUtils {
   {
+    vm.assume(_delegatee != address(0));
+    vm.assume(_beneficiary != address(0));
     _createDepositor();
@@ -185,4 +190,4 @@ contract UniStakerHandler is CommonBase, StdCheats, StdUtils {
 
-  function reduceDepositors(uint256 acc, function(uint256,address) external returns (uint256) func)
-    public
+  function reduceDepositors(uint256 acc, function(uint256,address) external view returns (uint256) func)
+    public view
     returns (uint256)
@@ -194,4 +199,4 @@ contract UniStakerHandler is CommonBase, StdCheats, StdUtils {
     uint256 acc,
-    function(uint256,address) external returns (uint256) func
-  ) public returns (uint256) {
+    function(uint256,address) external view returns (uint256) func
+  ) public view returns (uint256) {
     return _beneficiaries.reduce(acc, func);
@@ -199,4 +204,4 @@ contract UniStakerHandler is CommonBase, StdCheats, StdUtils {
 
-  function reduceDelegates(uint256 acc, function(uint256,address) external returns (uint256) func)
-    public
+  function reduceDelegates(uint256 acc, function(uint256,address) external view returns (uint256) func)
+    public view
     returns (uint256)
```

In particular, usage of the low-level Foundry's `deal` function, which modifies in place the storage of an ERC20 contract, is incompatible with `UNI`'s vote accounting mechanism, and leads to underflows in vote computations with the error thrown `Uni::_moveVotes: vote amount underflows`.

## Extensions to `UniStaker.invariants.t.sol` to track an additional invariant, `invariant_Total_stake_plus_direct_delegations_equals_current_votes`

We have extended `UniStaker.invariants.t.sol` with an additional invariant that asserts that on all changes, either via `UniStaker` or via direct user delegations via `UNI`, the total stake via `UniStaker` summed up with direct delegations, gives the total voting power for all delegates. The changes are outlined below.

```diff
diff --git a/test/UniStaker.invariants.t.sol b/test/UniStaker.invariants.t.sol
index 4c80ce1..5148548 100644
--- a/test/UniStaker.invariants.t.sol
+++ b/test/UniStaker.invariants.t.sol
@@ -8,3 +8,4 @@ import {UniStaker} from "src/UniStaker.sol";
 import {UniStakerHandler} from "test/helpers/UniStaker.handler.sol";
-import {ERC20VotesMock} from "test/mocks/MockERC20Votes.sol";
+import {Uni} from "test/mocks/Uni.sol";
+import {UniHandler} from "test/helpers/Uni.handler.sol";
 import {ERC20Fake} from "test/fakes/ERC20Fake.sol";
@@ -15,4 +16,23 @@ contract UniStakerInvariants is Test {
   ERC20Fake rewardToken;
-  ERC20VotesMock govToken;
+  Uni govToken;
+  UniHandler uniHandler;
   address rewardsNotifier;
+  address admin;
+  address alice;
+  address bob;
+  address carol;
+  address dave;
+  address eve;
+  address frank;
+
+  function _jumpAhead(uint256 _seconds) public {
+    vm.warp(block.timestamp + _seconds);
+  }
+
+  function _mintGovToken(address _to, uint256 _amount) internal {
+    vm.assume(_to != address(0));
+    vm.prank(admin);
+    govToken.mint(_to, _amount);
+    vm.stopPrank();
+  }
 
@@ -22,4 +42,22 @@ contract UniStakerInvariants is Test {
 
-    govToken = new ERC20VotesMock();
-    vm.label(address(govToken), "Governance Token");
+    _jumpAhead(1234);
+    admin = makeAddr("admin");
+    alice = makeAddr("alice");
+    bob = makeAddr("bob");
+    carol = makeAddr("carol");
+    dave = makeAddr("dave");
+    eve = makeAddr("eve");
+    frank = makeAddr("frank");
+
+    govToken = new Uni(admin, admin, 2000);
+    vm.label(address(govToken), "Uni Token");
+    _jumpAhead(1234);
+
+    _mintGovToken(admin, 1e27);
+    _mintGovToken(alice, 1e27);
+    _mintGovToken(bob, 1e27);
+    _mintGovToken(carol, 1e27);
+    _mintGovToken(dave, 1e27);
+    _mintGovToken(eve, 1e27);
+    _mintGovToken(frank, 1e27);
 
@@ -42,2 +80,19 @@ contract UniStakerInvariants is Test {
     targetContract(address(handler));
+
+    uniHandler = new UniHandler(govToken);
+    bytes4[] memory uniSelectors = new bytes4[](4);
+    uniSelectors[0] = UniHandler.transfer.selector;
+    uniSelectors[1] = UniHandler.approve.selector;
+    uniSelectors[2] = UniHandler.delegate.selector;
+    uniSelectors[3] = UniHandler.roll.selector;
+
+    targetSelector(FuzzSelector({addr: address(uniHandler), selectors: uniSelectors}));
+
+    targetContract(address(uniHandler));
+    targetSender(alice);
+    targetSender(bob);
+    targetSender(carol);
+    targetSender(dave);
+    targetSender(eve);
+    targetSender(frank);
   }
@@ -84,2 +139,23 @@ contract UniStakerInvariants is Test {
 
+  function invariant_Total_stake_plus_direct_delegations_equals_current_votes() public {
+    assertEq(uniStaker.totalStaked() + handler.reduceDelegates(0, this.accumulateDirectDelegateVotes), 
+      handler.reduceDelegates(0, this.accumulateCurrentDelegateVotes));
+  }
+
+  function accumulateDirectDelegateVotes(uint256 votes, address delegate)
+    external
+    view
+    returns (uint256)
+  {
+    return votes + uniHandler.sumDelegatorVotes(delegate);
+  }
+
+  function accumulateCurrentDelegateVotes(uint256 votes, address delegate)
+    external
+    view
+    returns (uint256)
+  {
+    return votes + govToken.getCurrentVotes(delegate);
+  }
+
   // Used to see distribution of non-reverting calls
```

## Necessary changes to `AddressSet.sol`

In order to be able to track external user delegations, we had to adapt slightly the helper library `AddressSet.sol`:

```diff
diff --git a/test/helpers/AddressSet.sol b/test/helpers/AddressSet.sol
index 83327a7..323ed2c 100644
--- a/test/helpers/AddressSet.sol
+++ b/test/helpers/AddressSet.sol
@@ -17,6 +17,20 @@ library LibAddressSet {
     }
   }
 
+  function remove(AddressSet storage s, address addr) internal {
+    if (s.saved[addr]) {
+      uint256 len = s.addrs.length;
+      for(uint256 i = 0; i < len; ++i) {
+        if(s.addrs[i] == addr) {
+          s.addrs[i] = s.addrs[len-1];
+          break;
+        }
+      }
+      s.addrs.pop();
+      s.saved[addr] = false;
+    }
+  }
+
   function contains(AddressSet storage s, address addr) internal view returns (bool) {
     return s.saved[addr];
   }
@@ -39,8 +53,8 @@ library LibAddressSet {
   function reduce(
     AddressSet storage s,
     uint256 acc,
-    function(uint256,address) external returns (uint256) func
-  ) internal returns (uint256) {
+    function(uint256,address) external view returns (uint256) func
+  ) internal view returns (uint256) {
     for (uint256 i; i < s.addrs.length; ++i) {
       acc = func(acc, s.addrs[i]);
     }
```


## Necessary changes to `foundry.toml`

We had to introduce a few changes to `foundry.toml`. On the one hand, a couple of dependencies were missing, so we've introduced them for the project to compile. On the other hand, the fuzzing/invariant test settings have been in our opinion very low, so we increased the number or the depth of the runs in order to increase the coverage. 

```diff
diff --git a/foundry.toml b/foundry.toml
index a3031f2..64d0f63 100644
--- a/foundry.toml
+++ b/foundry.toml
@@ -2,17 +2,23 @@
   evm_version = "paris"
   optimizer = true
   optimizer_runs = 10_000_000
-  remappings = ["openzeppelin/=lib/openzeppelin-contracts/contracts"]
+  remappings = [
+    "openzeppelin/=lib/openzeppelin-contracts/contracts",
+    "uniswap-periphery/=lib/v3-periphery/contracts",
+    "@uniswap/v3-core=lib/v3-core",
+  ]
   solc_version = "0.8.23"
   verbosity = 3
+  fuzz = { runs = 500 }
+  invariant = { runs = 100, depth = 100 }
 
 [profile.ci]
   fuzz = { runs = 5000 }
-  invariant = { runs = 1000 }
+  invariant = { runs = 1000, depth = 100 }
 
 [profile.lite]
   fuzz = { runs = 50 }
-  invariant = { runs = 10 }
+  invariant = { runs = 10, depth = 100 }
   # Speed up compilation and tests during development.
   optimizer = false
 
```

Increasing the fuzz/invariant bounds allowed us in particular to observe the following failing test.

```sh
[FAIL. Reason: assertion failed; counterexample: calldata=0xc1e611e700000000000000000000000000000000000000000000000000000000000029fa00000000000000000000000000000000000000000000000000000000000004d3000000000000000000000000aa10a84ce7d9ae517a52c6d5ca153b369af99ecf0000000000000000000000000000000000000000000000000000000000002d6900000000000000000000000000000000000000000000000000000000000000970000000000000000000000000000000000000000000000000000000000000631 args=[0x00000000000000000000000000000000000029fa, 1235, 0xaA10a84CE7d9AE517a52c6d5cA153b369Af99ecF, 11625 [1.162e4], 0x0000000000000000000000000000000000000097, 0x0000000000000000000000000000000000000631]] testFuzz_DeploysAndTransfersTokenToTwoSurrogatesWhenAccountsStakesToDifferentDelegatees(address,uint256,address,uint256,address,address) (runs: 370, μ: 803661, ~: 816488)
Logs:
  Bound Result 1235
  Bound Result 11625
  Error: a == b not satisfied [uint]
        Left: 1000000000000000000000000000
       Right: 0
```
The reason for the test failure was that due to increased number of alternatives tried: Foundry's fuzz testing engine picked admin's address to mint to, and thus [this assertion](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/test/UniStaker.t.sol#L414) failed as a result. We have repaired the failing test by disallowing to mint governance tokens to admin's address.

## Applying the changes to the UniStaker testing infrastructure, and running the tests

To correctly set up the environment, and apply the modifications, do the following:

- `git clone https://github.com/code-423n4/2024-02-uniswap-foundation.git`
- `cd 2024-02-uniswap-foundation`
- `forge install uniswap/v3-core`
- `forge install uniswap/v3-periphery`
- Download [this gist](https://gist.github.com/kuprumion/b7b0e03ea52ff925d0f9a9a4dcd7116f), and unpack it e.g. into `../uni`;
- Place the files as follows inside the repo:
  - `cp ../uni/foundry.toml ./`
  - `cp ../uni/Uni.sol ./test/mocks/`
  - `cp ../uni/Uni.handler.sol ./test/helpers/`
  - `cp ../uni/UniStaker.handler.sol ./test/helpers/`
  - `cp ../uni/AddressSet.sol ./test/helpers/`
  - `cp ../uni/UniStaker.t.sol ./test/`
  - `cp ../uni/UniStaker.invariants.t.sol ./test/`

Then execute the tests (excluding the integration tests) via this command:

```sh
forge test --nmp '*integration*'
```

To execute and examine the working of the newly introduced invariant, we recommend to focus on it, and execute it in verbose mode:

```sh
forge test -vvvv --nmp '*integration*' --match-test invariant_Total_stake_plus_direct_delegations_equals_current_votes
```