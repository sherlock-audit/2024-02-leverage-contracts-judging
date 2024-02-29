Suave Tan Otter

high

# Increase Liquidity and Decrease Liquidity have incorrect deadline check and lack slippage control

## Summary
The `LiquidityManager::_increaseLiquidity` and `LiquidityManager::_decreaseLiquidity` functions lack slippage control and have incorrect implementation of the `deadline` parameter.

## Vulnerability Detail
The `_increaseLiquidity` function increases the liquidity of a position by providing additional tokens and the `_decreaseLiquidity` decreases the liquidity of a position by removing tokens. Both functions consist of parameters that can control slippage and create a deadline for the transaction. These parameters ensure that the transaction cannot be manipulated through front-running. The parameters are `amount0Min`, `amount1Min`, and `deadline`. 

The problem is that `amount0Min`, `amount1Min` are always set to 0, which means there is no slippage control, allowing for the possibility of a sandwich attack. In addition, `deadline` is always set to `block.timestamp`, which defeats the entire purpose of the `deadline`, as it is used to compare if it has passed `block.timestamp` by checking if `block.timestamp <= deadline`. Since the deadline = `block.timestamp` in this case, the check will always pass, making it susceptible to MEV attacks.

## Impact
Users receiving less value due to manipulation of market from lack of slippage control (`amount0Min`, `amount1Min` are always 0) and funds being stolen due to transactions being executed in a way that is far less favorable to the users, due to `deadline = block.timestamp`.

## Code Snippet
https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L348-L423

## Tool used
Manual Review

## Recommendation
Implement slippage control for `amount0Min` and `amount1Min`. Set `deadline` to a future timestamp.
