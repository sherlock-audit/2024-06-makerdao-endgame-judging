Unique Pistachio Chipmunk

Medium

# Rewards accrued during the period of _totalSupply = 0 will get lost forever.

### Summary

During the period of ```_totalSupply = 0```, rewards will continue to accrue but are not distributed to any staker, as a result, these rewards will get lost forever. Loss of funds for the protocol. 

Given the 100% loss nature during this period, but the relative low probability of its occurence, I give a balanced ranking of ```medium```.

### Root Cause

During the period of ```_totalSupply = 0```, rewards will continue to accrue but are not distributed to any staker.

- ```rewardPerTokenStored``` remains the same. No distribution to stakers. 
-  ```lastUpdateTime``` will be updated, means accrue happens. The remaining reward period  ```periodFinish - lastUpdateTime``` will be shortened. 

[https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/main/endgame-toolkit/src/synthetix/StakingRewards.sol#L191-L199](https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/main/endgame-toolkit/src/synthetix/StakingRewards.sol#L191-L199)

### Internal pre-conditions

_supply = 0. 

### External pre-conditions

Some  time passes while _supply = 0 when there is an ongoing rewardRate != 0. 

### Attack Path

Below our POC shows how the rewards might get lost: 

1. Bob stakes 50000000 ether of staking tokens;
2. After 10512000 secs, 5000000000000 reward tokens are sent to the ```StakingRewards``` contract, initiating a reward stream of with a reward rate of 8267195.
3. After 1 day, Bob withdraws 50000000 ether of staking tokens and gets 714250000000 reward tokens.
4. After 356 days (an exaggeration but this is the period that rewards get lost), we call ```k.rewards.notifyRewardAmount(0)``` to start a new reward stream only using the remaining rewards in the ```StakingRewards```, if any. 
5. Although there is  4285750000000 reward tokens in the ```StakingRewards```, they are stuck and not available for future dispense, as confirmed by ```rewardRate = 0```. The following experiment further confirms future staking cannot get reward.
6. Bob stakes another 50000000 ether of staking tokens and wait for one day to withdraw them and get reward. This time, he gets 0 reward since ```rewardRate = 0```.  This further confirms that the remaining rewards (4285750000000) are all lost in the contract during that 356 days. They are lost forever and are not available for future dispense. 


### Impact

Rewards accrued during the period of _totalSupply = 0 will get lost forever. This is a loss for the protocol.

### PoC


run ```forge test --match-test testLostReward1 -vv```.

```javascript
// SPDX-FileCopyrightText: © 2023 Dai Foundation <www.daifoundation.org>
// SPDX-License-Identifier: AGPL-3.0-or-later
//
// This program is free software: you can redistribute it and/or modify
// it under the terms of the GNU Affero General Public License as published by
// the Free Software Foundation, either version 3 of the License, or
// (at your option) any later version.
//
// This program is distributed in the hope that it will be useful,
// but WITHOUT ANY WARRANTY; without even the implied warranty of
// MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
// GNU Affero General Public License for more details.
//
// You should have received a copy of the GNU Affero General Public License
// along with this program.  If not, see <https://www.gnu.org/licenses/>.
pragma solidity ^0.8.16;

import {DssTest, stdStorage, StdStorage} from "dss-test/DssTest.sol";
import {IERC20} from "openzeppelin-contracts/token/ERC20/IERC20.sol";
import {DssVestWithGemLike} from "./interfaces/DssVestWithGemLike.sol";
import {StakingRewards} from "./synthetix/StakingRewards.sol";
import {SDAO} from "./SDAO.sol";
import {VestedRewardsDistribution} from "./VestedRewardsDistribution.sol";
import "forge-std/Test.sol";
import "forge-std/console2.sol";
import {DssVestMintable} from "lib/dss-vest/src/DssVest.sol";
import {ERC20Mock} from "openzeppelin-contracts/mocks/ERC20Mock.sol";

contract VestedRewardsDistributionTest is DssTest {
    using stdStorage for StdStorage;

    struct VestParams {
        address usr;
        uint256 tot;
        uint256 bgn;
        uint256 tau;
        uint256 eta;
    }

    struct DistributionParams {
        VestedRewardsDistribution dist;
        DssVestWithGemLike vest;
        StakingRewards rewards;
        IERC20Mintable rewardsToken;
        uint256 vestId;
        VestParams vestParams;
    }

    DistributionParams k;
    IERC20Mintable rewardsToken;
    ERC20Mock stakingToken;

    uint256 constant DEFAULT_DURATION = 365 days;
    uint256 constant DEFAULT_CLIFF = 0;
    uint256 constant DEFAULT_STARTING_RATE = uint256(200_000 * WAD) / DEFAULT_DURATION;
    uint256 constant DEFAULT_FINAL_RATE = uint256(2_000_000 * WAD) / DEFAULT_DURATION;
    uint256 constant DEFAULT_TOTAL_REWARDS = ((DEFAULT_STARTING_RATE + DEFAULT_FINAL_RATE) * DEFAULT_DURATION) / 2;



    function setUp() public {
        // DssVest checks if params are not too far away in the future or in the past relative to `block.timestamp`.
        // It has a 20 years interval check hardcoded, so we need to be at a time that is at least 20 years ahead of
        // the Unix epoch.  We are setting the current date of the chain to 2000-01-01 to comply with that requirement.
        vm.warp(946692000);

        rewardsToken = IERC20Mintable(address(new SDAO("K Token", "K")));
        console2.log("inital rewardsToken: ", address(rewardsToken));
        stakingToken = new ERC20Mock("NST", "NST", address(111), 1);
        console2.log("initial stakingToken: ", address(stakingToken));

        k = _setUpDistributionParams(
            DistributionParams({
                dist: VestedRewardsDistribution(address(0)),
                vest: DssVestWithGemLike(address(0)),
                rewards: StakingRewards(address(0)),
                rewardsToken: rewardsToken,
                vestId: 0,
                vestParams: _makeVestParams()
            })
        );
    }

    function testLostReward1() public{
        address Bob = makeAddr("Bob");
        deal(address(stakingToken), Bob, 100000000 ether); // 100M ether

        console2.log("\n Bob stakes 50000000 ether...........................");
        vm.startPrank(Bob);                                            // 1. stake 50 000 000 ether 
        stakingToken.approve(address(k.rewards), 50000000 ether);
        k.rewards.stake(50000000 ether);
        vm.stopPrank();

        // printk();
        
        // 1st distribution
        skip(k.vestParams.tau / 3);
        console2.log("after: ", k.vestParams.tau / 3);

        console2.log(" before distribute 1........................");
        printBalances(address(k.rewards), "StakingRewards");

        k.dist.distribute();
        
        console2.log(" after distribute 1........................");
        printBalances(address(k.rewards), "StakingRewards");
        console2.log("rewardRate: ", k.rewards.rewardRate());

        

        skip(1 days);
        console2.log("after 1 day...\n");

        console2.log("\n  Before Bob withdraw 50000000 ether and get reward...........................");
        printBalances(address(k.rewards), "StakingRewards");
        printBalances(Bob, "Bob");
        
        vm.startPrank(Bob);                                            // 2. stake another 50 000 000 ether 
        k.rewards.withdraw(50000000 ether);
        k.rewards.getReward();
        vm.stopPrank();

        console2.log("\n  Before Bob withdraw 50000000 ether and get reward...........................");
        printBalances(address(k.rewards), "StakingRewards");
        printBalances(Bob, "Bob");

        uint256 initialRewardBalance = k.rewardsToken.balanceOf(Bob);

        skip(365 days);
        console2.log("after 356 days...\n");
        // by now all rewards have been accrued but not distributed to stakers, they are lost

         // simulate to distribute zero so that we can use the remaining rewards in the stakingRewards for future accrue
        vm.startPrank(address(k.dist));
        k.rewards.notifyRewardAmount(0);
        vm.stopPrank();
        assertEq(k.rewardsToken.balanceOf(address(k.rewards)), 4285750000000);
        assertEq(k.rewards.rewardRate(), 0); 



        console2.log("\n Bob stakes 50000000 ether...........................");
        vm.startPrank(Bob);                                            // 1. stake 50 000 000 ether 
        stakingToken.approve(address(k.rewards), 50000000 ether);
        k.rewards.stake(50000000 ether);
        vm.stopPrank();
  
        skip(1 days);  // after one day, Bob still receives no reward
        console2.log("after 1 day...\n");

        console2.log("\n  Before Bob withdraw 50000000 ether and get reward...........................");
        printBalances(address(k.rewards), "StakingRewards");
        printBalances(Bob, "Bob");


        vm.startPrank(Bob);                                            // 2. stake another 50 000 000 ether 
        k.rewards.withdraw(50000000 ether);
        k.rewards.getReward();
        vm.stopPrank();

        console2.log("\n  After Bob withdraw 50000000 ether and get reward...........................");
        printBalances(address(k.rewards), "StakingRewards");
        printBalances(Bob, "Bob");

        uint256 finalRewardBalance = k.rewardsToken.balanceOf(Bob);
        assertEq(initialRewardBalance, finalRewardBalance);
    }

    
    function printk() public{
        console2.log("\n k values: $$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$");
        console2.log("dist: ", address(k.dist));
        console2.log("vest: ", address(k.vest));
        console2.log("rewards (staking contract): ", address(k.rewards));
        console2.log("stakingToken: ", address(stakingToken));
        console2.log("rewardsToken:", address(k.rewardsToken));
        console2.log("vesatId: ", k.vestId);
        console2.log("vestParams.usr:", k.vestParams.usr);
        console2.log("vestParams.tot:", k.vestParams.tot);
        console2.log("vestParams.bgn:", k.vestParams.bgn);
        console2.log("vestParams.tau:", k.vestParams.tau);
        console2.log("vestParams.eta:", k.vestParams.eta);
        console2.log("$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$\n ");
    }

    function printVestAward(uint256 i) public {
       DssVestWithGemLike.Award  memory a = k.vest.awards(i);

       console2.log("The vest award information for ============", i);
       console2.log("usr: ", a.usr);
       console2.log("bgn: ", a.bgn);
       console2.log("clf: ", a.clf); // The cliff date  
       console2.log("fin: ", a.fin);
       console2.log("mgr: ", a.mgr);
       console2.log("res: ", a.res);
       console2.log("tot: ", a.tot);
       console2.log("rxd: ", a.rxd);
       console2.log("==========================================");
    }

    function printBalances(address a, string memory name) public
    {
        console2.log("\n =======================================");
        console2.log("balances for : ", name);
        console2.log("rewards token balance: ", k.rewardsToken.balanceOf(a));
        console2.log("=======================================\n");
    }


    function testDistribute1() public {   
        console2.log("\n testDistribution...........................");
        printk();
        // 1st distribution
        skip(k.vestParams.tau / 3);
        assertEq(k.rewardsToken.balanceOf(address(k.rewards)), 0, "Bad initial balance");

        printVestAward(1);

        vm.expectEmit(false, false, false, true, address(k.dist));
        emit Distribute(k.vestParams.tot / 3);
        console2.log("distribute 1........................");
        console2.log("distribute time: ", block.timestamp);
        k.dist.distribute();

       printBalances(address(k.rewards), "RewardsStaking");
     

        // 2nd distribution
        skip(k.vestParams.tau / 3);

        vm.expectEmit(false, false, false, true, address(k.dist));
        emit Distribute(k.vestParams.tot / 3);
        console2.log("distribute 2........................");
        k.dist.distribute();


        printBalances(address(k.rewards), "RewardsStaking");
         printBalances(address(k.dist), "Distribution");

        // 3rd distribution
        skip(k.vestParams.tau / 3);

        vm.expectEmit(false, false, false, true, address(k.dist));
        emit Distribute(k.vestParams.tot / 3);
        console2.log("distribute 3........................");
        k.dist.distribute();

       printBalances(address(k.rewards), "RewardsStaking");
        printBalances(address(k.dist), "Distribution");
    }

    

    function _setUpDistributionParams(
        DistributionParams memory _distParams
    ) internal returns (DistributionParams memory result) {
        result = _distParams;

        if (address(result.rewardsToken) == address(0)) {
            result.rewardsToken = IERC20Mintable(address(new SDAO("Token", "TKN")));
        }


        if (address(result.vest) == address(0)) {
            //result.vest = DssVestWithGemLike(      ?????
            //    deployCode("DssVest.sol:DssVestMintable", abi.encode(address(result.rewardsToken)))
            //);
            result.vest = DssVestWithGemLike(address(new DssVestMintable(address(result.rewardsToken))));
            result.vest.file("cap", type(uint256).max);
        }
   
        if (address(result.rewards) == address(0)) { // stakingTokens is zero?
            result.rewards = new StakingRewards(address(this), address(0), address(result.rewardsToken), address(stakingToken));
        }


        if (address(result.dist) == address(0)) {
            result.dist = new VestedRewardsDistribution(address(result.vest), address(result.rewards));
        }

        result.rewards.setRewardsDistribution(address(result.dist));
        _distParams.vestParams.usr = address(result.dist);            // distribution is the usr

        (result.vestId, result.vestParams) = _setUpVest(result.vest, _distParams.vestParams);
        result.dist.file("vestId", result.vestId);
        // Allow DssVest to mint tokens
        WardsLike(address(result.rewardsToken)).rely(address(result.vest));  // vest can call authorized fucntions in rewardsToken
    }

    function _setUpVest(
        DssVestWithGemLike _vest,
        address _usr
    ) internal returns (uint256 _vestId, VestParams memory result) {
        return _setUpVest(_vest, _usr, true);
    }

    function _setUpVest(
        DssVestWithGemLike _vest,
        address _usr,
        bool restrict
    ) internal returns (uint256 _vestId, VestParams memory result) {
        return _setUpVest(_vest, VestParams({usr: _usr, tot: 0, bgn: 0, tau: 0, eta: 0}), restrict);
    }

    function _setUpVest(
        DssVestWithGemLike _vest,
        VestParams memory _v
    ) internal returns (uint256 _vestId, VestParams memory result) {
        return _setUpVest(_vest, _v, true);
    }

    function _setUpVest(
        DssVestWithGemLike _vest,
        VestParams memory _v,
        bool restrict
    ) internal returns (uint256 _vestId, VestParams memory result) {
        result = _v;

        if (result.usr == address(0)) {
            revert("_setUpVest: usr not set");
        }
        if (result.tot == 0) {
            result.tot = 1.5 ether / 100000;      // change this ????
        }
        if (result.bgn == 0) {
            result.bgn = block.timestamp;
        }
        if (result.tau == 0) {
            result.tau = DEFAULT_DURATION;
        }
        if (result.eta == 0) {
            result.eta = DEFAULT_CLIFF;
        }

        _vestId = _vest.create(result.usr, result.tot, result.bgn, result.tau, result.eta, address(0));
        if (restrict) {
            _vest.restrict(_vestId);
        }
    }

    function _makeVestParams() internal pure returns (VestParams memory) {
        return VestParams({usr: address(0), tot: 0, bgn: 0, tau: 0, eta: 0});
    }

    bytes16 private constant _SYMBOLS = "0123456789abcdef";

    /**
     * @dev Converts a `uint256` to its ASCII `string` decimal representation.
     */
    function toString(uint256 value) internal pure returns (string memory) {
        unchecked {
            uint256 length = log10(value) + 1;
            string memory buffer = new string(length);
            uint256 ptr;
            /// @solidity memory-safe-assembly
            assembly {
                ptr := add(buffer, add(32, length))
            }
            while (true) {
                ptr--;
                /// @solidity memory-safe-assembly
                assembly {
                    mstore8(ptr, byte(mod(value, 10), _SYMBOLS))
                }
                value /= 10;
                if (value == 0) break;
            }
            return buffer;
        }
    }

    /**
     * @dev Return the log in base 10, rounded down, of a positive value.
     * Returns 0 if given 0.
     */
    function log10(uint256 value) internal pure returns (uint256) {
        uint256 result = 0;
        unchecked {
            if (value >= 10 ** 64) {
                value /= 10 ** 64;
                result += 64;
            }
            if (value >= 10 ** 32) {
                value /= 10 ** 32;
                result += 32;
            }
            if (value >= 10 ** 16) {
                value /= 10 ** 16;
                result += 16;
            }
            if (value >= 10 ** 8) {
                value /= 10 ** 8;
                result += 8;
            }
            if (value >= 10 ** 4) {
                value /= 10 ** 4;
                result += 4;
            }
            if (value >= 10 ** 2) {
                value /= 10 ** 2;
                result += 2;
            }
            if (value >= 10 ** 1) {
                result += 1;
            }
        }
        return result;
    }

    event Distribute(uint256 amount);
}

interface IERC20Mintable is IERC20 {
    function mint(address, uint256) external;
}

interface WardsLike {
    function rely(address) external;
}
```

### Mitigation

Introduce a new state variable called "remaining". When ```_supply = 0```, accrued rewards should be added to ```remaining```. Rewards accrued in ```remaining``` can be dispensed in another new stream. 