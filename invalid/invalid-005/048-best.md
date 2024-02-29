Mammoth Lipstick Whale

medium

# Low Liquidity in Uniswap V3 Pool Can Lead to ETH Lockup in _v3SwapExactInput Contract

## Summary
The ApproveSwapAndPay contract employs Uniswap V3 to perform ETH-to-project token swaps. When the terminal invokes the _v3SwapExactInput function, it provides the amount of ETH to be swapped for project tokens. The swap operation sets sqrtPriceLimitX96 to the lowest possible price, and the slippage is checked at the callback.

However, if the Uniswap V3 pool lacks sufficient liquidity or being manipulated before the transaction is executed, the swap will halt once the pool's price reaches the sqrtPriceLimitX96 value. Consequently, not all the ETH sent to the contract will be utilized, resulting in the remaining ETH becoming permanently locked within the contract.
## Vulnerability Detail
The _swap() function interacts with the Uniswap V3 pool. It sets sqrtPriceLimitX96 to the minimum or maximum feasible value to ensure that the swap attempts to utilize all available liquidity in the pool.

    function _v3SwapExactInput(
        v3SwapExactInputParams memory params
    ) internal returns (uint256 amountOut) {
        // fee must be non-zero
        (params.fee == 0).revertError(ErrLib.ErrorCode.INTERNAL_SWAP_POOL_REQUIRED);
        // Determine if tokenIn has a 0th token
        bool zeroForTokenIn = params.tokenIn < params.tokenOut;
        // Compute the address of the Uniswap V3 pool based on tokenIn, tokenOut, and fee
        // Call the swap function on the Uniswap V3 pool contract
        (int256 amount0Delta, int256 amount1Delta) = IUniswapV3Pool(
            computePoolAddress(params.tokenIn, params.tokenOut, params.fee)
        ).swap(
                address(this), //recipient
                zeroForTokenIn,
                params.amountIn.toInt256(),
                zeroForTokenIn ? MIN_SQRT_RATIO + 1 : MAX_SQRT_RATIO - 1,
                abi.encode(params.fee, params.tokenIn, params.tokenOut)
            );
        // Calculate the actual amount of output tokens received
        amountOut = uint256(-(zeroForTokenIn ? amount1Delta : amount0Delta));
    }


Finally, the uniswapV3SwapCallback() function uses the input from the pool callback to wrap ETH and transfer WETH to the pool. So, if _amountToSend < msg.value, the unused ETH is locked in the contract.

    function uniswapV3SwapCallback(
        int256 amount0Delta,
        int256 amount1Delta,
        bytes calldata data
    ) external {
        (amount0Delta <= 0 && amount1Delta <= 0).revertError(ErrLib.ErrorCode.INVALID_SWAP); // swaps entirely within 0-liquidity regions are not supported

        (uint24 fee, address tokenIn, address tokenOut) = abi.decode(
            data,
            (uint24, address, address)
        );
        (computePoolAddress(tokenIn, tokenOut, fee) != msg.sender).revertError(
            ErrLib.ErrorCode.INVALID_CALLER
        );
        uint256 amountToPay = amount0Delta > 0 ? uint256(amount0Delta) : uint256(amount1Delta);
        _pay(tokenIn, address(this), msg.sender, amountToPay);
    }
## Impact

## Code Snippet
https://github.com/sherlock-audit/2024-02-leverage-contracts/blob/main/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L197
## Tool used

Manual Review

## Recommendation
Consider returning the amount of unused ETH to the beneficiary.

