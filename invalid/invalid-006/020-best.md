Cheerful Pewter Sealion

High

# Fund Loss due to OverInflation of Burn Value in LockStateEngine Contract

### Summary

Fund Loss due to Over Inflation of Burn Value in LockStateEngine.sol Contract as  a result of error in fee reduction operation from sold value

### Root Cause

In https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/main/lockstake/src/LockstakeEngine.sol#L442 
```solidity
    function onRemove(address urn, uint256 sold, uint256 left) external auth {
        uint256 burn;
        uint256 refund;
        if (left > 0) {
 >>>           burn = _min(sold * fee / (WAD - fee), left);   ❌
            mkr.burn(address(this), burn);
            unchecked { refund = left - burn; }
            if (refund > 0) {
                // The following is ensured by the dog and clip but we still prefer to be explicit
                require(refund <= uint256(type(int256).max), "LockstakeEngine/overflow");
                vat.slip(ilk, urn, int256(refund));
                vat.frob(ilk, urn, urn, address(0), int256(refund), 0);
                lsmkr.mint(urn, refund);
            }
        }
        urnAuctions[urn]--;
        emit OnRemove(urn, sold, burn, refund);
    }
```
The pointer above from the onRemove(...) function shows how fee percentage value is being deducted from sold value to determine burn value, the problem is that the fee percentage is deducted wrongly. WAD is a representation of 100% while fee is a percentage of that 100%, so the correct way to to deduct the fee percentage from sold would have been (WAD - fee )/WAD * sold, so if fee is zero it would simply be  WAD/WAD * sold = sold. without any fee deduction. However the current implementation is a complete error as this would only inflate sold value depending on the fee value as fee is made the sole numerator

### Internal pre-conditions

 From this formular i.e  fee / (WAD - fee) and Depending on the value of fee
The more current value of fee set in contract heads towards 100% the more the over inflation from this formular.
The implication of this is that at the extreme case if fee is 99%, then burn calculation becomes 
        sold * 99 / (100 - 99) = sold * 99, inflating sold and burn value in the multiple of 99 when correct value should be a percentage of sold and not a multiple of it


### External pre-conditions

Due to this part of the formular i.e burn = _min(sold * fee / (WAD - fee), left); , it would be assumed that even if sold is overinflated the function _min(...) ensure the minimum value is selected in comparison to left variable, there the expected external precondition is that the value of left parameter will be set very high by the caller to ensure the overinflated value stands

### Attack Path

Once all this factors are in place, the caller calls the onRemove(...) function with a high left value and depending on the fee value, the resulting burn value is overinflated and thereby causing loss of fund in Protocol

### Impact

Fund Loss due to Over Inflation of Burn Value in LockStateEngine.sol Contract as  a result of error in fee reduction operation from sold value

### PoC

_No response_

### Mitigation

As adjusted below the correct way to calculate burn is to ensure it is a fragment percentage of sold with fee deducted
```solidity
    function onRemove(address urn, uint256 sold, uint256 left) external auth {
        uint256 burn;
        uint256 refund;
        if (left > 0) {
---            burn = _min(sold * fee / (WAD - fee), left);
+++            burn = _min(sold * (WAD - fee) / (WAD), left);
            mkr.burn(address(this), burn);
            unchecked { refund = left - burn; }
            if (refund > 0) {
                // The following is ensured by the dog and clip but we still prefer to be explicit
                require(refund <= uint256(type(int256).max), "LockstakeEngine/overflow");
                vat.slip(ilk, urn, int256(refund));
                vat.frob(ilk, urn, urn, address(0), int256(refund), 0);
                lsmkr.mint(urn, refund);
            }
        }
        urnAuctions[urn]--;
        emit OnRemove(urn, sold, burn, refund);
    }
```