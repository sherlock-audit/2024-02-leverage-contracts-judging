Soft Inky Mongoose

medium

# Out of Gas Scenario in the `_addKeysAndLoansInfo` function due to repeated loop can lead to DOS

## Summary
The functions mentioned above has loop, unnecessary operations which can lead to DOS are performed repeatedly.
## Vulnerability Detail
`LiquidityBorrowingManager.sol#_addKeysAndLoansInfo` function used to add borrowing keys and loan information for a specific borrowing key in borrow function.

https://github.com/RealWagmi/wagmi-leverage/blob/main/contracts/LiquidityBorrowingManager.sol#L419

As you could see the functions via the Code Snippet, all of the actions happen in the loops of the functions and cost gas, it may revert due to exceeding the block size gas limit according to length of `LoanInfo[]`.

```solidity

/**
     * @dev This internal function is used to add borrowing keys and loan information for a specific borrowing key.
     * @param borrowingKey The borrowing key to be added or updated.
     * @param sourceLoans An array of LoanInfo structs representing the loans to be associated with the borrowing key.
     */
    function _addKeysAndLoansInfo(
        bytes32 borrowingKey,
        LoanInfo[] memory sourceLoans
    ) private returns (uint256 pushCounter) {
        // Get the storage reference to the loans array for the borrowing key
        LoanInfo[] storage loans = loansInfo[borrowingKey];
        // Iterate through the sourceLoans array
        for (uint256 i; i < sourceLoans.length; ) {
            // Get the current loan from the sourceLoans array
            LoanInfo memory loan = sourceLoans[i];
            // Get the storage reference to the tokenIdLoansKeys array for the loan's token ID
            if (tokenIdToBorrowingKeys[loan.tokenId].add(borrowingKey)) {
                // Push the current loan to the loans array
                loans.push(loan);
                pushCounter++;
            } else {
                // If already exists, find the loan and update its liquidity
                for (uint256 j; j < loans.length; ) {
875                 if (loans[j].tokenId == loan.tokenId) {
876                     loans[j].liquidity += loan.liquidity;
877                     break;
878                 }
879                 unchecked {
880                     ++j;
881                 }
                }
            }
            unchecked {
                ++i;
            }
        }
        // Ensure that the number of loans does not exceed the maximum limit
        (loans.length > Constants.MAX_NUM_LOANS_PER_POSITION).revertError(
            ErrLib.ErrorCode.TOO_MANY_LOANS_PER_POSITION
        );
        // Add the borrowing key to the userBorrowingKeys mapping for the borrower if it does not exist
        userBorrowingKeys[msg.sender].add(borrowingKey);
    }


```

In this function, if already exists current loan, perform #L875-#L881 repeatedly until find loan that has same token Id with current loan.
Therefore it may revert due to exceeding the block size gas limit.
## Impact
The execution fails due to exceeding the block size gas limit.
## Code Snippet
https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L857C4-L894C6
## Tool used

Manual Review

## Recommendation
Modify the `LiquidityBorrowingManager.sol#_addKeysAndLoansInfo` function as follows.

```solidity

+   mapping(bytes32 => mapping(uint256 => LoanInfo)) tokenIdToLoans;
+   mapping(bytes32 => uint256) numLoans;

function _addKeysAndLoansInfo(
        bytes32 borrowingKey,
        LoanInfo[] memory sourceLoans
    ) private returns (uint256 pushCounter) {
        // Get the storage reference to the loans array for the borrowing key
        LoanInfo[] storage loans = loansInfo[borrowingKey];
        // Iterate through the sourceLoans array
        for (uint256 i; i < sourceLoans.length; ) {
            // Get the current loan from the sourceLoans array
            LoanInfo memory loan = sourceLoans[i];
            // Get the storage reference to the tokenIdLoansKeys array for the loan's token ID
            if (tokenIdToBorrowingKeys[loan.tokenId].add(borrowingKey)) {
                // Push the current loan to the loans array
--              loans.push(loan);

++              tokenIdToLoans[borrowingKey][loan.tokenId]=loan;
++              numLoans[borrowingKey]++;

                pushCounter++;
            } else {
                // If already exists, find the loan and update its liquidity
--               for (uint256 j; j < loans.length; ) {
--                  if (loans[j].tokenId == loan.tokenId) {
--                      loans[j].liquidity += loan.liquidity;
--                      break;
--                  }
--                  unchecked {
--                      ++j;
--                  }
--              }

++              tokenIdLoansKeys[borrowingKey][loan.tokenId] += loan.liquidity;

            }
            unchecked {
                ++i;
            }
        }
        // Ensure that the number of loans does not exceed the maximum limit
--      (loans.length > Constants.MAX_NUM_LOANS_PER_POSITION).revertError(
--          ErrLib.ErrorCode.TOO_MANY_LOANS_PER_POSITION
--      );

++      (numLoans[borrowingKey] > Constants.MAX_NUM_LOANS_PER_POSITION).revertError(
++          ErrLib.ErrorCode.TOO_MANY_LOANS_PER_POSITION
++      );

        // Add the borrowing key to the userBorrowingKeys mapping for the borrower if it does not exist
        userBorrowingKeys[msg.sender].add(borrowingKey);
    }

```