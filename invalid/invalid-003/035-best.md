Huge Teal Octopus

medium

# Protocol will be bricked on zkSync due to `computePoolAddress()` calculation

## Summary
`computePoolAddress()` bricks the protocol on zkSync.
## Vulnerability Detail
`computePoolAddress()` works for all EVM-compatible L2s, but zkSync is not really EVM-compatible. It uses a different method to calculate contract addresses (https://docs.zksync.io/build/developer-reference/differences-with-ethereum.html#create-create2). 
## Impact
Protocol won't function on zkSync.
## Code Snippet
https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L251
## Tool used

Manual Review

## Recommendation
Refactor `computePoolAddress()` to be compatible with zkSync or create a different contract for zkSync.