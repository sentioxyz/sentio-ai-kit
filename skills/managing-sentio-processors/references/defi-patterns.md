# DeFi Processor Patterns

Production-proven patterns from [sentioxyz/sentio-processors](https://github.com/sentioxyz/sentio-processors).

## Price Feeds & USD Valuation

Use `getPriceByType` and `token.getERC20TokenInfo` from `@sentio/sdk/utils`:

```typescript
import { getPriceByType, token } from "@sentio/sdk/utils"

// Cache token info to avoid repeated RPC calls
let tokenMap = new Map<string, Promise<token.TokenInfo | undefined>>()

async function getOrCreateToken(ctx: EthContext, address: string) {
  let infoPromise = tokenMap.get(address)
  if (!infoPromise) {
    infoPromise = token.getERC20TokenInfo(ctx.chainId, address)
    tokenMap.set(address, infoPromise)
  }
  return infoPromise
}

// Get USD value of a token amount
async function getUsdValue(ctx: EthContext, tokenAddr: string, amount: bigint) {
  const info = await getOrCreateToken(ctx, tokenAddr)
  if (!info) return BigDecimal(0)
  const price = await getPriceByType(ctx.chainId, tokenAddr, ctx.timestamp) || 0
  return amount.scaleDown(info.decimal).multipliedBy(price)
}
```

## DEX / AMM Processors

### Multi-Pool Binding (Uniswap V3 pattern)

Bind the same processor to many pool addresses in a loop:

```typescript
const poolAddresses = [
  "0x88e6a0c2ddd26feeb64f039a2c41296fcb3f5640", // USDC/ETH
  "0xcbcdf9626bc03e24f779434178a73a0b4bad62ed", // WBTC/ETH
  // ... more pools
]

for (const address of poolAddresses) {
  UniswapV3PoolProcessor.bind({ address })
    .onEventSwap(async (event, ctx) => {
      const info = await getOrCreatePool(ctx)
      const [amount0, price0] = await getTokenDetails(ctx, info.token0, event.args.amount0)
      vol.record(ctx, price0.abs(), { poolName: info.name, type: "swap" })
      ctx.eventLogger.emit("Swap", {
        distinctId: event.args.recipient,
        poolName: info.name,
        amount: price0,
        message: `${info.name} swap ${amount0} ${info.token0.symbol}`,
      })
    })
    .onEventBurn(burnHandler)
    .onEventMint(mintHandler)
    .onTimeInterval(tvlHandler, 60, 24 * 60 * 30)
}
```

### TVL Tracking with onTimeInterval

Read on-chain balances periodically for TVL snapshots:

```typescript
.onTimeInterval(async (_, ctx) => {
  const token0Balance = await getERC20Contract(ctx, token0Addr)
    .balanceOf(ctx.address, { blockTag: Number(ctx.blockNumber) })
  const token1Balance = await getERC20Contract(ctx, token1Addr)
    .balanceOf(ctx.address, { blockTag: Number(ctx.blockNumber) })
  const tvl0 = token0Balance.scaleDown(decimal0).multipliedBy(price0)
  const tvl1 = token1Balance.scaleDown(decimal1).multipliedBy(price1)
  ctx.meter.Gauge("tvl").record(tvl0, { token: symbol0, pool: poolName })
  ctx.meter.Gauge("tvl").record(tvl1, { token: symbol1, pool: poolName })
}, 60, 24 * 60 * 30)  // 60 min recent, 30 days backfill
```

### Sparse Gauge Options

For high-cardinality metrics (many pools/tokens), use sparse mode:

```typescript
export const volOptions = {
  sparse: true,
  aggregationConfig: {
    intervalInMinutes: [60],
  }
}
const vol = Gauge.register("vol", volOptions)
```

## Lending Protocol Processors

### Multi-Chain Deployment (AAVE pattern)

Deploy same processor across multiple chains:

```typescript
const CHAIN_ADDRESS_MAP = new Map<EthChainId, string>([
  [EthChainId.ETHEREUM, "0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2"],
  [EthChainId.OPTIMISM, "0x794a61358d6845594f94dc1db02a252b5b4814ad"],
  [EthChainId.ARBITRUM, "0x794a61358d6845594f94dc1db02a252b5b4814ad"],
  [EthChainId.POLYGON, "0x794a61358d6845594f94dc1db02a252b5b4814ad"],
])

CHAIN_ADDRESS_MAP.forEach((addr, chainId) => {
  PoolProcessor.bind({ address: addr, network: chainId })
    .onEventSupply(async (evt, ctx) => {
      ctx.meter.Counter("supply_counter").add(1)
      ctx.eventLogger.emit("supply", {
        distinctId: evt.args.user,
        amount: evt.args.amount,
        reserve: evt.args.reserve,
      })
    })
    .onEventBorrow(async (evt, ctx) => {
      const value = await getUsdValue(ctx, evt.args.reserve, evt.args.amount)
      ctx.eventLogger.emit("borrow", {
        distinctId: evt.args.user,
        amount: evt.args.amount,
        value,
        reserve: evt.args.reserve,
        borrowRate: evt.args.borrowRate,
      })
    })
    .onEventRepay(repayHandler)
    .onEventFlashLoan(flashLoanHandler)
})
```

### Total Supply/Borrow Snapshots (Compound pattern)

```typescript
.onTimeInterval(async (_, ctx) => {
  try {
    const totalSupply = await ctx.contract.totalSupply()
    const totalBorrow = await ctx.contract.totalBorrow()
    ctx.meter.Gauge("total_supply").record(totalSupply.scaleDown(6))
    ctx.meter.Gauge("total_borrow").record(totalBorrow.scaleDown(6))
  } catch (e) {
    console.log("get total supply failed", e)
  }
})
```

## Aptos DEX Patterns

### AptosDex Helper Class

The SDK provides `AptosDex` for standard DEX volume/TVL tracking:

```typescript
import { AptosDex, getPairValue } from "@sentio/sdk/aptos/ext"

const liquidSwap = new AptosDex<PoolType<any, any, any>>(
  volume, volumeByCoin, tvlAll, tvl, tvlByPool, {
    getXReserve: pool => pool.coin_x_reserve.value,
    getYReserve: pool => pool.coin_y_reserve.value,
    getExtraPoolTags: pool => ({ curve: pool.type_arguments[2] }),
    poolType: liquidity_pool.LiquidityPool.type()
  }
)

// In swap handler:
const value = await liquidSwap.recordTradingVolume(ctx,
  evt.type_arguments[0], evt.type_arguments[1],
  evt.data_decoded.x_in + evt.data_decoded.x_out,
  evt.data_decoded.y_in + evt.data_decoded.y_out,
  { curve: getCurve(evt.type_arguments[2]) }
)
```

### AptosResourcesProcessor for TVL

Track Move resources periodically:

```typescript
import { AptosResourcesProcessor } from "@sentio/sdk/aptos"

AptosResourcesProcessor.bind({ address: resourceAddress })
  .onTimeInterval(async (resources, ctx) => {
    // resources contains all resources at the address
    // Filter and process pool resources for TVL
    for (const resource of resources) {
      if (isPoolResource(resource)) {
        const tvl = calculateTvl(resource)
        ctx.meter.Gauge("tvl").record(tvl, { pool: resource.type })
      }
    }
  }, 60, 12 * 60)
```

## Sui DEX Patterns

### Factory + Pool Event Binding (Cetus pattern)

```typescript
import { pool, factory } from "./types/sui/cetus.js"

factory.bind({})
  .onEventCreatePoolEvent(async (event, ctx) => {
    ctx.meter.Counter("create_pool_counter").add(1)
    const pool_id = event.data_decoded.pool_id
    ctx.eventLogger.emit("CreatePoolEvent", {
      distinctId: event.sender,
      pool_id,
    })
  })

pool.bind({})
  .onEventSwapEvent(async (event, ctx) => {
    const poolInfo = await helper.getOrCreatePool(ctx, event.data_decoded.pool)
    const atob = event.data_decoded.atob
    const amount_in = Number(event.data_decoded.amount_in) / Math.pow(10,
      atob ? poolInfo.decimal_a : poolInfo.decimal_b)
    ctx.eventLogger.emit("SwapEvent", {
      distinctId: event.sender,
      amount_in,
      pairName: poolInfo.pairName,
    })
  })
```

### SuiObjectTypeProcessor for TVL

```typescript
SuiObjectTypeProcessor.bind({ objectType: pool.Pool.type() })
  .onTimeInterval(async (self, _, ctx) => {
    const coin_a = Number(self.data_decoded.coin_a) / Math.pow(10, decimal_a)
    const coin_b = Number(self.data_decoded.coin_b) / Math.pow(10, decimal_b)
    ctx.eventLogger.emit("tvl", { pool: ctx.objectId, tvl: tvl_a + tvl_b })
  }, 10, 1440 * 15)
```

### Accessing Raw Transaction Inputs

For events that need context from transaction inputs:

```typescript
.onEventCollectRewardEvent(async (event, ctx) => {
  // Access full transaction data
}, { inputs: true, allEvents: true })
```
