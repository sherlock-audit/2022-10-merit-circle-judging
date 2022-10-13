hyh

high

# Unit isn't recalculated on curve modification with setCurvePoint

## Summary

TimeLockPool's setCurvePoint() can add or remove curve points, changing its length.

It leaves `unit` variable that governs the duration to multiplier correspondence incorrect as it isn't updated.

## Vulnerability Detail

`unit` becomes incorrect after `onlyGov` setCurvePoint() updates the curve whenever its size changes.

As setCurvePoint() is an independent operation, the `unit` will just stay incorrect after it. I.e. setCurve() and __TimeLockPool_init() that update the `unit` are alternatives to setCurvePoint(), there is no use cases when they run after setCurvePoint(), fixing the variable. So here it is one scenario, where setCurvePoint() is run and `unit` is incorrect after that.

## Impact

When `unit` is incorrect the getMultiplier() calculations become incorrect too.

Suppose `curve.length` was `6`, `unit = maxLockDuration / (curve.length - 1) = maxLockDuration / 5`, then setCurvePoint() shortened the curve, so `curve.length` becomes `3`, but still `unit = maxLockDuration / 5`. Now `getMultiplier(2 * maxLockDuration / 5)` yields maximum shares multiplier as the very end of the curve will be used, while the lock is only `2 / 5` of the `maxLockDuration`.

The impact is duration multiplier logic either removes the shares from users (when setCurvePoint() increased the length) or provides them with extra shares (when setCurvePoint() decreased the length), in violation of the desired logic. As the shares have monetary value and the precondition is just running a routine setup operation, setting the severity to be high.

## Code Snippet

setCurvePoint() can change curve length, but leaves `unit` variable intact:

https://github.com/sherlock-audit/2022-10-merit-circle/blob/main/merit-liquidity-mining/contracts/TimeLockPool.sol#L313-L337

```solidity
    /**
     * @notice Can set a point of the curve.
     * @dev This function can replace any point in the curve by inputing the existing index,
     * add a point to the curve by using the index that equals the amount of points of the curve,
     * and remove the last point of the curve if an index greated than the length is used. The first
     * point of the curve index is zero.
     * @param _newPoint uint256 point to be set.
     * @param _position uint256 position of the array to be set (zero-based indexing convention).
     */
    function setCurvePoint(uint256 _newPoint, uint256 _position) external onlyGov {
        if (_newPoint > maxBonus) {
            revert MaxBonusError();
        }
        if (_position < curve.length) {
            curve[_position] = _newPoint;
        } else if (_position == curve.length) {
            curve.push(_newPoint);
        } else {
            if (curve.length - 1 < 2) {
                revert ShortCurveError();
            }
            curve.pop();
        }
        emit CurveChanged(_msgSender());
    }
```

Currently `unit` is recalculated on initialization:

https://github.com/sherlock-audit/2022-10-merit-circle/blob/main/merit-liquidity-mining/contracts/TimeLockPool.sol#L62

```solidity
    unit = _maxLockDuration / (curve.length - 1);
```

And on altering curve length with setCurve(), which rebuilds the entire curve:

https://github.com/sherlock-audit/2022-10-merit-circle/blob/main/merit-liquidity-mining/contracts/TimeLockPool.sol#L284-L309

```solidity
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
```

`unit` maps `_lockDuration` to the point on the curve, breaking this link if curve length had changed, while `unit` stayed the same:

https://github.com/sherlock-audit/2022-10-merit-circle/blob/main/merit-liquidity-mining/contracts/TimeLockPool.sol#L233-L246

```solidity
    function getMultiplier(uint256 _lockDuration) public view returns(uint256) {
        // There is no need to check _lockDuration amount, it is always checked before
        // in the functions that call this function

        // n is the time unit where the lockDuration stands
        uint n = _lockDuration / unit;
        // if last point no need to interpolate
        // trim de curve if it exceedes the maxBonus // TODO check if this is needed
        if (n == curve.length - 1) {
            return 1e18 + curve[n];
        }
        // linear interpolation between points
        return 1e18 + curve[n] + (_lockDuration - n * unit) * (curve[n + 1] - curve[n]) / unit;
    }
```

## Tool used

Manual Review

## Recommendation

Consider recalculation `unit` on the spot each time:

https://github.com/sherlock-audit/2022-10-merit-circle/blob/main/merit-liquidity-mining/contracts/TimeLockPool.sol#L313-L337

```solidity
    /**
     * @notice Can set a point of the curve.
     * @dev This function can replace any point in the curve by inputing the existing index,
     * add a point to the curve by using the index that equals the amount of points of the curve,
     * and remove the last point of the curve if an index greated than the length is used. The first
     * point of the curve index is zero.
     * @param _newPoint uint256 point to be set.
     * @param _position uint256 position of the array to be set (zero-based indexing convention).
     */
    function setCurvePoint(uint256 _newPoint, uint256 _position) external onlyGov {
        if (_newPoint > maxBonus) {
            revert MaxBonusError();
        }
        if (_position < curve.length) {
            curve[_position] = _newPoint;
        } else if (_position == curve.length) {
            curve.push(_newPoint);
+           unit = maxLockDuration / (curve.length - 1);
        } else {
            if (curve.length - 1 < 2) {
                revert ShortCurveError();
            }
            curve.pop();
+           unit = maxLockDuration / (curve.length - 1);
        }
        emit CurveChanged(_msgSender());
    }
```

setCurvePoint() is rare enough operation and cost of incorrect `unit` is rather big here, so the additional gas cost is justified.