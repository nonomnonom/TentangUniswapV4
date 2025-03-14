# Panduan Universal Router

[← Panduan State Pool](./03-reading-pool-state.md) | [→ Panduan ERC-6909](./05-erc6909.md)

Universal Router adalah kontrak yang memudahkan eksekusi swap di Uniswap v4. Berikut langkah-langkah implementasinya.

## Setup Project

```bash
# Install dependencies
forge install uniswap/v4-core
forge install uniswap/v4-periphery
forge install uniswap/permit2
forge install uniswap/universal-router
forge install OpenZeppelin/openzeppelin-contracts

# Tambahkan remapping
@uniswap/v4-core/=lib/v4-core/
@uniswap/v4-periphery/=lib/v4-periphery/
@uniswap/permit2/=lib/permit2/
@uniswap/universal-router/=lib/universal-router/
@openzeppelin/contracts/=lib/openzeppelin-contracts/
```

## Implementasi Kontrak

```solidity
// 1. Setup kontrak dasar
contract Example {
    using StateLibrary for IPoolManager;

    UniversalRouter public immutable router;
    IPoolManager public immutable poolManager;
    IPermit2 public immutable permit2;

    constructor(address _router, address _poolManager, address _permit2) {
        router = UniversalRouter(_router);
        poolManager = IPoolManager(_poolManager);
        permit2 = IPermit2(_permit2);
    }

    // 2. Fungsi approval token
    function approveTokenWithPermit2(
        address token,
        uint160 amount,
        uint48 expiration
    ) external {
        IERC20(token).approve(address(permit2), type(uint256).max);
        permit2.approve(token, address(router), amount, expiration);
    }

    // 3. Fungsi swap
    function swapExactInputSingle(
        PoolKey calldata key,
        uint128 amountIn,
        uint128 minAmountOut,
        uint256 deadline
    ) external returns (uint256 amountOut) {
        // Encode command
        bytes memory commands = abi.encodePacked(uint8(Commands.V4_SWAP));

        // Encode actions
        bytes memory actions = abi.encodePacked(
            uint8(Actions.SWAP_EXACT_IN_SINGLE),
            uint8(Actions.SETTLE_ALL),
            uint8(Actions.TAKE_ALL)
        );

        // Encode parameters
        bytes[] memory params = new bytes[](3);
        params[0] = abi.encode(
            IV4Router.ExactInputSingleParams({
                poolKey: key,
                zeroForOne: true,
                amountIn: amountIn,
                amountOutMinimum: minAmountOut,
                sqrtPriceLimitX96: uint160(0),
                hookData: bytes("")
            })
        );
        params[1] = abi.encode(key.currency0, amountIn);
        params[2] = abi.encode(key.currency1, minAmountOut);

        // Execute swap via router
        router.execute(commands, params, deadline);
    }
}
```

## Catatan Penting
- Universal Router menggunakan sistem encoding khusus untuk commands dan inputs
- Permit2 digunakan untuk manajemen approval token yang lebih aman
- Ada 2 jenis swap: Exact Input dan Exact Output
- Sequence actions penting: SWAP -> SETTLE -> TAKE
- Gunakan deadline untuk proteksi dari MEV attacks 