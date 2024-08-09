Curved Cinnamon Tuna

High

# Unauthorized Token Recovery via recoverERC20 Function

## Summary
The `recoverERC20` function allows the contract owner to recover any ERC20 token from the contract, except the staking token. While this is useful for recovering mistakenly sent tokens, it can be misused by a malicious to withdraw important tokens, potentially harming the users and the integrity of the contract.

## Vulnerability Detail
1. Initial State:
- The contract holds a significant amount of ERC20 tokens, intended for user rewards or other purposes.
- Users have staked their tokens and are expecting to receive rewards.
2. Malicious Action:
- The owner, either maliciously or due to a compromised private key, decides to exploit the `recoverERC20` function.
- The owner calls `recoverERC20` with the address of a valuable ERC20 token and the amount to transfer.
3. Execution:
- The `recoverERC20` function executes successfully, transferring the specified ERC20 tokens from the contract to the owner's address.
- Example: 
```solidity
stakingRewards.recoverERC20(address(rewardToken), rewardToken.balanceOf(address(stakingRewards)));
```
4. Depletion of Contract Funds:
- The contract's balance of the specified ERC20 tokens is depleted.
- Users who were expecting rewards or other distributions from these tokens are now left without their expected returns.

## Impact
- Financial Loss
- Erosion of Trust

## Code Snippet
https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/main/endgame-toolkit/src/synthetix/StakingRewards.sol#L166-L170

## Tool used

Manual Review

## Recommendation
- Implement role-based access control to restrict who can call `recoverERC20`.
- Introduce a time-lock mechanism to delay the execution of the `recoverERC20` function.
- Require multiple signatures from trusted parties before executing the `recoverERC20` function
- Implement monitoring and alert systems to detect and notify about any `recoverERC20` function calls.

## PoC
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import "forge-std/Test.sol";
import "openzeppelin-contracts/token/ERC20/ERC20.sol";
import "../src/synthetix/StakingRewards.sol";

// Custom ERC20 token with mint function
contract CustomERC20 is ERC20 {
    constructor(string memory name, string memory symbol) ERC20(name, symbol) {}

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
}

contract ExploitTest is Test {
    CustomERC20 rewardToken;
    CustomERC20 stakingToken;
    StakingRewards stakingRewards;
    address owner = address(0x1);
    address rewardsDistribution = address(0x2);
    address attacker = address(0x3);
    address user = address(0x4);

    function setUp() public {
        // Deploy custom ERC20 tokens
        rewardToken = new CustomERC20("Reward Token", "RWT");
        stakingToken = new CustomERC20("Staking Token", "STK");

        // Mint tokens
        rewardToken.mint(owner, 1000 ether);
        stakingToken.mint(user, 1000 ether);

        // Deploy StakingRewards contract
        stakingRewards = new StakingRewards(owner, rewardsDistribution, address(rewardToken), address(stakingToken));

        // Transfer reward tokens to staking contract
        vm.startPrank(owner);
        rewardToken.transfer(address(stakingRewards), 1000 ether);
        vm.stopPrank();

        // User stakes tokens
        vm.startPrank(user);
        stakingToken.approve(address(stakingRewards), 1000 ether);
        stakingRewards.stake(100 ether);
        vm.stopPrank();
    }

    function testExploit() public {
        // Simulate the owner calling recoverERC20
        vm.startPrank(owner);
        stakingRewards.recoverERC20(address(rewardToken), rewardToken.balanceOf(address(stakingRewards)));
        vm.stopPrank();

        // Verify that the contract's reward token balance is zero
        assertEq(rewardToken.balanceOf(address(stakingRewards)), 0);

        // Verify that the owner's reward token balance has increased
        assertEq(rewardToken.balanceOf(owner), 1000 ether);

        // Verify that the user's expected rewards are now zero
        assertEq(stakingRewards.earned(user), 0);

        // If all assertions pass, the test will pass
    }
}
```
forge test --match-path test/ExploitTest.sol
[⠒] Compiling...
[⠊] Compiling 1 files with Solc 0.8.25
[⠒] Solc 0.8.25 finished in 1.85s
Compiler run successful!

Ran 1 test for test/ExploitTest.sol:ExploitTest
[PASS] testExploit() (gas: 71295)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 22.75ms (8.98ms CPU time)

Ran 1 test suite in 23.81ms (22.75ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)

