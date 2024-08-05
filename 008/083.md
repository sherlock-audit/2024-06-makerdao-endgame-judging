Interesting Tiger Mole

Medium

# The administrator calling yank() will result in MKR being permanently locked in the LockstakeEngine.

### Summary

In the LockstakeClipper.sol::yank() function, the lack of lsmkr.mint() will result in MKR being permanently locked in the LockstakeEngine.

### Root Cause

https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/main/lockstake/src/LockstakeClipper.sol#L476
```javascript
   // Cancel an auction during End.cage or via other governance action.
    function yank(uint256 id) external auth lock {
        require(sales[id].usr != address(0), "LockstakeClipper/not-running-auction");
        dog.digs(ilk, sales[id].tab);
        uint256 lot = sales[id].lot;
@>>        vat.flux(ilk, address(this), msg.sender, lot);
@>>        engine.onRemove(sales[id].usr, 0, 0);
        _remove(id);
        emit Yank(id);
    }
```
In the yank() function, we can see that only the accounting of the collateral is transferred to msg.sender. However, MKR is neither transferred to msg.sender nor is lsmkr minted to msg.sender. This will result in msg.sender being unable to withdraw MKR from the LockstakeEngine.
Let’s take a look at the LockstakeEngine::free() function.

https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/main/lockstake/src/LockstakeEngine.sol#L340
```javascript
 function freeNoFee(address urn, address to, uint256 wad) external auth urnAuth(urn) {
@>>        _free(urn, wad, 0);
        mkr.transfer(to, wad);
        emit FreeNoFee(urn, to, wad);
    }
```
https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/main/lockstake/src/LockstakeEngine.sol#L360C3-L378C6
```javascript
  function _free(address urn, uint256 wad, uint256 fee_) internal returns (uint256 freed) {
        require(wad <= uint256(type(int256).max), "LockstakeEngine/overflow");
        address urnFarm = urnFarms[urn];
        if (urnFarm != address(0)) {
            LockstakeUrn(urn).withdraw(urnFarm, wad);
        }
@>>        lsmkr.burn(urn, wad);
        vat.frob(ilk, urn, urn, address(0), -int256(wad), 0);
        vat.slip(ilk, urn, -int256(wad));
        address voteDelegate = urnVoteDelegates[urn];
        if (voteDelegate != address(0)) {
            VoteDelegateLike(voteDelegate).free(wad);
        }
        uint256 burn = wad * fee_ / WAD;
        if (burn > 0) {
            mkr.burn(address(this), burn);
        }
        unchecked { freed = wad - burn; } // burn <= wad always
    }
```
It can be seen that due to the lack of lsmkr token, _free() will revert, causing free() to also revert. As a result, msg.sender will never receive the MKR they are entitled to, and the MKR will be permanently locked in the LockstakeEngine. msg.sender only receives the accounting of the collateral, but not the collateral itself, and cannot extract the collateral.

### Internal pre-conditions

To remove the liquidated auction, use the yank function.

### External pre-conditions

1.	There is a position that meets the liquidation conditions.
2.	Initiate the liquidation auction.

### Attack Path

	1.	There is a position that meets the liquidation conditions.
	2.	Initiate the liquidation auction.
	3.	Use the yank function to remove the liquidated auction.

### Impact

The liquidated MKR is permanently locked in the LockstakeEngine, leading to a loss of funds.

### PoC

```javascript

function testClipperYankRevert4FreeMkr() public{
        address urn = _urnSetUp(false, false);
        uint256 id = _forceLiquidation(urn);

        //mkr number for 
         (,, uint256 lot,, address usr,,) = clip.sales(id);
        console2.log("lot is ", lot);

        vm.expectEmit(true, true, true, true);
        emit OnRemove(urn, 0, 0, 0);
        vm.prank(pauseProxy); clip.yank(id);
        assertEq(engine.urnAuctions(urn), 0);

       //pauseProxy transfers the accounting of gem token to urn, then we can test free
        vm.prank(pauseProxy);
        dss.vat.frob(ilk, urn, pauseProxy,address(0), int256(lot), 0);

      //will revert for free Mkr on engin, But it should be sucess.
      engine.free(urn, pauseProxy, lot);

    }
```
add this code in LockstakeEngine.t.sol
then run  `forge test --mt testClipperYankRevert4FreeMkr -vvv`
will get:
```bash
    ├─ [2264] LockstakeEngine::free(0x67c6b529B25c9e34Ab8571e3ed77963A66A2514F, 0xBE8E3e3618f7474F8cB1d074A26afFef007E98FB, 100000000000000000000000 [1e23])
    │   ├─ [713] LockstakeMkr::burn(0x67c6b529B25c9e34Ab8571e3ed77963A66A2514F, 100000000000000000000000 [1e23])
    │   │   └─ ← revert: LockstakeMkr/insufficient-balance
    │   └─ ← revert: LockstakeMkr/insufficient-balance
    └─ ← revert: LockstakeMkr/insufficient-balance

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 46.49s (10.03s CPU time)

```

### Mitigation

```diff
   // Cancel an auction during End.cage or via other governance action.
    function yank(uint256 id) external auth lock {
        require(sales[id].usr != address(0), "LockstakeClipper/not-running-auction");
        dog.digs(ilk, sales[id].tab);
        uint256 lot = sales[id].lot;
-        vat.flux(ilk, address(this), msg.sender, lot);
+        vat.slip(ilk, address(this), -int256(lot));//@audit 为什么事this 少了slice???
+        engine.onTake(sales[id].usr, msg.sender, lot);//如果是link？？
        engine.onRemove(sales[id].usr, 0, 0);
        _remove(id);
        emit Yank(id);
    }
```