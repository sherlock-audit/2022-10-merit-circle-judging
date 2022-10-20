Chom

high

# cumulativeRewardsOf logic is incorrect

## Summary
cumulativeRewardsOf logic is incorrect

## Vulnerability Detail
```solidity
 function cumulativeRewardsOf(address _account) public view override returns (uint256) { 
   return ((pointsPerShare * getSharesOf(_account)).toInt256() + pointsCorrection[_account]).toUint256() / POINTS_MULTIPLIER; 
 } 
```

pointsPerShare is first multiplied with getSharesOf(_account) then add it with pointsCorrection[_account]

But pointsCorrection[_account] is a correction of getSharesOf(_account), it should be added first to get net share to calculate the reward.

## Impact
cumulativeRewardsOf is completely wrong. Causing the reward system to be broken. Fund loss is happening anytime.

## Code Snippet
https://github.com/sherlock-audit/2022-10-merit-circle/blob/main/merit-liquidity-mining/contracts/base/AbstractRewards.sol#L73-L75

## Tool used

Manual Review

## Recommendation
1. getSharesOf(_account).toInt256() + pointsCorrection[_account] first
2. Multiply with pointsPerShare
3. Divide with POINTS_MULTIPLIER

```solidity
 function cumulativeRewardsOf(address _account) public view override returns (uint256) { 
   return (pointsPerShare * (getSharesOf(_account).toInt256() + pointsCorrection[_account])).toUint256() / POINTS_MULTIPLIER; 
 } 
```
