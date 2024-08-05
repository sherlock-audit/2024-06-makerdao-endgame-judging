Helpful Dijon Spider

Medium

# Calling `SNStInit` just 1 block after `SNstDeploy` will always revert and deposits will be stuck getting no yield

### Summary

Not calling `SNst::drip()` before `SNst::file()` in `SNstInit::init()` will DoS the protocol until a manual fix is applied. Depositing at the same block that `SNst` is deployed will get the funds stuck getting no yield until the manual fix comes through.

### Root Cause

[SNst::init()](https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/main/sdai/deploy/SNstInit.sol#L62-L64) does not call `SNst::drip()` before `SNst::file()`:
```solidity
function init(
    ...
) internal {
    ...
    dss.vat.rely(instance.sNst);
     //@audit note that SNst::drip() is not called, which reverts if block.timestamp != rho
    SNstLike(instance.sNst).file("nsr", cfg.nsr);
    ...
}
```
So it reverts on [SNst::file()](https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/main/sdai/src/SNst.sol#L206):
```solidity
function file(bytes32 what, uint256 data) external auth {
    ...
        require(rho == block.timestamp, "SNst/chi-not-up-to-date");
    ...
}
```
Additionally, deposits can be made when `block.timestamp == rho`, as `SNst::drip()` does [not](https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/main/sdai/src/SNst.sol#L226) call `vat::suck()`, which is what makes it revert as `vat::rely()` has not yet been set. After 1 block, they can not withdraw because `block.timestamp != rho` and `SNst::drip()` calls [vat::suck()](https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/main/sdai/src/SNst.sol#L221), which reverts.
```solidity
function drip() public returns (uint256 nChi) {
    ...
    if (block.timestamp > rho_) {
        ...
        vat.suck(address(vow), address(this), diff * RAY); //@audit reverts before vat::rely() is called in SNstInit
        ...
    } else { //@audit when block.timestamp == rho it goes here, allowing deposits
        nChi = chi_;
    }
    ...
}
```


### Internal pre-conditions

1. `SNstInit::init()` must not be called in the same block `SNstDeploy::deploy()` is called.

### External pre-conditions

None.

### Attack Path

1. `SNstDeploy::deploy()` deploys the token.
2. `$NST` is deposited into `SNst` in the same block of the deployment.
3. Unless `SNstInit::init()` is called in the same block as the previous 2 points, it will be impossible to withdraw and deposits will get no yield.

### Impact

Funds are stuck until a manual `Vat::rely()` call is made allowing `SNst::drip()` to be called which allows withdrawals. When this call happens some time in the future, yield will be lost because the `nsr` is initially set to `1`, so deposits will get no interest.

### PoC

Add the following test to `SNst-integration.t.sol`:
```solidity
function testStuckDeposit() public {
    SNstInstance memory inst = SNstDeploy.deploy(address(this), pauseProxy, address(nstJoin));
    token = SNst(inst.sNst);

    address user = makeAddr("user");
    vm.startPrank(user);
    deal(address(nst), user, 100 ether);
    nst.approve(address(token), type(uint256).max);
    token.deposit(100 ether, user);
    vm.stopPrank();

    skip(1);

    vm.startPrank(pauseProxy);
    vm.expectRevert("SNst/chi-not-up-to-date");
    token.file("nsr", 1000000001547125957863212448); // init lib would also revert
    vm.stopPrank();

    vm.startPrank(user);
    vm.expectRevert("Vat/not-authorized"); // drip() fails as rely() was not called
    token.redeem(100 ether, user, user);
}
```

### Mitigation

`SNst::init()` must call `SNst::drip()` before `SNst::file()`:
```solidity
function init(
    ...
) internal {
    ...
    dss.vat.rely(instance.sNst);
    
    SNstLike(instance.sNst).drip(); //@audit add this so SNst::file() does not revert

    SNstLike(instance.sNst).file("nsr", cfg.nsr);
    ...
}
```
Additionally, deposits must not be allowed at the block `SNst::deploy()` is called, or they will be stuck until `SNst::init()` is executed, which could take some time. The easiest fix is to set `rho` 1 block in the past, such that deposits will revert on `SNst::drip()` when calling `vat::suck()` before `vat::rely()` has been called in `SNst::init()`:
```solidity
function initialize() initializer external {
    ...
    rho = uint64(block.timestamp - 1); //@audit this stops deposits before SNst::init() is called
    ...
}
```