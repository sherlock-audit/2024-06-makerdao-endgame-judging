Fancy Cloth Orca

High

# h-01 reentrant with stolen of funds  0xaliyah

## Summary

1. while the temp. var; `balance` L222 is the misinformation toward the effect at L237
2. while the temp. var; `balance` L222 is the lagging indication toward the effect at L237 if the `from` address was made any withdrawal or any transferFrom in the way that induced that reentry
3. L222 is the lagging indication
4. the `from` address made a withdrawal when the `transferFrom` function gave up control to the attacker at L222
5. since withdraw made and L222 now stale then L237 give the now empty-balance `from` address free increment

## Vulnerability Detail

1. recipients for free balance increment may be found

## Impact

1. high impact + high likeliness owasp

## Code Snippet

[poc](https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/main/endgame-toolkit/src/SDAO.sol#L237)

## Tool used

Manual Review

## Recommendation

[checks effects interactions](https://fravoll.github.io/solidity-patterns/checks_effects_interactions.html)
[Will Shahda](https://medium.com/coinmonks/protect-your-solidity-smart-contracts-from-reentrancy-attacks-9972c3af7c21)