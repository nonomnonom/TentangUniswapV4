# Panduan Membaca State Pool

[← Panduan Unlock Callback](./02-unlock-callback-delta.md) | [→ Panduan Universal Router](./04-universal-router.md)

Uniswap v4 menggunakan pendekatan baru untuk menyimpan dan mengakses data pool melalui StateLibrary dan extsload.

## Setup Dasar

```solidity
// 1. Import yang diperlukan
import {IPoolManager} from "v4-core/interfaces/IPoolManager.sol";
import {PoolKey} from "v4-core/types/PoolKey.sol";
import {PoolId, PoolIdLibrary} from "v4-core/types/PoolId.sol";
import {StateLibrary} from "v4-core/libraries/StateLibrary.sol";

// 2. Buat kontrak reader
contract PoolStateReader {
    using PoolIdLibrary for PoolKey;
    using StateLibrary for IPoolManager;

    IPoolManager public immutable poolManager;

    constructor(IPoolManager _poolManager) {
        poolManager = _poolManager;
    }
}
```

## Fungsi-fungsi Pembacaan State

### 1. getPoolState
```solidity
function getPoolState(PoolKey calldata key) external view returns (
    uint160 sqrtPriceX96,
    int24 tick,
    uint24 protocolFee,
    uint24 lpFee
) {
    return poolManager.getSlot0(key.toId());
}
```
- Membaca harga, tick, dan fee settings
- Berguna untuk: oracle harga, bot trading, manajemen likuiditas

### 2. getPoolLiquidity
```solidity
function getPoolLiquidity(PoolKey calldata key) external view returns (uint128 liquidity) {
    return poolManager.getLiquidity(key.toId());
}
```
- Membaca total likuiditas dalam pool
- Berguna untuk: analisis slippage, monitoring kedalaman pasar

### 3. getPositionInfo
```solidity
function getPositionInfo(
    PoolKey calldata key,
    address owner,
    int24 tickLower,
    int24 tickUpper,
    bytes32 salt
) external view returns (
    uint128 liquidity,
    uint256 feeGrowthInside0LastX128,
    uint256 feeGrowthInside1LastX128
) {
    return poolManager.getPositionInfo(key.toId(), owner, tickLower, tickUpper, bytes32(salt));
}
```
- Membaca informasi posisi likuiditas spesifik
- Berguna untuk: dashboard manajemen likuiditas, sistem rebalancing otomatis

## Catatan Penting
- Menggunakan StateLibrary lebih efisien dari segi gas
- Menggunakan extsload untuk membaca storage slots langsung
- Transient storage (via exttload) untuk data sementara dalam transaksi 