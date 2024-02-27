Huge Teal Octopus

medium

# Minimum loan fee isn't enforced properly for multiple-loan positions

## Summary
A check in `borrow()` is used in conjunction with the `_calcFeeCompensationUpToMin()` in `repay()` to enforce a minimum fee amount per loan. This min fee enforcement is important for HFT activity, but the minimum loan fee is not fully applied for positions with multiple loans.
## Vulnerability Detail
The protocol sponsors have stated that `Constants.MINIMUM_AMOUNT` is to be used as a minimum fee per loan (on top of the entrance fee). This fee is not enforced correctly for borrower positions with multiple loans. This is due to 2 factors, the first is in `_precalculateBorrowing()` which is called in `borrow()`:
```solidity
            cache.borrowedAmount = _extractLiquidity(
                zeroForSaleToken,
                params.saleToken,
                params.holdToken,
                params.loans 
            );
            // the empty loans[] disallowed
            (cache.borrowedAmount == 0).revertError(ErrLib.ErrorCode.LOANS_IS_EMPTY);
            // Increment the total borrowed amount for the hold token information
            holdTokenRateInfo.totalBorrowed += cache.borrowedAmount;
        }
        // Calculate the prepayment per day fees based on the borrowed amount and daily rate collateral
        cache.dailyRateCollateral = FullMath.mulDivRoundingUp(
            cache.borrowedAmount,
            cache.dailyRateCollateral,
            Constants.BP 
        );
        // Check if the dailyRateCollateral is less than the minimum amount defined in the Constants contract
        if (cache.dailyRateCollateral < Constants.MINIMUM_AMOUNT) {
            cache.dailyRateCollateral = Constants.MINIMUM_AMOUNT;
        }
```
We can see above that if a borrower takes out a loan for the position such that the calculated 1-day collateral is less than `Constants.MINIMUM_AMOUNT`, `cache.dailyRateCollateral` will be set to the min amount (this will later be added to the buyer's collateral to transfer to the protocol).

This works fine for 1 loan, but for 2 loans or more the minimum fee per loan won't be enforced for all the loans, only one of them.

Notice how the minimum fee is enforced in `repay()` (in conjunction with `borrow()`):
```solidity
            if (collateralBalance > 0) {
                uint256 compensation = _calcFeeCompensationUpToMin(
                    collateralBalance,
                    currentFees,
                    borrowing.feesOwed
                );
                currentFees += compensation;
```
```solidity
    function _calcFeeCompensationUpToMin(
        int256 collateralBalance,
        uint256 currentFees,
        uint256 feesOwed
    ) private pure returns (uint256 compensation) {
        uint256 minimum = Constants.MINIMUM_AMOUNT * Constants.COLLATERAL_BALANCE_PRECISION;
        uint256 total = currentFees + feesOwed;
        if (total < minimum) {
            compensation = minimum - total;
            if (uint256(collateralBalance) < compensation) {
                compensation = uint256(collateralBalance);
            }
        }
    }
```
As we can see the min collateral enforcement in `borrow()` works in tandem with the min fee enforcement in `repay()` to take the min fee from the collateral balance if necessary and enforce the min fee per loan. But similarly here in `repay()` as in `borrow()`, this fee is only enforced for one loan; if a position has multiple loans then the minimum fee per loan won't be enforced.
## Impact
Borrowers can avoid paying some minimum loan fees.
## Code Snippet
https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L925-L945
https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L617-L623
https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L758-L771
## Tool used

Manual Review

## Recommendation
First, the below check should multiply the min amount by `params.loans.length`:
```solidity
        if (cache.dailyRateCollateral < Constants.MINIMUM_AMOUNT) {
            cache.dailyRateCollateral = Constants.MINIMUM_AMOUNT;
        }
```
Second, the minimum calcuation below from `_calcFeeCompensationUpToMin()` should also be multiplied by the number of loans.
```solidity
        uint256 minimum = Constants.MINIMUM_AMOUNT * Constants.COLLATERAL_BALANCE_PRECISION;
```
