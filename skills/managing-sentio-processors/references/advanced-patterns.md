# Advanced Processor Patterns

Advanced patterns for production Sentio processors beyond basic event handling.

## Multi-Contract Binding

### Same ABI, Multiple Addresses

Bind the same processor type to multiple addresses in a loop:

```typescript
const POOL_ADDRESSES = [
  "0x88e6a0c2ddd26feeb64f039a2c41296fcb3f5640",
  "0xcbcdf9626bc03e24f779434178a73a0b4bad62ed",
  "0x8ad599c3a0ff1de082011efddc58f1908eb6e6d8",
]

for (const address of POOL_ADDRESSES) {
  UniswapV3PoolProcessor.bind({ address })
    .onEventSwap(swapHandler)
    .onEventBurn(burnHandler)
    .onEventMint(mintHandler)
}
```

### Different Contracts, Single Project

Bind multiple different contract types:

```typescript
// Factory contract
UniswapFactoryProcessor.bind({ address: FACTORY_ADDRESS })
  .onEventPoolCreated(async (event, ctx) => {
    ctx.meter.Counter("pools_created").add(1)
  })

// Individual pool contracts
UniswapPoolProcessor.bind({ address: POOL_ADDRESS })
  .onEventSwap(swapHandler)

// ERC20 for token tracking
ERC20Processor.bind({ address: TOKEN_ADDRESS })
  .onEventTransfer(transferHandler)

// Rewards/staking contract
RewardProcessor.bind({ address: REWARD_ADDRESS })
  .onEventRewardClaimed(rewardHandler)
```

## Named Processor Bindings

Use the `name` field for disambiguation in multi-binding scenarios:

```typescript
ERC20Processor.bind({
  address: SY_ADDRESS,
  startBlock: START_BLOCK,
  name: "Pendle Pool SY",     // Descriptive name
  network: CONFIG.BLOCKCHAIN,
}).onEventTransfer(syHandler)

ERC20Processor.bind({
  address: LP_ADDRESS,
  startBlock: START_BLOCK,
  name: "Pendle Pool LP",     // Different name for same processor type
  network: CONFIG.BLOCKCHAIN,
}).onEventTransfer(lpHandler)
```

## Lazy Caching Pattern

Cache expensive lookups (token info, pool info) to avoid redundant RPC calls:

```typescript
let poolInfoMap = new Map<string, Promise<PoolInfo>>()

async function getOrCreatePool(ctx: ContractContext): Promise<PoolInfo> {
  let infoPromise = poolInfoMap.get(ctx.address)
  if (!infoPromise) {
    // Store the Promise itself, not the result — prevents duplicate concurrent requests
    infoPromise = buildPoolInfo(
      ctx.contract.token0(),
      ctx.contract.token1(),
      ctx.contract.fee()
    )
    poolInfoMap.set(ctx.address, infoPromise)
  }
  return await infoPromise
}
```

**Key insight:** Store the `Promise` in the cache, not the resolved value. This prevents duplicate RPC calls when multiple handlers fire concurrently for the same pool.

## GlobalProcessor

Monitor all transactions/events on a network without binding to a specific contract:

```typescript
import { GlobalProcessor } from '@sentio/sdk/eth'

GlobalProcessor.bind({ network: EthChainId.ETHEREUM })
  .onBlockInterval(async (block, ctx) => {
    ctx.meter.Gauge("gas_used").record(block.gasUsed)
    ctx.meter.Gauge("tx_count").record(block.transactions.length)
  }, 1)  // Every block
  .onTransaction(async (tx, ctx) => {
    if (tx.value > 0n) {
      ctx.meter.Counter("eth_transfers").add(1)
    }
  })
  .onTrace(["0xa9059cbb"], async (trace, ctx) => {
    // Track ERC20 transfer function calls across all contracts
    ctx.meter.Counter("erc20_transfer_calls").add(1)
  })
```

## Handler Delegation Pattern

Extract handler logic into separate modules for large processors:

```typescript
// src/handlers/swap.ts
export async function handleSwap(event: SwapEvent, ctx: PoolContext) {
  const poolInfo = await getOrCreatePool(ctx)
  const volume = await calculateVolume(ctx, poolInfo, event)
  ctx.eventLogger.emit("Swap", { distinctId: event.args.recipient, volume })
}

// src/handlers/liquidity.ts
export async function handleMint(event: MintEvent, ctx: PoolContext) { ... }
export async function handleBurn(event: BurnEvent, ctx: PoolContext) { ... }

// src/processor.ts
import { handleSwap } from "./handlers/swap.js"
import { handleMint, handleBurn } from "./handlers/liquidity.js"

PoolProcessor.bind({ address: POOL })
  .onEventSwap(handleSwap)
  .onEventMint(handleMint)
  .onEventBurn(handleBurn)
```

## Contract View Calls in Handlers

Read on-chain state within event or interval handlers:

```typescript
.onBlockInterval(async (_, ctx) => {
  // Call view functions on the bound contract
  const phase = (await ctx.contract.currentPhase()).toString()
  const reward = (await ctx.contract.rewardPerBlockForStaking()).scaleDown(18)
  ctx.meter.Gauge("reward_per_block").record(reward, { phase })
}, 250)

// Read from external contracts
import { getERC20Contract } from '@sentio/sdk/eth/builtin/erc20'

.onEventSwap(async (event, ctx) => {
  const balance = await getERC20Contract(ctx, tokenAddress)
    .balanceOf(ctx.address, { blockTag: Number(ctx.blockNumber) })
})
```

## Aptos View Function Calls

```typescript
import { coin } from '@sentio/sdk/aptos/builtin/0x1'

coin.bind({ network: AptosNetwork.MAIN_NET })
  .onEventWithdrawEvent(async (evt, ctx) => {
    // Call a Move view function
    const [res] = await coin.view.supply(ctx.getClient(), {
      typeArguments: ['0x1::aptos_coin::AptosCoin']
    })
    if (res.vec.length > 0) {
      ctx.meter.Gauge('supply').record(res.vec[0])
    }
  })
```

## Sui Dynamic Fields

```typescript
import { SuiObjectProcessor } from '@sentio/sdk/sui'

SuiObjectProcessor.bind({ objectId: OBJECT_ID })
  .onTimeInterval(async (self, objects, ctx) => {
    const fields = await ctx.coder.getDynamicFields(
      objects,
      BUILTIN_TYPES.U64_TYPE,  // key type
      targetType                // value type
    )
    ctx.meter.Gauge('field_count').record(fields.length)
    for (const field of fields) {
      // Process dynamic field data
    }
  }, 60)
```

## Partition Keys

Partition data for scale-out processing:

```typescript
.onEventTransfer(handler, undefined, {
  partitionKey: (event) => event.args.from  // Partition by sender
})

// Or static partition:
.onEventSwap(handler, undefined, {
  partitionKey: "pool-swaps"
})
```

## Skip Decoding (Performance)

Skip ABI decoding for handlers that only need raw data:

```typescript
.onEvent(handler, filter, {
  skipDecoding: true  // Access raw log data, skip ABI decoding
})
```

## Sui startCheckpoint

For Sui processors, use `startCheckpoint` instead of `startBlock`:

```typescript
mint.bind({
  startCheckpoint: 8500000n  // Note: BigInt
})
  .onEventMintEvent(handler)
```

## baseLabels

Add default labels to all metrics from a processor binding:

```typescript
UniswapProcessor.bind({
  address: POOL,
  baseLabels: { pool: "usdc-eth", version: "v3" },
})
  .onEventSwap(async (event, ctx) => {
    // All metrics from this handler automatically include pool and version labels
    ctx.meter.Counter("swaps").add(1)  // → { pool: "usdc-eth", version: "v3" }
  })
```
