Huge Teal Octopus

high

# Fees aren't distributed properly for positions with multiple lenders, causing loss of funds for lenders

## Summary
Fees distributed are calculated according to a lender's amount lent divided by the total amount lent, which causes more recent lenders to steal fees from older lenders.
## Vulnerability Detail
The fees distributed to each lender are determined by the following calculation (https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L546-L549):
```solidity
                uint256 feesAmt = FullMath.mulDiv(feesOwed, cache.holdTokenDebt, borrowedAmount); //fees owed multiplied by the individual amount lent, divided by the total amount lent
                ...
                loansFeesInfo[creditor][cache.holdToken] += feesAmt;
                harvestedAmt += feesAmt;
```
The above is from `harvest()`; `repay()` calculates the fees similarly. Because `borrow()` doesn't distribute fees, the following scenario will occur when a borrower increases an existing position:
1. Borrower has an existing position with fees not yet collected by the lenders.
2. Borrower increases the position with a loan from a new lender.
3. `harvest()` or `repay()` is called, and the new lender is credited with some of the previous fees earned by the other lenders due to the fees calculation. Other lenders lose fees.

This scenario can naturally occur during the normal functioning of the protocol, or a borrower/attacker with a position with a large amount of uncollected fees can maliciously open a proportionally large loan with an attacker to steal most of the fees.

Also note that ANY UDPATE ISSUE? LOW PRIO
## Impact
Loss of funds for lenders, potential for borrowers to steal fees.
## Code Snippet
https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L546-L549
## Tool used

Manual Review

## Recommendation
A potential fix is to harvest fees in the borrow() function; the scenario above will no longer be possible.
