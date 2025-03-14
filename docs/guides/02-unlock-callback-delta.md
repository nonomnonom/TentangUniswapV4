# Panduan Unlock Callback & Delta

[← Panduan Custom Hooks](./01-custom-hooks.md) | [→ Panduan State Pool](./03-reading-pool-state.md)

Unlock callback adalah konsep dasar yang penting dalam Uniswap v4. Ini adalah mekanisme yang memungkinkan kontrak untuk berinteraksi dengan PoolManager.

## Cara Kerja

```solidity
// 1. Buat kontrak yang mewarisi SafeCallback
import {SafeCallback} from "v4-periphery/src/base/SafeCallback.sol";

contract IntegratingContract is SafeCallback {
    constructor(IPoolManager _poolManager) SafeCallback(_poolManager) {}
}

// 2. Implementasi unlock callback
function _unlockCallback(bytes calldata data) internal override returns (bytes memory) {
    (...) = abi.decode(data, (...));
}

// 3. Panggil fungsi unlock
bytes memory unlockData = abi.encode(encode_operations_here);
bytes memory unlockResultData = poolManager.unlock(unlockData);
```

## Operasi yang Tersedia

### A. Operasi Liquidity-accessing (Menghasilkan delta):
- modify liquidity: Menambah/kurang likuiditas
- swap: Trading token
- donate: Memberikan token revenue ke posisi dalam range

### B. Operasi Delta-resolving (Menyelesaikan delta):
- settle: Menyelesaikan delta negatif setelah transfer token
- take: Transfer token dari manager untuk delta positif
- mint: Membuat klaim ERC6909
- burn: Menghapus klaim ERC6909
- clear: Menghapus delta positif yang kecil

## Catatan Penting
- Delta negatif: PoolManager perlu menerima token
- Delta positif: PoolManager perlu membayar token
- Semua delta harus diselesaikan sebelum mengakhiri operasi 