Fantastic Spruce Perch

Medium

# Leftover dust debt can cause liquidation auction to occur at significantly lowered price

## Summary
Using `frob` to refund the gem inside `onRemove` can disallow liquidations due to dust check

## Vulnerability Detail
When an auction is removed (on completetion) from the clipper, the leftover amount of the auction if any, is refunded back to the user by calling the `onRemove` method

[gh link](https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/dba30d7a676c20dfed3bda8c52fd6702e2e85bb1/lockstake/src/LockstakeClipper.sol#L415-L420)
```solidity

    function take(
        uint256 id,           // Auction id
        uint256 amt,          // Upper limit on amount of collateral to buy  [wad]
        uint256 max,          // Maximum acceptable price (DAI / collateral) [ray]
        address who,          // Receiver of collateral and external call address
        bytes calldata data   // Data to pass in external call; if length 0, no call is done
    ) external lock isStopped(3) {
        
        .....

        } else if (tab == 0) {
            uint256 tot = sales[id].tot;
            vat.slip(ilk, address(this), -int256(lot));
=>          engine.onRemove(usr, tot - lot, lot);
            _remove(id);
        } else {
```

After burning the associated fees, the remaining amount is credited to the urn by invoking `vat.slip` and `vat.frob`
[gh link](https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/dba30d7a676c20dfed3bda8c52fd6702e2e85bb1/lockstake/src/LockstakeEngine.sol#L438-L454)
```solidity
    function onRemove(address urn, uint256 sold, uint256 left) external auth {
        uint256 burn;
        uint256 refund;
        if (left > 0) {
            burn = _min(sold * fee / (WAD - fee), left);
            mkr.burn(address(this), burn);
            unchecked { refund = left - burn; }
            if (refund > 0) {
                // The following is ensured by the dog and clip but we still prefer to be explicit
                require(refund <= uint256(type(int256).max), "LockstakeEngine/overflow");
                vat.slip(ilk, urn, int256(refund));
=>              vat.frob(ilk, urn, urn, address(0), int256(refund), 0);
                lsmkr.mint(urn, refund);
            }
        }
        urnAuctions[urn]--;
        emit OnRemove(urn, sold, burn, refund);
```

But incase the urn's current debt is less than debt, the `frob` call will revert
[gh link](https://github.com/makerdao/dss/blob/fa4f6630afb0624d04a003e920b0d71a00331d98/src/vat.sol#L173)
```solidity
    function frob(bytes32 i, address u, address v, address w, int dink, int dart) external {

        ....

        require(either(urn.art == 0, tab >= ilk.dust), "Vat/dust");
```

This will cause the complete liquidation to not happen till there is no leftover amount which would occur at a significantly low price from the expected/market price. The condition of `tab >= ilk.dust` can occur due to an increase in the dust value of the ilk by the admin

Example:
initial dust 10k, mat 1.5, user debt 20k, user collateral worth 30k
liquidation of 10k debt happens and dust was increased to 11k
now the 15k worth collateral will only be sold when there will be 0 leftover (since else the onRemove function will revert)
assuming exit fee and liquidation penalty == 15%
in case there was no issue, user would've got ~(15k - 11.5k liquidation penalty included - 2k exit fee == 1.5k) back, but here they will get 0 back
so loss ~= 1.5k/30k ~= 5%   

### POC
Apply the following diff and run `forge test --mt testHash_liquidationFail`
```diff
diff --git a/lockstake/test/LockstakeEngine.t.sol b/lockstake/test/LockstakeEngine.t.sol
index 83fa75d..0bbb3fa 100644
--- a/lockstake/test/LockstakeEngine.t.sol
+++ b/lockstake/test/LockstakeEngine.t.sol
@@ -86,6 +86,7 @@ contract LockstakeEngineTest is DssTest {
 
     function setUp() public {
         vm.createSelectFork(vm.envString("ETH_RPC_URL"));
+        vm.rollFork(20407096);
 
         dss = MCD.loadFromChainlog(LOG);
 
@@ -999,6 +1000,66 @@ contract LockstakeEngineTest is DssTest {
         }
     }
 
+    function testHash_liquidationFail() public {
+        // config original dust == 9k, update dust == 11k, remaining hole == 30k, user debt == 40k
+        // liquidate 30k, 10k remaining, update dust to 11k, and increase the asset price/increase the ink of user ie. preventing further liquidation
+        // now the clipper auction cannot be fulfilled 
+        vm.startPrank(pauseProxy);
+
+        dss.vat.file(cfg.ilk, "dust", 9_000 * 10**45);
+        dss.dog.file(cfg.ilk, "hole", 30_000 * 10**45);
+
+        vm.stopPrank();
+        
+        address urn = engine.open(0);
+
+        // setting mkr price == 1 for ease
+        vm.store(address(pip), bytes32(uint256(1)), bytes32(uint256(1 * 10**18)));
+        // mkr price == 1 and mat = 3, so 40k borrow => 120k mkr
+        deal(address(mkr), address(this), 120_000 * 10**18, true);
+        mkr.approve(address(engine), 120_000 * 10**18);
+        engine.lock(urn, 120_000 * 10**18, 5);
+        engine.draw(urn, address(this), 40_000 * 10**18);
+
+        uint auctionId;
+       {
+        // liquidate 30k
+        vm.store(address(pip), bytes32(uint256(1)), bytes32(uint256(0.98 * 10**18))); // Force liquidation
+        dss.spotter.poke(ilk);
+        assertEq(clip.kicks(), 0);
+        assertEq(engine.urnAuctions(urn), 0);
+        auctionId=dss.dog.bark(ilk, address(urn), address(this));
+        assertEq(clip.kicks(), 1);
+        assertEq(engine.urnAuctions(urn), 1);
+       }
+
+       // bring price back up (or increase the ink) to avoid liquidation of remaining position
+       {
+        vm.store(address(pip), bytes32(uint256(1)), bytes32(uint256(1.2 * 10**18)));
+       }
+
+        // update dust
+        vm.startPrank(pauseProxy);
+
+        dss.vat.file(cfg.ilk, "dust", 11_000 * 10**45);
+
+        vm.stopPrank();
+
+        // attempt to fill the auction completely will fail now till left becomes zero
+        assert(_art(cfg.ilk,urn) == uint(10_000*10**18));
+        
+        address buyer = address(888);
+        vm.prank(pauseProxy); dss.vat.suck(address(0), buyer, 30_000 * 10**45);
+        vm.prank(buyer); dss.vat.hope(address(clip));
+        assertEq(mkr.balanceOf(buyer), 0);
+        // attempt to take the entire auction. will fail due to frob reverting
+
+        vm.prank(buyer); 
+        vm.expectRevert("Vat/dust");
+        clip.take(auctionId, 100_000 * 10**18, type(uint256).max, buyer, "");
+    }
+
+
     function _forceLiquidation(address urn) internal returns (uint256 id) {
         vm.store(address(pip), bytes32(uint256(1)), bytes32(uint256(0.05 * 10**18))); // Force liquidation
         dss.spotter.poke(ilk);

```

## Impact
Even in an efficient liquidation market, a liquidated user's assets will be sold at a significantly lower price causing loss for the user. If there are extremely favourable conditions like control of validators for block ranges/inefficient liquidation market, then a user can self liquidate oneself to retain the collateral while evading fees/at a lowered dai price (for this the attacker will have to be the person who `takes` the auction once it becomes `takeable`). Also the setup Arbitrage Bots will loose gas fees by invoking the take function  

## Code Snippet
https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/dba30d7a676c20dfed3bda8c52fd6702e2e85bb1/lockstake/src/LockstakeEngine.sol#L438-L454

## Tool used
Manual Review

## Recommendation
Use `grab` instead of `frob` to update the gem balance