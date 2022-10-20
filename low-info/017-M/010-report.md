8olidity

medium

# unit might be 0,The getMultiplier() function is not available

## Summary
unit might be 0
## Vulnerability Detail
Because of this
```solidity
uint256 public constant MIN_LOCK_DURATION = 10 minutes;
if (_maxLockDuration < MIN_LOCK_DURATION) {
    revert SmallMaxLockDuration();
}
```
maxLockDuration minimum is 600. If curve.length is greater than 600 on that day, unit will be 0 and getMultiplier() will calculate incorrectly
#### poc
```javascript
// merit-liquidity-mining/test/TimeLockPool.ts
const MAX_LOCK_DURATION = 3;
    it("Replacing with a same length curve should do it correctly", async() => {
        // Mapping a new curve for replacing the old one
        const NEW_CURVE = CURVE.map(function(x) {
            return (hre.ethers.BigNumber.from(x).mul(2).toString())
        })
        await timeLockPool.connect(deployer).setCurve(NEW_CURVE);
        console.log(await timeLockPool.maxLockDuration());  // console  3
        console.log(await timeLockPool.unit());    // console 0 

        for(let i=0; i< NEW_CURVE.length; i++){
            const curvePoint = await timeLockPool.curve(i);
            expect(curvePoint).to.be.eq(NEW_CURVE[i])
        }
        await expect(timeLockPool.curve(NEW_CURVE.length + 1)).to.be.reverted;
    })
```
## Impact
The getMultiplier() function is not available
## Code Snippet
https://github.com/sherlock-audit/2022-10-merit-circle/blob/main/merit-liquidity-mining/contracts/TimeLockPool.sol#L280-L311
```solidity
    function setCurve(uint256[] calldata _curve) external onlyGov {
        if (_curve.length < 2) {
            revert ShortCurveError();
        }
        // same length curves
        if (curve.length == _curve.length) {
            for (uint i=0; i < curve.length; i++) {
                curve[i] = maxBonusError(_curve[i]);
            }
        // replacing with a shorter curve
        } else if (curve.length > _curve.length) {
            for (uint i=0; i < _curve.length; i++) {
                curve[i] = maxBonusError(_curve[i]);
            }
            uint initialLength = curve.length;
            for (uint j=0; j < initialLength - _curve.length; j++) {
                curve.pop();
            }
            unit = maxLockDuration / (curve.length - 1);
        // replacing with a longer curve
        } else {
            for (uint i=0; i < curve.length; i++) {
                curve[i] = maxBonusError(_curve[i]);
            }
            uint initialLength = curve.length;
            for (uint j=0; j < _curve.length - initialLength; j++) {
                curve.push(maxBonusError(_curve[initialLength + j]));
            }
            unit = maxLockDuration / (curve.length - 1);
        }
        emit CurveChanged(_msgSender());
    }
```



## Tool used
vscode
Manual Review

## Recommendation
Judge unit's value after you calculate it