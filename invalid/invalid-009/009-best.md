Suave Tan Otter

medium

# `ApproveSwapAndPay::_tryApprove` does not approve to zero first

## Summary
Some ERC20 tokens (i.e., USDT) require setting approval to 0 first before being able to reset it to another value. 

## Vulnerability Detail
The `ApproveSwapAndPay::_tryApprove` function attempts to approve a specific amount of tokens for a spender. It is called in the `ApproveSwapAndPay::_maxApproveIfNecessary` function to approve allowances to the max value. However, it does not reset the approval to 0 first, which for some ERC20 tokens can cause issues.

## Impact
Depending on the token, it may revert the transaction or simply not return any value.

## Code Snippet
https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L66-L71

https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L82-L95

## Tool used
Manual Review

## Recommendation
It is recommended to set the allowance to zero before increasing the allowance and use safeApprove/safeIncreaseAllowance.