Brief Sandstone Crocodile

medium

# Usage of ``slot0`` is extremely volatile

## Summary
The protocol utilizes Uniswap's ``slot0`` function to get an exact price at the given timestamp, creating room for manipulation error.

## Vulnerability Detail
As  it can be seen: https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L335-L346
We utilize the spot price of the asset pair, instead of using a time-weighted method to determine a more accurate result.
As though it can be seen as a design decision, I (and probably many others) still believe this issue to be valid, due to the price volatility it opens up for the swap.

## Impact
Medium, as it is a straightforward issue.

## Code Snippet
https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/LightQuoterV3.sol#L203-L223
https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L335-L346

## Tool used

Manual Review

## Recommendation
Use Uniswap's TWAP methods
