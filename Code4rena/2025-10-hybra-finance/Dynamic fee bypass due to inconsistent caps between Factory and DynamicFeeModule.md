**Official Finding:** [Code4rena Submission](https://code4rena.com/audits/2025-10-hybra-finance/submissions?uid=x3QP4YDsxso)

## Finding Description and Impact

The `CLFactory.getSwapFee()` function silently rejects fees above 10% (100,000 basis points) and falls back to the base `tickSpacingToFee`, creating a mismatch with `DynamicSwapFeeModule` which is designed to support fees up to 50% for anti-MEV protection.

```solidity
function getSwapFee(address pool) external view override returns (uint24) {
    if (swapFeeModule != address(0)) {
        (bool success, bytes memory data) = swapFeeModule.excessivelySafeStaticCall(
            200_000, 32, abi.encodeWithSelector(IFeeModule.getFee.selector, pool)
        );
        if (success) {
            uint24 fee = abi.decode(data, (uint24));
            if (fee <= 100_000) {  // Silently rejects fees > 10%
                return fee;
            }
        }
    }
    return tickSpacingToFee[CLPool(pool).tickSpacing()];  // Falls back to 0.05%-1%
}
```

When governance configures fees above 10% through `setCustomFee()`, the transaction succeeds, the module correctly returns the configured fee, but the factory ignores it without reverting or emitting any event.

The module's constants explicitly declare support for 50% fees:

```solidity
uint256 public constant MAX_BASE_FEE = 500_000; // 50% - for launch anti-MEV protection
uint256 public constant MAX_FEE_CAP = 500_000; // 50% - for launch anti-MEV protection
```

The comment "for launch anti-MEV protection" indicates the intended use case: during volatile token launches or extreme market conditions, protocols need 20-30%+ fees to make MEV sandwich attacks unprofitable. When the factory silently reduces these to 0.05%, MEV bots can profitably exploit traders while LPs lose 99%+ of intended fee revenue.

This is demonstrably a bug rather than intended design for three reasons:

1. The factory's `getUnstakedFee()` accepts fees up to 100% using the same pattern, proving the 10% cap is not a universal safety policy
2. `setCustomFee()`, `setFeeCap()`, and `setDefaultFeeCap()` all validate against 50%, allowing configuration that the factory then ignores
3. If the 10% cap was intentional, the module would enforce it with a revert rather than allowing invalid configurations

For a pool with $10M trading volume during a volatile launch where governance set 30% fees:

- Expected LP revenue: $3,000,000 (30% of $10M)
- Actual LP revenue: $5,000 (0.05% of $10M)
- Lost revenue: $2,995,000 (99.83% reduction)

Additionally, MEV bots can read the on-chain code to know they'll only pay 0.05%, allowing them to profitably execute attacks that would have been unprofitable at the intended 30% fee rate. This defeats the entire anti-MEV protection mechanism that the module was designed to provide.

## Recommended Mitigation Steps

Align the factory's acceptance threshold with the module's designed maximum:

```solidity
function getSwapFee(address pool) external view override returns (uint24) {
    if (swapFeeModule != address(0)) {
        (bool success, bytes memory data) = swapFeeModule.excessivelySafeStaticCall(
            200_000, 32, abi.encodeWithSelector(IFeeModule.getFee.selector, pool)
        );
        if (success) {
            uint24 fee = abi.decode(data, (uint24));
            if (fee <= 500_000) {  // Match MAX_FEE_CAP from DynamicSwapFeeModule
                return fee;
            }
        }
    }
    return tickSpacingToFee[CLPool(pool).tickSpacing()];
}
```

This preserves the anti-MEV protection feature, aligns with the module's constants, and allows governance-configured high fees to take effect as intended.

## Proof of Concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.7.6;
pragma abicoder v2;

import "forge-std/Test.sol";
import {CLFactory} from "../contracts/core/CLFactory.sol";
import {CLPool} from "../contracts/core/CLPool.sol";
import {DynamicSwapFeeModule} from "../contracts/core/fees/DynamicSwapFeeModule.sol";
import {MockERC20} from "../contracts/mocks/MockERC20.sol";

contract C4PoC is Test {
    CLFactory public factory;
    DynamicSwapFeeModule public dynamicFeeModule;
    CLPool public pool;
    
    MockERC20 public token0;
    MockERC20 public token1;
    
    address public feeManager = address(0xFEE);
    
    function setUp() public {
        // Deploy tokens
        token0 = new MockERC20("Token A", "TKA", 18);
        token1 = new MockERC20("Token B", "TKB", 18);
        if (address(token0) > address(token1)) {
            (token0, token1) = (token1, token0);
        }
        
        // Deploy factory
        address poolImpl = address(new CLPool());
        factory = new CLFactory(poolImpl);
        
        address[] memory pools = new address[](0);
        uint24[] memory fees = new uint24[](0);
        dynamicFeeModule = new DynamicSwapFeeModule(
            address(factory),
            1e12,      // scalingFactor
            200_000,   // defaultFeeCap: 20% (> factory's 10% acceptance threshold)
            pools,
            fees
        );

        factory.setSwapFeeModule(address(dynamicFeeModule));
        factory.setSwapFeeManager(feeManager);

        uint160 sqrtPriceX96 = 79228162514264337593543950336; // tick 0
        address poolAddress = factory.createPool(
            address(token0),
            address(token1),
            50, // tickSpacing
            sqrtPriceX96
        );
        pool = CLPool(poolAddress);

        token0.mint(address(this), 1000000e18);
        token1.mint(address(this), 1000000e18);
        pool.mint(address(this), -5000, 5000, 1000000e18, abi.encode(address(this)));
    }

    function test_FactorySilentlyRejectsDynamicFeesAbove10Percent() public {
        // Governance sets 15% fee for anti-MEV protection (within DynamicSwapFeeModule's 50% MAX_BASE_FEE)
        vm.prank(feeManager);
        dynamicFeeModule.setCustomFee(address(pool), 150_000); // 15%
        
        // DynamicSwapFeeModule correctly returns the configured 15% fee
        uint24 moduleFee = dynamicFeeModule.getFee(address(pool));
        assertEq(uint256(moduleFee), 150_000, "Module should return 15% fee");
        
        // Factory silently rejects it and falls back to base fee
        uint24 factoryFee = factory.getSwapFee(address(pool));
        assertEq(uint256(factoryFee), 500, "Factory should reject >10% and return base fee");
        
        assertGt(uint256(moduleFee), 100_000, "Module fee should exceed 10% threshold");
        assertLt(uint256(factoryFee), uint256(moduleFee) / 100, "Factory fee should be <1% of intended fee");
    }
}
```
## Links to affected code
- [CLFactory.sol#L176-L189](https://github.com/code-423n4/2025-10-hybra-finance/blob/6299bfbc089158221e5c645d4aaceceea474f5be/cl/contracts/core/CLFactory.sol#L176-L189)