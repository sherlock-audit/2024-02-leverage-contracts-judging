Huge Teal Octopus

high

# Entrance fees are distributed wrongly in loans with multiple lenders

## Summary
Entrance fees are distributed improperly, some lenders are likely to lose some portion of their entrance fees. Also, calling `updateHoldTokenEntranceFee()` can cause improper entrance fee distribution in loans with multiple lenders.
## Vulnerability Detail
Note that entrance fees are added to the borrower's `feesOwed` when borrowing:
```solidity
        borrowing.feesOwed += entranceFee;
```
Also note that the fees distributed to each lender are determined by the following calculation (https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L546-L549):
```solidity
                uint256 feesAmt = FullMath.mulDiv(feesOwed, cache.holdTokenDebt, borrowedAmount); //fees owed multiplied by the individual amount lent, divided by the total amount lent
                ...
                loansFeesInfo[creditor][cache.holdToken] += feesAmt;
                harvestedAmt += feesAmt;
```
This is a problem because the entrance fees will be distributed among all lenders instead of credited to each lender. Example:
1. A borrower takes a loan of 100 tokens from a lender and pays an entrance fee of 10 tokens.
2. After some time, the lender harvests fees and fees are set to zero. (This step could be frontrunning the below step.)
3. The borrower immediately takes out another loan of 100 tokens and pays and entrance fee of 10 tokens.
4. When fees are harvested again, due to the calculation in the code block above, 5 tokens of the entrance fee go to the first lender and 5 tokens go to the second lender. The first lender has collected 15 tokens of entrance fees, while the second lender has collected only 5- despite both loans having the same borrowed amount.

Furthermore, if the entrance fee is increased then new lenders will lose part of their entrance fee. Example:
1. A borrower takes a loan of 100 tokens from a lender and pays an entrance fee of 10 tokens.
2. The entrance fee is increased.
3. The borrower increases the position by taking a loan of 100 tokens from a new lender, and pays an entrance fee of 20 tokens.
4. `harvest()` is called, and both lenders receive 15 tokens out of the total 30 tokens paid as entrance fees. This is wrong since the first lender should receive 10 and the second lender should receive 20.
## Impact
Lenders are likely to lose entrance fees.
## Code Snippet
https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L1036
https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L546-L549
## Tool used

Manual Review

## Recommendation
Could add the entrance fee directly to the lender's fees balance instead of adding it to feesOwed, and then track the entrance fee in the loan data to be used in min fee enforcement calculations.
