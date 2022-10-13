minhquanym

high

# Flash loan vulnerability - User can bypass `MIN_LOCK_DURATION` limit

## Summary
User can use `increaseLock(...)` function to bypass the min duration limit in `TimeLockPool`
https://github.com/sherlock-audit/2022-10-merit-circle/blob/main/merit-liquidity-mining/contracts/TimeLockPool.sol#L197

## Vulnerability Detail
In `TimeLockPool`, there is a min lockup duration `MIN_LOCK_DURATION = 10 minutes` to prevent flash loan or MEV transaction ordering. However, exploiter can trick this limit by using `increaseLock(...)` function. Exploiter can create a lock with minimal amount every block and he will wait for the lock to be ended in the next block and deposit using `increaseLock(...)` function.

## Impact
Since user can deposit without the min lockup duration limit, user can use flash loan to get huge amount of shares, claim reward, pay back the loan without any risk.

## Code Snippet
Function `increaseLock(...)` does not check the min lockup duration
```solidity
function increaseLock(uint256 _depositId, address _receiver, uint256 _increaseAmount) external {
    // Check if actually increasing
    if (_increaseAmount == 0) {
        revert ZeroAmountError();
    }

    Deposit memory userDeposit = depositsOf[_msgSender()][_depositId];

    // Only can extend if it has not expired
    if (block.timestamp >= userDeposit.end) {
        revert DepositExpiredError();
    }

    depositToken.safeTransferFrom(_msgSender(), address(this), _increaseAmount);

    // Multiplier should be acording the remaining time to the deposit to end
    uint256 remainingDuration = uint256(userDeposit.end - block.timestamp);

    uint256 mintAmount = _increaseAmount * getMultiplier(remainingDuration) / 1e18;

    depositsOf[_receiver][_depositId].amount += _increaseAmount;
    depositsOf[_receiver][_depositId].shareAmount += mintAmount;

    _mint(_receiver, mintAmount);
    emit LockIncreased(_depositId, _receiver, _msgSender(), _increaseAmount);
}
```

## Tool used

Manual Review

## Recommendation

Consider adding min lockup duration check in `increaseLock(...)` function.
