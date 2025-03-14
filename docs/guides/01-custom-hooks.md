# Panduan Pengembangan Custom Hooks

Panduan ini akan membantu Anda memahami dan mengimplementasikan custom hooks di Uniswap v4.

## 📋 Daftar Isi
- [Memahami Hooks](#memahami-hooks)
- [Setup Project](#setup-project)
- [Implementasi Dasar](#implementasi-dasar)
- [Deployment](#deployment)
- [Testing](#testing)

## Memahami Hooks

Hooks adalah kontrak yang memungkinkan Anda menambahkan logika kustom pada berbagai titik dalam lifecycle pool Uniswap v4.

### Lifecycle Hooks yang Tersedia:

1. **Initialize Hooks**
   - `beforeInitialize`
   - `afterInitialize`

2. **Liquidity Hooks**
   - `beforeAddLiquidity`
   - `afterAddLiquidity`
   - `beforeRemoveLiquidity`
   - `afterRemoveLiquidity`

3. **Swap Hooks**
   - `beforeSwap`
   - `afterSwap`

4. **Donate Hooks**
   - `beforeDonate`
   - `afterDonate`

## Setup Project

### 1. Inisialisasi Project Foundry

```bash
forge init my-uniswap-v4-hook
cd my-uniswap-v4-hook
```

### 2. Install Dependencies

```bash
forge install uniswap/v4-core
forge install uniswap/v4-periphery
```

### 3. Setup remappings.txt

```text
@uniswap/v4-core/=lib/v4-core/
@uniswap/v4-periphery/=lib/v4-periphery/
```

## Implementasi Dasar

### 1. Kontrak Hook Dasar

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import {BaseHook} from "@uniswap/v4-periphery/src/BaseHook.sol";
import {Hooks} from "@uniswap/v4-core/src/libraries/Hooks.sol";
import {IPoolManager} from "@uniswap/v4-core/src/interfaces/IPoolManager.sol";
import {PoolKey} from "@uniswap/v4-core/src/types/PoolKey.sol";
import {PoolId, PoolIdLibrary} from "@uniswap/v4-core/src/types/PoolId.sol";

contract CustomHook is BaseHook {
    using PoolIdLibrary for PoolKey;

    constructor(IPoolManager _poolManager) BaseHook(_poolManager) {}

    function getHookPermissions() public pure override returns (Hooks.Permissions memory) {
        return Hooks.Permissions({
            beforeInitialize: false,
            afterInitialize: false,
            beforeAddLiquidity: false,
            beforeRemoveLiquidity: false,
            afterAddLiquidity: false,
            afterRemoveLiquidity: false,
            beforeSwap: true,
            afterSwap: true,
            beforeDonate: false,
            afterDonate: false,
            beforeSwapReturnDelta: false,
            afterSwapReturnDelta: false,
            afterAddLiquidityReturnDelta: false,
            afterRemoveLiquidityReturnDelta: false
        });
    }

    function beforeSwap(
        address sender,
        PoolKey calldata key,
        IPoolManager.SwapParams calldata params,
        bytes calldata hookData
    ) external override returns (bytes4) {
        // Implementasi logika beforeSwap
        return BaseHook.beforeSwap.selector;
    }

    function afterSwap(
        address sender,
        PoolKey calldata key,
        IPoolManager.SwapParams calldata params,
        BalanceDelta delta,
        bytes calldata hookData
    ) external override returns (bytes4) {
        // Implementasi logika afterSwap
        return BaseHook.afterSwap.selector;
    }
}
```

### 2. Contoh Hook untuk Points System

```solidity
contract PointsHook is BaseHook {
    // Points token untuk rewards
    ERC20 public immutable pointsToken;
    
    // Mapping untuk tracking points
    mapping(address => uint256) public userPoints;
    
    constructor(IPoolManager _poolManager) BaseHook(_poolManager) {
        pointsToken = new ERC20("Trading Points", "PTS");
    }
    
    function afterSwap(
        address sender,
        PoolKey calldata key,
        IPoolManager.SwapParams calldata params,
        BalanceDelta delta,
        bytes calldata
    ) external override returns (bytes4) {
        // Award points berdasarkan volume trading
        uint256 volume = params.amountSpecified < 0 
            ? uint256(-params.amountSpecified)
            : uint256(params.amountSpecified);
            
        // 1 point per 1 ETH volume
        uint256 points = volume / 1e18;
        userPoints[sender] += points;
        pointsToken.mint(sender, points);
        
        return BaseHook.afterSwap.selector;
    }
}
```

## Deployment

### 1. Setup Script Deployment

```solidity
// script/DeployHook.s.sol
import "forge-std/Script.sol";
import {Hooks} from "@uniswap/v4-core/src/libraries/Hooks.sol";
import {PoolManager} from "@uniswap/v4-core/src/PoolManager.sol";
import {HookMiner} from "@uniswap/v4-periphery/src/utils/HookMiner.sol";

contract DeployHook is Script {
    function run() public {
        // Define flags berdasarkan hooks yang digunakan
        uint160 flags = uint160(
            Hooks.BEFORE_SWAP_FLAG | 
            Hooks.AFTER_SWAP_FLAG
        );
        
        // Mine address yang valid
        (address hookAddress, bytes32 salt) = HookMiner.find(
            CREATE2_DEPLOYER,
            flags,
            type(CustomHook).creationCode,
            abi.encode(POOL_MANAGER)
        );
        
        // Deploy hook
        vm.broadcast();
        CustomHook hook = new CustomHook{salt: salt}(
            IPoolManager(POOL_MANAGER)
        );
        
        require(
            address(hook) == hookAddress,
            "Hook deployment failed"
        );
    }
}
```

### 2. Deploy Hook

```bash
forge script script/DeployHook.s.sol --rpc-url $RPC_URL --private-key $PRIVATE_KEY
```

## Testing

### 1. Setup Test Environment

```solidity
// test/CustomHook.t.sol
import "forge-std/Test.sol";
import {CustomHook} from "../src/CustomHook.sol";
import {PoolManager} from "@uniswap/v4-core/src/PoolManager.sol";

contract CustomHookTest is Test {
    CustomHook hook;
    PoolManager manager;
    
    function setUp() public {
        // Setup test environment
        manager = new PoolManager(500000);
        hook = new CustomHook(manager);
    }
    
    function testBeforeSwap() public {
        // Test implementasi
    }
    
    function testAfterSwap() public {
        // Test implementasi
    }
}
```

### 2. Run Tests

```bash
forge test
```

## Best Practices

1. **Keamanan**
   - Validasi input dengan ketat
   - Implementasi access control
   - Hindari reentrancy
   - Audit code sebelum deployment

2. **Gas Optimization**
   - Minimalisir storage writes
   - Batasi kompleksitas komputasi
   - Gunakan calldata untuk parameters
   - Cache frequently accessed values

3. **Maintainability**
   - Dokumentasi yang jelas
   - Modular design
   - Unit tests yang komprehensif
   - Upgrade path yang jelas

## Troubleshooting

### Common Issues

1. **Invalid Hook Address**
   - Pastikan flags sesuai dengan implementasi
   - Gunakan HookMiner untuk generate valid address
   - Verifikasi salt dan deployment parameters

2. **Gas Issues**
   - Optimasi storage operations
   - Batasi kompleksitas logika
   - Monitor gas usage dalam tests

3. **Integration Issues**
   - Test di local environment
   - Verifikasi interface implementations
   - Check permissions settings

## Resources

- [Uniswap v4 Core Documentation](https://docs.uniswap.org/v4)
- [Hook Examples Repository](https://github.com/Uniswap/v4-periphery/tree/main/src/hooks)
- [Technical Discussions](https://github.com/Uniswap/v4-core/discussions)
- [Security Considerations](https://github.com/Uniswap/v4-core/blob/main/docs/SECURITY.md) 