Huge Teal Octopus

medium

# `liquidationBonus` may be forever unclaimable if a lender burns their NFT

## Summary
In the case that normal liquidation is impossible and emergency liquidation must be used, a lender burning the UniV3 NFT can lock the `liquidationBonus` forever.
## Vulnerability Detail
Notice that the `liquidationBonus` is only transferred to the last emergency liquidator, i.e. when `completeRepayment` equals `true`:
```solidity
            if (completeRepayment) {
                LoanInfo[] memory empty;
                _removeKeysAndClearStorage(borrowing.borrower, params.borrowingKey, empty);
                feesAmt += liquidationBonus;
            } else {
                ...
            }
            holdTokenOut = removedAmt + feesAmt;
            // Transfer removedAmt + feesAmt to msg.sender and emit EmergencyLoanClosure event
            Vault(VAULT_ADDRESS).transferToken(borrowing.holdToken, msg.sender, holdTokenOut);
```
And this variable is only set to true when all loans are removed from the borrower's `loansInfo` in `_calculateEmergencyLoanClosure()`, but this will never happen if a lender has burned the UniV3 NFT due to the `msg.sender` check against the NFT owner:
```solidity
    function _calculateEmergencyLoanClosure(
        bool zeroForSaleToken,
        bytes32 borrowingKey,
        uint256 totalfeesOwed,
        uint256 totalBorrowedAmount
    ) private returns (uint256 removedAmt, uint256 feesAmt, bool completeRepayment) {
        // Create a memory struct to store liquidity cache information.
        NftPositionCache memory cache;
        // Get the array of LoanInfo structs associated with the given borrowing key.
        LoanInfo[] storage loans = loansInfo[borrowingKey];
        // Iterate through each loan in the loans array.
        for (uint256 i; i < loans.length; ) {
            LoanInfo memory loan = loans[i];
            // Get the owner address of the loan's token ID using the underlyingPositionManager contract.
            address creditor = _getOwnerOf(loan.tokenId);
            // Check if the owner of the loan's token ID is equal to the `msg.sender`.
            if (creditor == msg.sender) {
                // If the owner matches the `msg.sender`, replace the current loan with the last loan in the loans array
                // and remove the last element.
                loans[i] = loans[loans.length - 1];
                loans.pop();
                // Remove the borrowing key from the tokenIdToBorrowingKeys mapping.
                tokenIdToBorrowingKeys[loan.tokenId].remove(borrowingKey);
                // Update the liquidity cache based on the loan information.
                _upNftPositionCache(zeroForSaleToken, loan, cache);
                // Add the holdTokenDebt value to the removedAmt.
                removedAmt += cache.holdTokenDebt;
                // Calculate the fees amount based on the total fees owed and holdTokenDebt.
                feesAmt += FullMath.mulDiv(totalfeesOwed, cache.holdTokenDebt, totalBorrowedAmount);
            } else {
                // If the owner of the loan's token ID is not equal to the `msg.sender`,
                // the function increments the loop counter and moves on to the next loan.
                unchecked {
                    ++i;
                }
            }
        }
        // Check if all loans have been removed, indicating complete repayment.
        completeRepayment = loans.length == 0;
    }
```
Therefore, if a borrower's position is only emergency-liquidatable then the `liquidationBonus` can be lost forever.
## Impact
Lenders won't receive the `liquidationBonus` and it may be forever unclaimable.
## Code Snippet
https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L656-L659
https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L784-L823
## Tool used

Manual Review

## Recommendation
Perform a check to see if the only loans left are from burned NFTs and set `completeRepayment` to `true` if so.

Also want to mention that there's a similar issue that can be fixed in a similar way
