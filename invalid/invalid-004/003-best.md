Cuddly Inky Rat

Medium

# Loss of rewards when distribution occurs caused by # notifyRewardAmount

## Summary
The distribute function is responsible for distributing rewards that have accrued since the last distribution. It interacts with a vesting contract (dssVest) to fetch the amount of rewards that are due, then transfers these rewards to a staking rewards contract (stakingRewards). Finally, it notifies the staking rewards contract about the new reward amount.

However the some part of the  rewards in the contract will locked due to precision loss and trauncations
https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/main/endgame-toolkit/src/VestedRewardsDistribution.sol#L152

## Vulnerability Detail
When `distirbution` occurs to the function calls stakingRewards.notifyRewardAmount(amount). This function in the StakingRewards contract adjusts the internal accounting to account for the new rewards. 

In the notifyRewardAMount there is sought of precision loss, meaning users get to get less of their notified reward rather than expected, This can be seen here:
```solidity
 function notifyRewardAmount(uint256 reward) external override onlyRewardsDistribution updateReward(address(0)) {
        if (block.timestamp >= periodFinish) {
            rewardRate = reward / rewardsDuration;
        } else {
            uint256 remaining = periodFinish - block.timestamp;
            uint256 leftover = remaining * rewardRate;
            rewardRate = (reward + leftover) / rewardsDuration;
        }

        // Ensure the provided reward amount is not more than the balance in the contract.
        // This keeps the reward rate in the right range, preventing overflows due to
        // very high values of rewardRate in the earned and rewardsPerToken functions;
        // Reward + leftover must be less than 2^256 / 10^18 to avoid overflow.
        uint256 balance = rewardsToken.balanceOf(address(this));
        require(rewardRate <= balance / rewardsDuration, "Provided reward too high");

        lastUpdateTime = block.timestamp;
        periodFinish = block.timestamp + rewardsDuration;
        emit RewardAdded(reward);
    }

```
How can this occur, if you look at the function logic you can see that there can be a precision loss because of the way the integer is been handled, Solidity only supports integer division, which can result in precision loss. This precision loss can become significant when dealing with reward rates and durations, causing the calculations to deviate from the expected values.

The staking rewards contract is initialized with a `rewardsDuration` of 7 days (604800 seconds).
The `periodFinish` is the timestamp when the current reward period ends.
Initially, rewardRate is 0, and the contract has a balance of 1000 `rewardsToken`.

Assume the current `periodFinish` is `July 1st, 2024`, and Alice adds a reward on `July 2nd, 2024` (after the current period has finished).
Alice calls `notifyRewardAmount` with reward = 700.

Since `block.timestamp` is after `periodFinish`, the condition `block.timestamp` >= `periodFinish` is true.
The `rewardRate` is calculated as `reward` / `rewardsDuration`, which is 700 / 604800 = 0 (due to integer division).

The balance in the contract is 1000 tokens.
`balance / rewardsDuration` is 1000 / 604800 = 0 (due to integer division).
The check `require(rewardRate <= 0)` is true since rewardRate is 0.
The function proceeds without reverting.

`lastUpdateTime` is updated to the current timestamp `(July 2nd, 2024).`
`periodFinish` is set to July 9th, 2024.
RewardAdded event is emitted with reward = 700.

The rewardRate is set to 0 due to integer division (700 / 604800).
This means no rewards will be distributed over the new period despite adding 700 tokens.


## Impact
This indicates a significant issue because the actual reward distribution  does not match the intended reward

## Code Snippet
```solidity
function distribute() external returns (uint256 amount) {
        require(vestId != INVALID_VEST_ID, "VestedRewardsDistribution/invalid-vest-id");

        amount = dssVest.unpaid(vestId);
        require(amount > 0, "VestedRewardsDistribution/no-pending-amount");

        lastDistributedAt = block.timestamp;
        dssVest.vest(vestId, amount);

        require(gem.transfer(address(stakingRewards), amount), "VestedRewardsDistribution/transfer-failed");
        stakingRewards.notifyRewardAmount(amount);

        emit Distribute(amount);
    }
```
```solidity
 function notifyRewardAmount(uint256 reward) external override onlyRewardsDistribution updateReward(address(0)) {
        if (block.timestamp >= periodFinish) {
            rewardRate = reward / rewardsDuration;
        } else {
            uint256 remaining = periodFinish - block.timestamp;
            uint256 leftover = remaining * rewardRate;
            rewardRate = (reward + leftover) / rewardsDuration;
        }

        // Ensure the provided reward amount is not more than the balance in the contract.
        // This keeps the reward rate in the right range, preventing overflows due to
        // very high values of rewardRate in the earned and rewardsPerToken functions;
        // Reward + leftover must be less than 2^256 / 10^18 to avoid overflow.
        uint256 balance = rewardsToken.balanceOf(address(this));
        require(rewardRate <= balance / rewardsDuration, "Provided reward too high");

        lastUpdateTime = block.timestamp;
        periodFinish = block.timestamp + rewardsDuration;
        emit RewardAdded(reward);
    }
```

## Tool used
Manual Review

## Recommendation
Find a way to prevent precision loss, either by using safeMath because the notifyRewardAmount shouldn't be in anyway decreasing.