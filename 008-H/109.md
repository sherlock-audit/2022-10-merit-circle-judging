WATCHPUG

medium

# Extend lock period should never result in a decrease of overall rewards (`total length of locked period * shares`)

## Summary

The current implementation may result in a burn of shares when `extendLock()`, which may result in a decrease of overall rewards (`total length of locked period * shares`).

## Vulnerability Detail

In the current implementation, when the user calls `extendLock()` for a deposit with `duration1` at time `t`, with `_increaseDuraiton = duration2`, it will create a new lock with a lock period of `duration1 + duration2 - time`.

For example:

If the multiplier for a 6 mos lock is 150% and 200% for a 1 year lock.

- Alice deposited 100 $MC tokens with the initial lock of 1 year, received 200 shares;
- 6 mos later, Alice called `extendLock()` to extend the deposit's lock for 10 more minutes (`MIN_LOCK_DURATION`), ~50 shares will be burned.

Expected result:

The total rewards should be no less than a 1 year lock;

Actual result:

The total rewards should is only `200 * 6 mos + 150 * 6 mos` vs `200 * 12 mos` for a 1 year lock;

By extend the lock for 10 more mins, Alice actually reduced the total rewards by 12.5%.

## Impact

Users may lose rewards by extending the lock.

## Code Snippet

https://github.com/Merit-Circle/merit-liquidity-mining/blob/ce5feaae19126079d309ac8dd9a81372648437f1/contracts/TimeLockPool.sol#L148-L184

## Tool used

Manual Review

## Recommendation

Consider introducing a new concept called `deferredLock` when `extendLock()` will result in the `newEndTime - block.timestamp < currentDuration`, in which case, instead of burning shares, it will set a deferred lock to be executable only after the current lock expires:

```solidity
function extendLock(uint256 _depositId, uint256 _increaseDuration) external {
    // Check if actually increasing
    if (_increaseDuration == 0) {
        revert ZeroDurationError();
    }

    Deposit memory userDeposit = depositsOf[_msgSender()][_depositId];

    // Only can extend if it has not expired
    if (block.timestamp >= userDeposit.end) {
        revert DepositExpiredError();
    }
    
    // Enforce min increase to prevent flash loan or MEV transaction ordering
    uint256 increaseDuration = _increaseDuration.max(MIN_LOCK_DURATION);
    
    // New duration is the time expiration plus the increase
    uint256 duration = maxLockDuration.min(uint256(userDeposit.end - block.timestamp) + increaseDuration);

    uint256 mintAmount = userDeposit.amount * getMultiplier(duration) / 1e18;

    // Multiplier curve changes with time, need to check if the mint amount is bigger, equal or smaller than the already minted
    
    // If the new amount if bigger mint the difference
    if (mintAmount > userDeposit.shareAmount) {
        depositsOf[_msgSender()][_depositId].shareAmount =  mintAmount;
        _mint(_msgSender(), mintAmount - userDeposit.shareAmount);

        depositsOf[_msgSender()][_depositId].start = uint64(block.timestamp);
        depositsOf[_msgSender()][_depositId].end = uint64(block.timestamp) + uint64(duration);
        emit LockExtended(_depositId, _increaseDuration, _msgSender());

        // reset deferredLock if any
        if (deferredLocks[_msgSender()][_depositId] > 0) {
            deferredLocks[_msgSender()][_depositId] = 0;
        }
    // If the new amount is less then set a deferred lock
    } else if (mintAmount < userDeposit.shareAmount) {
        deferredLocks[_msgSender()][_depositId] = increaseDuration
    }
}
```

For expired locks, when there is a `deferredLock`, extend the lock and burn the differences in shares:

```solidity
function kick(uint256 _depositId, address _user) external {
    if (_depositId >= depositsOf[_user].length) {
        revert NonExistingDepositError();
    }
    Deposit memory userDeposit = depositsOf[_user][_depositId];
    if (block.timestamp < userDeposit.end) {
        revert TooSoonError();
    }

    uint256 newSharesAmount = userDeposit.amount;

    uint256 deferredLock = deferredLocks[_user][_depositId];
    if (deferredLock != 0) {
        newSharesAmount = userDeposit.amount * getMultiplier(deferredLock) / 1e18;

        depositsOf[_user][_depositId].start = uint64(block.timestamp);
        depositsOf[_user][_depositId].end = uint64(block.timestamp) + uint64(deferredLock);
        emit LockExtended(_depositId, _increaseDuration, _user);
    }

    // burn pool shares
    _burn(_user, userDeposit.shareAmount - newSharesAmount);
}
```