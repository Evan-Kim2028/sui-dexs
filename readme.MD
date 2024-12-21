# sui_dex

This is a continuation of the [sui-apps](https://github.com/sentioxyz/sentio-processors/tree/main/projects/sui-apps) project.




# Overview
This system tracks decentralized exchange (DEX) activities on Sui, including swaps, liquidity events, and TVL (Total Value Locked). The system is designed to be modular and extensible for adding new DEXes.

# Architecture

1. Constants Definition (constant.ts)
    Each DEX has its specific event types and package IDs defined for:  
    - Swap events
    - Pool creation events
    - Add/Remove liquidity events

    Example for a DEX (Cetus):
    ```
    export const CETUS_SWAP_TYPE = "0x1eabed72...::pool::SwapEvent"
    export const CETUS_POOL_TYPE = "0x1eabed72...::pool::Pool"
    export const CETUS_ADD_LIQUIDITY_TYPE = "0x1eabed72...::pool::AddLiquidityEvent"
    ```

2. Event Processing Pipeline
    a. Pool Creation & Tracking
        - Each DEX has its own processor file (e.g., dex-tvl-cetus.ts)
        - Listens for pool creation events
        - Binds new pools to a template processor for ongoing monitoring:
        ```
        factory.bind({})
        .onEventCreatePoolEvent(async (event, ctx) => {
            const pool_id = event.data_decoded.pool_id
            template.bind({
                objectId: pool_id
            }, ctx)
        })
        ```
    b. TVL Tracking
        - Template processor runs at intervals (default 1440 blocks)
        - Fetches current pool state
        - Calculates TVL for each token in pool
        - Emits standardized TVL metrics:
        ```
        ctx.eventLogger.emit("tvl_gauge", {
            pool: poolInfo.pool,
            tvl: total_tvl,
            pairName: poolInfo.pairName,
            project
        })
        ```
    c. Swap Event Processing (dex.ts)
        1. Registers event handlers for all whitelisted DEX types
        2. Processes swap events through standardization functions:
        - `recordClmmV3SwapEvent` (for V3-style DEXes)
        - `recordAmmV2SwapEvent` (for V2-style DEXes)
        - `recordClobSwapEvent` (for orderbook-style DEXes)
        - `recordMultiAssetSwapEvent` (for multi-asset pools)

3. Helper Functions (`dex-helper.ts`)
Key utilities for processing events:
    - Pool Information
    ```
    interface poolInfo {
        pool: string,
        symbol_a: string,
        symbol_b: string,
        decimal_a: number,
        decimal_b: number,
        pairName: string,
        fee: number,
        type_a: string,
        type_b: string
    }
    ```
    - Price and Value Calculation
        - `calculateSwapVol_USD`: Calculates USD volume for swaps
        - `calculateValueUSD`: Calculates total USD value for liquidity events
        - `calculateSingleTypeValueUSD`: Calculates USD value for single token

## Event Standardization
All events are normalized into a common format for consistent tracking:
```
ctx.eventLogger.emit("dex.swapEvents", {
    distinctId: sender,
    pool,
    amount_in,
    type_in,
    amount_out,
    type_out,
    usd_volume,
    pairName,
    project
})
```

This standardization ensures consistent analytics and monitoring across all DEXes in the ecosystem.