Proud Porcelain Antelope

Medium

# NGT minting is not possible via DssVest

## Summary

As stated in the [README](https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/main/endgame-toolkit/README.md?plain=1#L52), rewards are going to be generated through a `DssVestMintable`, therefore it should have access to mint NGT tokens. However, no `ngt.rely(vest)` is being made in neither the `VestInit`, nor any other place in the scope.

## Vulnerability Detail

As was stated in the ChainSecurity's MakerDAO Endgame Toolkit audit, item 6.2 "Vest Minting Not Possible":
> DssVestMintable.pay() calls the NGT token's mint() function to generate tokens for the vesting.
The function is guarded and can only be accessed by a ward. The DssVestMintable contract is
never set as a ward of the NGT contract.

It's also stated in the audit that MakerDAO fixed the issue by adding the following line into the initialization script: `RelyLike(ngt).rely(vest);`.
In practice, it was not added, and the script still lacks this line.

## Impact

It is impossible for `VestedRewardsDistribution` to get rewards since `DssVest` won't be able to successfully execute the internal `pay` function. This leads to depositors of `stakingRewards` not getting their expected profit, which can be considered as a corresponding loss of funds for them.

## Code Snippet

https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/main/endgame-toolkit/script/dependencies/VestInit.sol#L31

## Tool Used

Manual Review

## Recommendation

Add `ngt` as a parameter to `VestInit.init`, and add `RelyLike(ngt).rely(vest);` into `VestInit.init`.
