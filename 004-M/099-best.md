Refined Scarlet Seagull

Medium

# Permanently unliquidatable unhealthy positions can be spoofed to grief keepers and disrupt liquidations

### Summary

A combination of issues in the `LockstakeEngine` (LSE) allows creating permanently unhealthy positions that cannot be liquidated. This can lead to gas griefing of liquidations bots, disruption of the liquidation process, shielding of unhealthy positions from liquidations due to spoofing, and potential bad debt accumulation in the system.

This is achieved by exploiting the the health check's incorrect debt rate usage and using the VoteDelegate's `reserveHatch` mechanism's cooldown to move from one VD to another VD continuously. 

### Root Cause

Two main issues contribute to this vulnerability:

1. The `selectVoteDelegate` function in LockstakeEngine [uses an outdated debt rate](https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/main/lockstake/src/LockstakeEngine.sol#L267-L268) for health checks, allowing the creation of immediately liquidatable positions. This is because the rate should be updated using jug's drip before the check ([as is done in `draw`](https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/main/lockstake/src/LockstakeEngine.sol#L383)). WIthout it, a position can be locked at the outdated health threshold, and immediately turned unhealthy by dripping from the jug. While normally in the vat this is not a problem because a liquidation can be trigerred immediately, in LSE this is not true due to the RH mechanism explained below.

2. The VoteDelegate's `reserveHatch` (RH) mechanism can be used to induce a cooldown period, during which the `free` action required for the liquidation trigger (`onKick`) can be blocked by frontrunning the transaction with a lock from another urn. This lock will [trigger the chief's flashloan protection, and cause the `free` to revert](https://etherscan.io/address/0x0a3f6849f78076aefaDf113F5BED87720274dDC0#code#L463).

Normally, a liquidation auction (`dog.bark`) can be triggered at any point if the price makes a position unhealthy. However, the liquidation process (`dog.bark -> LockstakeClipper.kick -> LSE.onKick`) can be blocked indefinitely by manipulating the VoteDelegate's reserveHatch and deposit mechanisms using this attack.


### Internal pre-conditions

n/a

### External pre-conditions

- The `jug.drip()` function is not called in the same block health checks in `selectVoteDelegate`. Because the attacker can time their attack, this is not a problem.
- The target VoteDelegate contract is not in cooldown already. There is no reason for it to be, since the attacker chooses their VD.

### Attack Path


Scenario 1: Creating permanently unhealthy, unliquidatable positions

1. An attacker creates a position with some amount of debt (by drawing).
2. Some time later, attacker calls `reserveHatch` on a VoteDelegate VD1 they have deposited into. 
3. After 6 blocks (when the hatch is open), they reduce their collateral amount (by calling `free`, which also doesn't `drip`). They then call `jug.drip`, making the position unhealthy.
4. The position now "appears" liquidatable.
5. Liquidator A - doesn't check reserve hatch status, or ignores it being in cooldown:
	- Calls `dog.bark`, but their transaction is frontun by the attacker deposit into the same VD (currently in cooldown), from another ls-urn, of 1 wei of MKR. 
	- Liquidator's transactions revert, wasting gas on "decoy" urns.
6. Liquidator B - checks reserve hatch, sees it cannot be reserved (is in cooldown), and avoids being trigerring the liquidation (to avoid being Liquidator A). No liquidation is trigerred.
7. 5 blocks before the cooldown ends, the attacker calls RH on another VoteDelegate VD2 (or creates a VD and calls its RH).
8. At or near cooldown end (after 240 seconds), the attacker atomically:
   a. Wipes a minimal amount of debt to be just at the outdated health threshold.
   b. Moves the stake to the new VoteDelegate VD2, which is now in cooldown.
   c. Calls drip again and becomes unhealthy again.
8. We're back at step 3.
9. The spoofed liquidatable positions disrupts liquidator bots and shield other liquidatable positions.

Noteably, while `draw` triggers drip, the attacker can manipulate the health of their position without calling drip via using `free` as well (reducing collateral up to threshold).

### Impact


1. Gas griefing for time-sensitive actions: Liquidator bots can lose more than 50 USD (more than 0.5% for 10K total value) due to failed transactions. The PoC shows that a bark transaction revert costs 410K gas. At 200 gwei and 3500 USD ETH price, this translates to a loss of 287 USD per failed liquidation attempt. 

2. Potential bad debt accumulation: Other legitimate liquidations may not be kicked when unhealthy, leading to bad debt in the system.

4. Disruption of liquidation mechanisms: The ability to create "decoy" liquidations can overwhelm liquidation automation and disrupt the overall liquidation process.

### PoC


This PoC shows a gas cost of approximately 410K for a failed bark transaction.

The PoC add a test for `LockstakeEngineBenchmarks` class in `Benchmarks.t.sol` file in this measurement PR https://github.com/makerdao/lockstake/pull/38/files of branch https://github.com/makerdao/lockstake/tree/benchmarks. I've also merged latest `dev` into it to ensure it's using latest contracts. 

The output of the PoC:
```bash
>>> forge test --mc LockstakeEngineBenchmarks --mt testGas -vvv

Logs:
    Bark revert cost: 410183

```

The added tests:
A proof of concept demonstrating the gas cost for a failed bark transaction:

```solidity
function testGasRevertingBark() public {  
    (bool withDelegate, bool withStaking, uint256 numYays) = (true, true, 5);  
    address[] memory yays = new address[](numYays);  
    for (uint256 i; i < numYays; i++) yays[i] = address(uint160(i + 1));  
    vm.prank(voter); VoteDelegate(voteDelegate).vote(yays);  
  
    mkr.approve(voteDelegate, 100_000 * 10**18);  
    VoteDelegate(voteDelegate).lock(100_000 * 10**18);  
  
    address urn = _urnSetUp(withDelegate, withStaking);  
  
    vm.store(address(pip), bytes32(uint256(1)), bytes32(uint256(0.05 * 10**18))); // Force liquidation  
    dss.spotter.poke(ilk);  
    vm.expectRevert();  
    uint256 startGas = gasleft();  
    uint256 id = dss.dog.bark(ilk, address(urn), address(this));  
    uint256 gasUsed = startGas - gasleft();  
    console2.log("  Bark revert cost:", startGas - gasleft());  
}
```



### Mitigation


1. Call `jug.drip()` before performing health checks in the `selectVoteDelegate` function to ensure up-to-date debt rate is used.

2. Modify `onKick` to start the auction even if the `free` action fails. Implement an additional method for keepers to call `free` before `onTake()` is executed at the end of the auction. This can be achieved by adding `afterKick` and `tryAfterKick` functions:

```diff
    function selectVoteDelegate(address urn, address voteDelegate) external urnAuth(urn) {
	    // ..
        if (art > 0 && voteDelegate != address(0)) {
-            (, uint256 rate, uint256 spot,,) = vat.ilks(ilk);
+            (, , uint256 spot,,) = vat.ilks(ilk);
+            uint256 rate = jug.drip(ilk);
            require(ink * spot >= art * rate, "LockstakeEngine/urn-unsafe");
        }
		// ...
    }

    function onKick(address urn, uint256 wad) external auth {
        uint256 inkBeforeKick = ink + wad;
+       toUndelegate[urn] += inkBeforeKick;
-        _selectVoteDelegate(urn, inkBeforeKick, urnVoteDelegates[urn], address(0));
        ..
        urnAuctions[urn]++;
+       tryAfterKick(urn);
    }

+    function afterKick(address urn) external {
+        require(urnAuctions[urn] > 0, "LockstakeEngine/no-active-auction");
+        _selectVoteDelegate(urn, toUndelegate[urn], urnVoteDelegates[urn], address(0));
+        toUndelegate[urn] = 0;
+    }

+    function tryAfterKick(address urn) external {
+        address(this).call(abi.encodeCall(this.afterKick, (urn))); // ignores success
+    }
```

These changes will ensure that liquidations are never blocked from being started and provide sufficient time for MKR to be freed.