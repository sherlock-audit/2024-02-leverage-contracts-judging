# Issue H-1: Entrance fees are distributed wrongly in loans with multiple lenders 

Source: https://github.com/sherlock-audit/2024-02-leverage-contracts-judging/issues/39 

## Found by 
0xDetermination
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



## Discussion

**sherlock-admin**

The protocol team fixed this issue in PR/commit https://github.com/RealWagmi/wagmi-leverage/commit/84416fcedfcc7eb062917bdc69f919bba9d3c0b7.

**fann95**

Yes, the problem existed and is associated with the same error as #41.
This issue is related to an erroneous scheme for accumulating fees and affected almost all functions in the contract, so the PR turned out to be quite large.

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid; high(1)



**nevillehuang**

@fann95 Is the root cause the same as #41?

**fann95**

I think so since it was assumed that the entrance fee would be distributed the same way as the fees for borrowing.

**nevillehuang**

@fann95 Can you take a look at this [comment](https://github.com/sherlock-audit/2024-02-leverage-contracts-judging/issues/40#issuecomment-1974818523) and let me know your thoughts

**fann95**

> Can you take a look at this [comment](https://github.com/sherlock-audit/2024-02-leverage-contracts-judging/issues/40#issuecomment-1974818523) and let me know your thoughts

done


**nevillehuang**

See comments [here](https://github.com/sherlock-audit/2024-02-leverage-contracts-judging/issues/40#issuecomment-1982026558)

# Issue H-2: Fees aren't distributed properly for positions with multiple lenders, causing loss of funds for lenders 

Source: https://github.com/sherlock-audit/2024-02-leverage-contracts-judging/issues/41 

## Found by 
0xDetermination, zraxx
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



## Discussion

**sherlock-admin**

The protocol team fixed this issue in PR/commit https://github.com/RealWagmi/wagmi-leverage/commit/84416fcedfcc7eb062917bdc69f919bba9d3c0b7.

**fann95**

Yes, the problem existed and is associated with the same error as #39.
This issue is related to an erroneous scheme for accumulating fees and affected almost all functions in the contract, so the PR turned out to be quite large.

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: high(1)



**nevillehuang**

See comments [here](https://github.com/sherlock-audit/2024-02-leverage-contracts-judging/issues/40#issuecomment-1982026558)

