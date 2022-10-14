hickuphh3

medium

# Loss of rewards if `pointsPerShare` increment exceeds `type(int256).max`

## Summary
Rewards would be unrecoverable if `pointsPerShare` is incremented to more than `type(int256).max`.

## Vulnerability Detail
When rewards are distributed by calling `distributeRewards()`, the internal function `_distributeRewards()` may cause `pointsPerShare` to overflow to a point where its multiplication with `shares` cannot be safely casted to `int256`.

### POC
Assume 
- `pointsPerShare = 0`
- `shares = getTotalShares() = 1` and `amount = (type(int256).max) / POINTS_MULTIPLIER + 1`

Then, `pointsPerShare` becomes
```solidity
_amount * POINTS_MULTIPLIER / shares
= (type(int256).max) / type(uint128).max + 1) * type(uint128).max / 1
= type(int256).max) + type(uint128).max 
```

This would then cause `cumulativeRewardsOf()` <- `withdrawableRewardsOf()` <- `_prepareCollect()` to revert ie. claiming of rewards to revert because the safecasting to `int256` would revert:
```solidity
return ((pointsPerShare * getSharesOf(_account)).toInt256() + pointsCorrection[_account]).toUint256() / POINTS_MULTIPLIER;
```
causing distributed rewards to be unclaimable.

## Impact
Rewards can never be claimed nor recovered.

## Code Snippet
https://github.com/Merit-Circle/merit-liquidity-mining/blob/ce5feaae19126079d309ac8dd9a81372648437f1/contracts/base/AbstractRewards.sol#L74

https://github.com/Merit-Circle/merit-liquidity-mining/blob/ce5feaae19126079d309ac8dd9a81372648437f1/contracts/base/AbstractRewards.sol#L95-L97

## Tool used
Manual Review

## Recommendation
In `_distributeRewards()`, ensure that `pointsPerShare * shares` can be safely casted to `toInt256`, ie. does not exceed `uint256(type(int256).max)`.

```diff
if (_amount > 0) {
   pointsPerShare = pointsPerShare + (_amount * POINTS_MULTIPLIER / shares);
+  if (pointsPerShare * shares  > uint256(type(int256).max)) revert PointsOverflow();
   emit RewardsDistributed(msg.sender, _amount);
}
```

P.S the unsafe casting of `pointsPerShare` in `_correctPoints` will be fine because `pointsPerShare` < `pointsPerShare * shares` <= `uint256(type(int256).max)`