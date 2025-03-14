# Panduan Position Manager

[← Panduan ERC-6909](./05-erc6909.md) | [→ Kembali ke Index](./00-index.md)

Position Manager adalah komponen untuk mengelola posisi likuiditas di Uniswap v4 dengan sistem berbasis command.

## Konsep Dasar

### 1. Command-Based Design
```solidity
// Contoh: Minting posisi baru membutuhkan 2 commands
bytes memory actions = abi.encodePacked(
    Actions.MINT_POSITION,  // Buat posisi
    Actions.SETTLE_PAIR     // Sediakan token
);
```

### 2. Tipe-tipe Action:

```solidity
// Liquidity Operations
uint256 constant MINT_POSITION = 0x02;       // Buat posisi baru (-delta)
uint256 constant INCREASE_LIQUIDITY = 0x00;  // Tambah likuiditas (-delta)
uint256 constant DECREASE_LIQUIDITY = 0x01;  // Kurangi likuiditas (+delta)
uint256 constant BURN_6909 = 0x18;          // Burn token (+delta)

// Delta-Resolving Operations
uint256 constant SETTLE_PAIR = 0x0d;     // Bayar 2 token ke pool
uint256 constant TAKE_PAIR = 0x11;       // Terima 2 token dari pool
uint256 constant CLOSE_CURRENCY = 0x12;  // Handle dua arah delta
uint256 constant CLEAR_OR_TAKE = 0x13;   // Take jika worth it
```

## Implementasi Minting Posisi

```solidity
function mintNewPosition(
    PoolKey calldata poolKey,
    int24 tickLower,
    int24 tickUpper,
    uint256 liquidity,
    uint128 amount0Max,
    uint128 amount1Max,
    address recipient
) external returns (uint256 tokenId) {
    // 1. Define actions
    bytes memory actions = abi.encodePacked(
        Actions.MINT_POSITION,
        Actions.SETTLE_PAIR
    );

    // 2. Setup parameters
    bytes[] memory params = new bytes[](2);
    
    // Parameters untuk MINT_POSITION
    params[0] = abi.encode(
        poolKey,     // Pool target
        tickLower,   // Batas bawah harga
        tickUpper,   // Batas atas harga
        liquidity,   // Jumlah likuiditas
        amount0Max,  // Max token0
        amount1Max,  // Max token1
        recipient,   // Penerima NFT
        ""          // Hook data
    );

    // Parameters untuk SETTLE_PAIR
    params[1] = abi.encode(
        poolKey.currency0,
        poolKey.currency1
    );

    // 3. Execute mint
    positionManager.modifyLiquidities(
        abi.encode(actions, params),
        block.timestamp + 60  // 60 detik deadline
    );
}
```

## Best Practices

### 1. Grouping Operations
```solidity
// Efisien: Group operasi dengan delta sejenis
bytes memory actions = abi.encodePacked(
    Actions.MINT_POSITION,      // Delta1: -100 token
    Actions.INCREASE_LIQUIDITY, // Delta2: -50 token
    Actions.SETTLE_PAIR        // Resolve total: -150 token
);

// Kurang efisien: Resolve delta berkali-kali
bytes memory actions = abi.encodePacked(
    Actions.MINT_POSITION,      // Delta: -100 token
    Actions.SETTLE_PAIR,        // Resolve: -100 token
    Actions.INCREASE_LIQUIDITY, // Delta baru: -50 token
    Actions.SETTLE_PAIR        // Resolve lagi: -50 token
);
```

### 2. Urutan Operasi
- Group operasi yang menghasilkan delta sejenis
- Resolve semua delta di akhir jika memungkinkan
- Gunakan CLOSE_CURRENCY jika final delta tidak bisa diprediksi

### 3. Fee Handling
- Fee otomatis dikreditkan saat increase liquidity
- Fee bisa digunakan untuk cover token yang dibutuhkan
- Pertimbangkan fee accumulation dalam kalkulasi