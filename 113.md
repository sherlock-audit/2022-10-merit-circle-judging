WATCHPUG

medium

# Lack of deadline control in `deposit()` may result in an unfavorable lock in some edge cases

## Summary

The `deposit()` transaction can get minted much later than expected in some edge cases, which means the end time of the lock may not be favorable by then.

## Vulnerability Detail

The lock end time of the deposit is decided by the time the transaction gets minted, which can be out of the user's control in some edge cases (network congestion, network went offline, etc).

For example:

- Alice sent a transaction to `deposit()` and lock for 1 day;
- The network delayed the transaction and only minted it 12 hours later;
- Alice's deposit is now set to unlock 24 hours later from the time the transaction got minted, 12 hours later than expected.

## Impact

`deposit()` can lock funds for a longer time than expected in some edge cases.

## Code Snippet

https://github.com/sherlock-audit/2022-10-merit-circle/blob/main/merit-liquidity-mining/contracts/TimeLockPool.sol#L85-L107

## Tool used

Manual Review

## Recommendation

Consider adding a `deadline` parameter and revert if `block.timestamp > deadline` in `deposit()`.