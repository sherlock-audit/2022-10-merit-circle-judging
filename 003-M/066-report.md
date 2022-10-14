innertia

medium

# Unsafe cast from uint256 to uint64

## Summary
Unsafe casts lead to setting unintended end times.
## Vulnerability Detail
The `duration` used in the deposit is calculated as `uint256`, but is cast as `uint64` when it is finally assigned.
This can cause overflow and lead to unintended configuration.
## Impact
The user may not get the expected behavior, rewards, etc. due to a different time period than intended by the user.
## Code Snippet
https://github.com/sherlock-audit/2022-10-merit-circle/blob/main/merit-liquidity-mining/contracts/TimeLockPool.sol#L102  

https://github.com/sherlock-audit/2022-10-merit-circle/blob/main/merit-liquidity-mining/contracts/TimeLockPool.sol#L182
## Tool used

Manual Review

## Recommendation
should use `SafeCast`.