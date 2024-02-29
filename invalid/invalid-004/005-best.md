Brief Sandstone Crocodile

medium

# ApproveSwapAndPay#_tryApprove() - USDT token approval would not work on mainnet

## Summary
The ``_tryApprove`` function is an internal method, used in ``_maxApproveIfNecessary``, in order to approve an external swap target to use the user's tokens. The protocol has done a good job at working around USDT's edge-case of not allowing non-zero approvals when there is existing allowance, by approving to 0 on fail and then reattempting the approve call.
The problem is that USDT's mainnet implementation does not fully comply with the ERC20 standard.

## Vulnerability Detail
Apart from the fact that USDT accepts only zero approvals when there is existing allowance, on the ETH mainnet it also does not return a bool value when ``approve`` is called directly (it reverts). Thus meaning that any attempt to execute swaps at targets would fail for USDT on the mainnet, breaking core functionality.
Also keep in mind that there are also other tokens with such false-returning or no-return-value functionality on their approve methods, but since USDT is the most widely used, it is important to be noted first.

You can read more here:
https://forum.openzeppelin.com/t/can-not-call-the-function-approve-of-the-usdt-contract/2130
https://ethereum.stackexchange.com/questions/151297/approve-usdt-on-ethereum

## Impact
Medium, since USDT is practically unusable on Ethereum, many token pair pools would be affected due to the high popularity of USDT and the mainnet. USDT is not the only token with such incompliancy to the standard.

## Code Snippet
https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L66-L71
https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L82-L95
https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L149

## Tool used

Manual Review

## Recommendation
Use `SafeERC20`'s `forceApprove` method by OpenZeppelin.

