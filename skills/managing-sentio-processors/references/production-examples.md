# Production Processor Examples

Complete working processor examples from [sentioxyz/sentio-processors](https://github.com/sentioxyz/sentio-processors).

## 1. ERC-20 Transfer Tracker (Minimal)

```typescript
import { Counter } from '@sentio/sdk'
import { EthChainId } from '@sentio/chain'
import { ERC20Processor } from '@sentio/sdk/eth/builtin'

const transfers = Counter.register('transfer_count')

ERC20Processor.bind({
  address: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48',
  network: EthChainId.ETHEREUM,
  startBlock: 6082465,
})
  .onEventTransfer(async (event, ctx) => {
    transfers.add(ctx, 1)
    ctx.eventLogger.emit('Transfer', {
      distinctId: event.args.from,
      from: event.args.from,
      to: event.args.to,
      amount: event.args.value.scaleDown(6),
    })
  })
```

## 2. Uniswap V3 Pool Tracker (Multi-Pool DEX)

```typescript
import { BigDecimal, Gauge } from "@sentio/sdk"
import { EthChainId } from "@sentio/sdk/eth"
import { getPriceByType, token } from "@sentio/sdk/utils"
import { UniswapProcessor, UniswapContext } from './types/eth/uniswap.js'
import { getERC20Contract } from '@sentio/sdk/eth/builtin/erc20'

const vol = Gauge.register("vol", {
  sparse: true,
  aggregationConfig: { intervalInMinutes: [60] }
})

let poolInfoMap = new Map<string, Promise<PoolInfo>>()

interface PoolInfo {
  token0: token.TokenInfo
  token1: token.TokenInfo
  token0Address: string
  token1Address: string
  fee: string
}

async function getOrCreatePool(ctx: UniswapContext): Promise<PoolInfo> {
  let p = poolInfoMap.get(ctx.address)
  if (!p) {
    p = (async () => {
      const [addr0, addr1, fee] = await Promise.all([
        ctx.contract.token0(), ctx.contract.token1(), ctx.contract.fee()
      ])
      const [info0, info1] = await Promise.all([
        token.getERC20TokenInfo(EthChainId.ETHEREUM, addr0),
        token.getERC20TokenInfo(EthChainId.ETHEREUM, addr1),
      ])
      return { token0: info0, token1: info1, token0Address: addr0, token1Address: addr1, fee: fee.toString() }
    })()
    poolInfoMap.set(ctx.address, p)
  }
  return p
}

const poolAddresses = [
  "0x88e6a0c2ddd26feeb64f039a2c41296fcb3f5640", // USDC/ETH
  "0x8ad599c3a0ff1de082011efddc58f1908eb6e6d8", // USDC/ETH 0.3%
  // ... add more
]

for (const address of poolAddresses) {
  UniswapProcessor.bind({ address })
    .onEventSwap(async (event, ctx) => {
      const info = await getOrCreatePool(ctx)
      const price0 = await getPriceByType(EthChainId.ETHEREUM, info.token0Address, ctx.timestamp) || 0
      const amount0 = event.args.amount0.scaleDown(info.token0.decimal)
      const usdVol = amount0.abs().multipliedBy(price0)

      vol.record(ctx, usdVol, {
        poolName: `${info.token0.symbol}/${info.token1.symbol}`,
        type: "swap",
      })

      ctx.eventLogger.emit("Swap", {
        distinctId: event.args.recipient,
        poolName: `${info.token0.symbol}/${info.token1.symbol}`,
        amount: usdVol,
      })
    })
    .onTimeInterval(async (_, ctx) => {
      const info = await getOrCreatePool(ctx)
      for (const [addr, tokenInfo] of [[info.token0Address, info.token0], [info.token1Address, info.token1]]) {
        const balance = await getERC20Contract(ctx as any, addr as string)
          .balanceOf(ctx.address, { blockTag: Number(ctx.blockNumber) })
        const price = await getPriceByType(EthChainId.ETHEREUM, addr as string, ctx.timestamp) || 0
        const tvl = balance.scaleDown((tokenInfo as token.TokenInfo).decimal).multipliedBy(price)
        ctx.meter.Gauge("tvl").record(tvl, { token: (tokenInfo as token.TokenInfo).symbol })
      }
    }, 60, 24 * 60 * 30)
}
```

## 3. AAVE Multi-Chain Lending

```typescript
import { BigDecimal, Gauge } from '@sentio/sdk'
import { EthChainId } from '@sentio/sdk/eth'
import { PoolProcessor } from './types/eth/pool.js'
import { getPriceByType, token } from "@sentio/sdk/utils"

const vol = Gauge.register("vol", { sparse: true, aggregationConfig: { intervalInMinutes: [60] } })

const CHAINS = new Map<EthChainId, string>([
  [EthChainId.ETHEREUM, "0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2"],
  [EthChainId.OPTIMISM, "0x794a61358d6845594f94dc1db02a252b5b4814ad"],
  [EthChainId.ARBITRUM, "0x794a61358d6845594f94dc1db02a252b5b4814ad"],
  [EthChainId.POLYGON, "0x794a61358d6845594f94dc1db02a252b5b4814ad"],
])

let tokenMap = new Map<string, Promise<token.TokenInfo | undefined>>()

CHAINS.forEach((addr, chainId) => {
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
      const info = await getOrCreateToken(ctx.chainId, evt.args.reserve)
      if (!info) return
      const price = await getPriceByType(ctx.chainId, evt.args.reserve, ctx.timestamp) || 0
      const scaledAmount = evt.args.amount.scaleDown(info.decimal)
      vol.record(ctx, scaledAmount.multipliedBy(price), { token: info.symbol, type: "borrow" })
      ctx.eventLogger.emit("borrow", {
        distinctId: evt.args.user,
        amount: evt.args.amount,
        value: scaledAmount.multipliedBy(price),
        reserve: evt.args.reserve,
        borrowRate: evt.args.borrowRate,
      })
    })
    .onEventRepay(async (evt, ctx) => {
      ctx.meter.Counter("repay_counter").add(1)
      ctx.eventLogger.emit("repay", {
        distinctId: evt.args.user,
        amount: evt.args.amount,
        reserve: evt.args.reserve,
      })
    })
    .onEventFlashLoan(async (evt, ctx) => {
      ctx.eventLogger.emit("flashloan", {
        distinctId: evt.args.initiator,
        amount: evt.args.amount,
        asset: evt.args.asset,
        premium: evt.args.premium,
      })
    })
})

async function getOrCreateToken(chainId: EthChainId, addr: string) {
  let p = tokenMap.get(`${chainId}-${addr}`)
  if (!p) {
    p = token.getERC20TokenInfo(chainId, addr).catch(() => undefined)
    tokenMap.set(`${chainId}-${addr}`, p)
  }
  return p
}
```

## 4. Aptos DEX (LiquidSwap)

```typescript
import { AptosResourcesProcessor } from "@sentio/sdk/aptos"
import { AptosDex, getPairValue } from "@sentio/sdk/aptos/ext"
import * as v0 from "./types/aptos/liquidswap.js"

const liquidity_pool = v0.liquidity_pool

const liquidSwap = new AptosDex(volume, volumeByCoin, tvlAll, tvl, tvlByPool, {
  getXReserve: pool => pool.coin_x_reserve.value,
  getYReserve: pool => pool.coin_y_reserve.value,
  getExtraPoolTags: pool => ({ curve: pool.type_arguments[2] }),
  poolType: liquidity_pool.LiquidityPool.type()
})

liquidity_pool.bind()
  .onEventPoolCreatedEvent(async (evt, ctx) => {
    ctx.meter.Counter("num_pools").add(1)
  })
  .onEventSwapEvent(async (evt, ctx) => {
    await liquidSwap.recordTradingVolume(ctx,
      evt.type_arguments[0], evt.type_arguments[1],
      evt.data_decoded.x_in + evt.data_decoded.x_out,
      evt.data_decoded.y_in + evt.data_decoded.y_out,
    )
  })
  .onEventLiquidityAddedEvent(async (evt, ctx) => {
    const value = await getPairValue(ctx,
      evt.type_arguments[0], evt.type_arguments[1],
      evt.data_decoded.added_x_val, evt.data_decoded.added_y_val)
    ctx.eventLogger.emit("liquidity", {
      distinctId: ctx.transaction.sender,
      value: value.toNumber(),
    })
  })

// Periodic TVL via resource processor
AptosResourcesProcessor.bind({ address: RESOURCE_ADDRESS })
  .onTimeInterval(async (resources, ctx) => {
    // Process pool resources for TVL snapshots
  }, 60, 12 * 60)
```

## 5. Sui DEX (Cetus)

```typescript
import { pool, factory } from "./types/sui/cetus.js"
import { SuiObjectTypeProcessor } from "@sentio/sdk/sui"
import { getPriceByType } from "@sentio/sdk/utils"

factory.bind({})
  .onEventCreatePoolEvent(async (event, ctx) => {
    ctx.meter.Counter("create_pool_counter").add(1)
    ctx.eventLogger.emit("CreatePoolEvent", {
      distinctId: event.sender,
      pool_id: event.data_decoded.pool_id,
    })
  })

pool.bind({})
  .onEventSwapEvent(async (event, ctx) => {
    const poolInfo = await getOrCreatePool(ctx, event.data_decoded.pool)
    const atob = event.data_decoded.atob
    const amount_in = Number(event.data_decoded.amount_in) /
      Math.pow(10, atob ? poolInfo.decimal_a : poolInfo.decimal_b)
    const usd_volume = await calculateSwapVol(ctx, poolInfo, amount_in, atob)
    ctx.eventLogger.emit("SwapEvent", {
      distinctId: event.sender,
      amount_in,
      usd_volume,
      pairName: poolInfo.pairName,
    })
  })

// TVL tracking via object type processor
SuiObjectTypeProcessor.bind({ objectType: pool.Pool.type() })
  .onTimeInterval(async (self, _, ctx) => {
    const coin_a = Number(self.data_decoded.coin_a) / Math.pow(10, decimal_a)
    const coin_b = Number(self.data_decoded.coin_b) / Math.pow(10, decimal_b)
    const price_a = await getPriceByType(ctx.chainId, type_a, ctx.timestamp) || 0
    const price_b = await getPriceByType(ctx.chainId, type_b, ctx.timestamp) || 0
    ctx.eventLogger.emit("tvl", {
      pool: ctx.objectId,
      tvl: coin_a * price_a + coin_b * price_b,
    })
  }, 10, 1440 * 15)
```

## 6. Points System with Store (Lombard)

```typescript
import { GLOBAL_CONFIG } from "@sentio/runtime"
import { BigDecimal } from "@sentio/sdk"
import { ERC20Processor } from "@sentio/sdk/eth/builtin"
import { isNullAddress } from "@sentio/sdk/eth"
import { AccountSnapshot } from "./schema/store.js"
import { getERC20ContractOnContext } from "@sentio/sdk/eth/builtin/erc20"

GLOBAL_CONFIG.execution = { sequential: true }

const DAILY_POINTS = 1000
const MULTIPLIER = 2
const MS_PER_DAY = 86400000
const TOKEN_DECIMALS = 8

ERC20Processor.bind({ network: NETWORK, address: VAULT_ADDRESS })
  .onEventTransfer(async (event, ctx) => {
    const newSnapshots = await Promise.all(
      [event.args.from, event.args.to]
        .filter(a => !isNullAddress(a))
        .map(a => process(ctx, a, undefined, "Transfer"))
    )
    await ctx.store.upsert(newSnapshots)
  })
  .onTimeInterval(async (_, ctx) => {
    const accounts = await ctx.store.list(AccountSnapshot, [])
    const newSnapshots = await Promise.all(
      accounts.map(a => process(ctx, a.id.toString(), a, "TimeInterval"))
    )
    await ctx.store.upsert(newSnapshots)
  }, 60, 4 * 60)

async function process(ctx, account, snapshot, trigger) {
  if (!snapshot) snapshot = await ctx.store.get(AccountSnapshot, account)
  const points = snapshot ? calcPoints(ctx, snapshot) : new BigDecimal(0)

  const [lbtcTotal, lpBalance, totalSupply] = await Promise.all([
    getERC20ContractOnContext(ctx, LBTC_ADDRESS).balanceOf(VAULT_ADDRESS),
    ctx.contract.balanceOf(account),
    ctx.contract.totalSupply(),
  ])

  const newSnapshot = new AccountSnapshot({
    id: account,
    timestampMilli: BigInt(ctx.timestamp.getTime()),
    lbtcBalance: (lbtcTotal * lpBalance) / totalSupply,
  })

  ctx.eventLogger.emit("point_update", {
    account, points, trigger,
    newBalance: newSnapshot.lbtcBalance.scaleDown(TOKEN_DECIMALS),
    multiplier: MULTIPLIER,
  })
  return newSnapshot
}

function calcPoints(ctx, snapshot) {
  const deltaDay = (ctx.timestamp.getTime() - Number(snapshot.timestampMilli)) / MS_PER_DAY
  return snapshot.lbtcBalance
    .scaleDown(TOKEN_DECIMALS)
    .multipliedBy(DAILY_POINTS)
    .multipliedBy(deltaDay)
    .multipliedBy(MULTIPLIER)
}
```

## 7. Scallop Lending (Sui, Minimal)

```typescript
import { SuiContext } from "@sentio/sdk/sui"
import { mint } from "./types/sui/scallop.js"
import { normalizeSuiAddress } from "@mysten/sui/utils"

mint.bind({ startCheckpoint: 8500000n })
  .onEventMintEvent(async (event: mint.MintEventInstance, ctx: SuiContext) => {
    const deposit_asset = normalizeSuiAddress(event.data_decoded.deposit_asset.name)
    const coin_symbol = COIN_MAP.get(deposit_asset) ?? "unk"
    const decimal = DECIMAL_MAP.get(coin_symbol) ?? 9
    const deposit_amount = Number(event.data_decoded.deposit_amount) / 10 ** decimal

    if (deposit_amount >= 5) {
      ctx.eventLogger.emit("Mint", {
        distinctId: event.sender,
        deposit_amount,
        coin_symbol,
        project: "scallop",
      })
    }
  })
```
