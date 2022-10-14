WATCHPUG

medium

# Curve points should be guaranteed to be monotonic increasing

## Summary

Lack of checks to ensure the points in the curve are monotonic increasing, which can result in a malfunction of `deposit()` / `extendLock()` due to underflow when the curve is not set properly.

## Vulnerability Detail

In the current implementation, `getMultiplier()` assume the later point in the curve is always bigger than the previous point, otherwise `curve[n + 1] - curve[n]` will revert due to underflow.

However, since there is no check in `__TimeLockPool_init()` / `setCurve()` / `setCurvePoint()` to guarantee that, a lower point can actually be set after a higher point.

## Impact

`deposit()` / `extendLock()` may revert due to underflow.

## Code Snippet

https://github.com/sherlock-audit/2022-10-merit-circle/blob/main/merit-liquidity-mining/contracts/TimeLockPool.sol#L35-L63

https://github.com/sherlock-audit/2022-10-merit-circle/blob/main/merit-liquidity-mining/contracts/TimeLockPool.sol#L280-L311

https://github.com/sherlock-audit/2022-10-merit-circle/blob/main/merit-liquidity-mining/contracts/TimeLockPool.sol#L322-L337

## Tool used

Manual Review

## Recommendation

Consider adding a new internal function to validate the curve points:

```solidity
function checkCurve(uint256[] calldata _curve) internal {
    if (_curve.length < 2) {
        revert ShortCurveError();
    }
    for (uint256 i; i < curve.length - 1; ++i) {
            if (
                curve[i + 1] < curve[i]
            ) {
                revert CurveIncreaseError();
            }
        }
}
```