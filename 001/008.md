Suave Tan Otter

medium

# ApproveSwapAndPay.sol swap functions lack slippage protection, leading to loss of user funds

## Summary
The `ApproveSwapAndPay::_callExternalSwap`,  `ApproveSwapAndPay::_v3SwapExactInput`, and `ApproveSwapAndPay::uniswapV3SwapCallback`  functions lacks slippage protection when performing swaps, which can lead to frontrun, stealing funds from users.

## Vulnerability Detail
The `_callExternalSwap` function performs a series of external swap calls by iterating through an array of `SwapParams`, executing each corresponding swap. 

The `_v3SwapExactInput` function performs a token swap using Uniswap V3 with exact input.

The `uniswapV3SwapCallback` function is called when a swap is executed on a Uniswap V3 pool. 

None of these functions enforce slippage protection and address the risk of unfavorable price movements during the execution of a trade. A malicious MEV bot could watch for these transactions in the mempool. When it sees such a transaction, it could perform a "sandwich attack", trading massively in the same direction as the trade in advance of it to push the price out of whack, and then trading back after the user's transaction, so that they end up pocketing a profit at users expense.

## Impact
User losing their funds by receiving very little of their borrowed token back from the swap.

## Code Snippet
https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L135-L158

https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L186-L206

https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L222-L238

## Tool used
Manual Review

## Recommendation
Enforce slippage parameters to ensure that the amount they received back from Uniswap is in line with what to expect.