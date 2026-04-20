---
name: sentio-processors
description: Use when initializing Sentio projects, writing blockchain processor code, adding contracts/ABIs, testing processors, or deploying to the Sentio platform. Triggers on sentio CLI usage, processor development, multi-chain indexing, blockchain data analytics with Sentio SDK, DeFi protocols, points systems, or store entities.
---

# Managing Sentio Processors

## Overview

Sentio SDK is a TypeScript blockchain data indexing platform. Processors handle on-chain events, transactions, and state changes across Ethereum, Aptos, Sui, Solana, Starknet, Bitcoin, Cosmos, Fuel, and IOTA.

**Reference examples:** See [sentioxyz/sentio-processors](https://github.com/sentioxyz/sentio-processors) for 120+ production processors.

## When to Use

- Creating/initializing a Sentio indexing project
- Adding contracts or ABIs to a project
- Writing processor code for any supported chain
- Testing processors with `TestProcessorServer`
- Deploying processors with `sentio upload`
- Configuring `sentio.yaml`
- Multi-chain or multi-contract setups
- DeFi protocols (DEX, lending, staking, points systems)

## Progressive Disclosure

This skill follows a layered approach:

1. **This file** — CLI commands, core patterns, all handler types, metrics/events/store overview, best practices
2. **[references/advanced-patterns.md](references/advanced-patterns.md)** — Multi-contract binding, GlobalProcessor, lazy caching, view calls, partition keys, baseLabels
3. **[references/defi-patterns.md](references/defi-patterns.md)** — Price feeds, DEX/AMM, lending protocols, TVL tracking, Aptos/Sui DEX helpers
4. **[references/store-and-points.md](references/store-and-points.md)** — Store schema advanced features (relationships, timeseries, indexes, iterators, filter operators)
5. **[references/position-tracking-templates.md](references/position-tracking-templates.md)** — 10 protocol-specific points/position tracking templates (Simple Holding, Aave, Morpho, Vault/LP, Uni V3, Pendle, Compound, NFT, Uni V2, Uni V4)
6. **[references/production-examples.md](references/production-examples.md)** — 7 complete production processor examples (Uniswap, AAVE, Cetus, LiquidSwap, Lombard points, Scallop)

## Project Lifecycle

```
sentio create → sentio add → sentio gen → write processor → sentio test → sentio upload
```

### 1. Initialize Project

```bash
sentio create <project-name> --chain-type <type> --chain-id <id>
```

| Flag | Description | Default |
|------|-------------|---------|
| `--chain-type` | `eth`, `aptos`, `sui`, `solana`, `iota`, `fuel`, `starknet`, `raw` | `eth` |
| `--chain-id` | EVM chain ID (only for eth) | `1` |
| `--subproject` | Monorepo mode (no root dependencies) | false |

### 2. Add Contracts & Generate Types

```bash
sentio add <address> --chain <chain> --name <Name>   # Downloads ABI, updates sentio.yaml
sentio gen                                             # Generate TypeScript bindings
sentio build                                           # Full: gen + typecheck + bundle
```

| Flag | Description |
|------|-------------|
| `-c, --chain` | Chain ID (`1`, `56`) or name (`aptos_mainnet`, `sui_mainnet`) |
| `-n, --name` | Contract/module display name |
| `--api-key` | Explorer API key (Etherscan) |

### 3. Login & Deploy

```bash
sentio login                              # OAuth browser flow (prod)
sentio login --api-key <key>              # API key auth
sentio login --host test                  # Target test environment

sentio upload                             # Build + deploy
sentio upload --skip-build                # Deploy only
sentio upload --continue-from <ver>       # Hot-swap without re-indexing
sentio upload --checkpoint "1:18000000"   # Rollback to specific block
sentio upload --num-workers=4             # Parallel workers for compute-heavy processors
```

### 4. AI Processor Generation

```bash
sentio generate-processor --prompt "Track token transfers and calculate volume"
sentio gen-processor --prompt "Monitor DEX swaps"  # Short alias
```

## package.json

Recommend **yarn** over npm. With npm you may hit `BaseContract` / `DeferredTopicFilter` type conflicts (TS2344 due to duplicate ethers).

```json
{
  "name": "project-name",
  "type": "module",
  "scripts": {
    "build": "sentio build",
    "gen": "sentio gen",
    "upload": "sentio upload",
    "test": "sentio test"
  },
  "dependencies": {
    "@sentio/sdk": "^3.4.0"
  },
  "devDependencies": {
    "@sentio/cli": "^3.4.0",
    "typescript": "^5.4.5"
  }
}
```

If using npm and hitting ethers type conflicts, add:
```json
"overrides": {
  "ethers": "npm:@sentio/ethers@6.13.1-patch.4"
}
```

## sentio.yaml Quick Reference

```yaml
project: owner/project-name     # Required
host: prod                       # prod | test | staging | local

contracts:
  - chain: "1"                   # EVM: numeric IDs ("1", "56", "137", "42161")
    address: "0x..."
    name: "USDC"
  - chain: aptos_mainnet         # Move: aptos_mainnet, sui_mainnet, iota_mainnet
    address: "0x1"
  - chain: sui_mainnet
    address: "0xdee9..."
    name: "deepbook"

networkOverrides:                # Redirect chain ID to custom RPC
  - chain: '2390'
    host: "https://rpc.tac.build"

variables:                       # Runtime env vars
  - key: API_KEY
    value: xxx
    isSecret: true

debug: false
numWorkers: 1
```

## Processor Patterns by Chain

**Proxy contracts:** Use the **implementation contract's ABI** while binding to the **proxy address**.

### Ethereum / EVM

```typescript
import { Counter, Gauge } from '@sentio/sdk'
import { EthChainId } from '@sentio/chain'
import { ERC20Processor } from '@sentio/sdk/eth/builtin'
// Or generated: import { MyContractProcessor } from './types/eth/mycontract.js'

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
  .onTimeInterval(async (block, ctx) => {}, 60, 240)
```

**Handlers:** `onEvent*`, `onBlockInterval`, `onTimeInterval`, `onTransaction`, `onTrace`, `onCallXxx`

#### Event Filtering

Filter events to only process matching ones (e.g., only mints):

```typescript
const mintFilter = ERC20Processor.filters.Transfer(
  "0x0000000000000000000000000000000000000000", // from = null address (mint)
  null  // any recipient
);

ERC20Processor.bind({ address: TOKEN, network: NETWORK })
  .onEventTransfer(async (event, ctx) => {
    // only mint events reach here
  }, mintFilter);
```

#### Generic Event Handler

Catches any event from the contract:

```typescript
.onEvent(async (event, ctx) => {
  // event.eventName to determine which event
});
```

#### Function Call Handler (`onCallXxx`)

Triggers on internal function calls (trace-based). Access args and return values:

```typescript
MyContractProcessor.bind({ address: CONTRACT, network: NETWORK })
  .onCallDeposit(async (call, ctx) => {
    // call.args, call.returnValue, call.error
  });
```

**GlobalProcessor:** `GlobalProcessor.bind({ network }).onBlockInterval(handler)`

### Aptos

```typescript
import { AptosNetwork } from '@sentio/sdk/aptos'
import { coin } from '@sentio/sdk/aptos/builtin/0x1'

coin.bind({ network: AptosNetwork.MAIN_NET })
  .onEventWithdrawEvent(async (evt, ctx) => {
    ctx.meter.Counter('withdrawals').add(1)
    ctx.eventLogger.emit('Withdraw', {
      distinctId: evt.guid.account_address,
      amount: evt.data_decoded.amount,
    })
  })
```

**Handlers:** `onEvent*`, `onEntry*`, `onTransaction`, `onTimeInterval`, `onVersionInterval`
**Resources:** `AptosResourcesProcessor.bind({ address }).onResourceChange(handler, typeString)`

### Sui

```typescript
import { SuiNetwork, SuiObjectTypeProcessor } from '@sentio/sdk/sui'
import { pool } from './types/sui/deepbook.js'

pool.bind({ network: SuiNetwork.MAIN_NET })
  .onEventSwap(async (evt, ctx) => {
    ctx.meter.Counter('swaps').add(1)
  })

SuiObjectTypeProcessor.bind({ objectType: pool.Pool.type() })
  .onObjectChange((changes, ctx) => {
    ctx.meter.Counter('pool_updates').add(changes.length)
  })
```

**Handlers:** `onEvent*`, `onEntryFunctionCall`, `onObjectChange`, `onTimeInterval`
**Note:** Use `startCheckpoint: 8500000n` (BigInt) instead of `startBlock` for Sui.

### Solana

```typescript
SolanaGlobalProcessor.bind({
  name: 'my-program',
  instructionCoder: new Anchor.BorshInstructionCoder(idl),
}).onInstruction('transfer', async (instruction, ctx, accounts) => {
  ctx.meter.Counter('transfers').add(1)
})
```

### Starknet

```typescript
StarknetProcessor.bind({ address: '0x...', network: StarknetChainId.STARKNET_MAINNET })
  .onEvent('Transfer', async (events, ctx) => {
    ctx.meter.Counter('transfers').add(events.length)
  })
```

## Factory Pattern (ProcessorTemplate)

For dynamically-created contracts (e.g., DEX pair factory):

```typescript
import { UniswapV2PairProcessorTemplate } from "./types/eth/uniswapv2pair.js";
import { UniswapV2FactoryProcessor } from "./types/eth/uniswapv2factory.js";

// 1. Define template with handlers
const pairTemplate = new UniswapV2PairProcessorTemplate()
  .onEventSwap(async (event, ctx) => {
    ctx.meter.Counter("swap_volume").add(event.args.amount0Out);
  })
  .onEventSync(async (event, ctx) => {
    ctx.meter.Gauge("reserve0").record(event.args.reserve0);
  });

// 2. Bind factory and dynamically bind new pairs
UniswapV2FactoryProcessor.bind({
  address: FACTORY_ADDRESS,
  network: NETWORK,
})
  .onEventPairCreated(async (event, ctx) => {
    pairTemplate.bind(
      { address: event.args.pair, startBlock: ctx.blockNumber },
      ctx
    );
  });
```

## Metrics & Events

```typescript
// Global registration (recommended)
const counter = Counter.register('name', { unit: 'token' })
counter.add(ctx, value, { label: 'value' })

// Inline via context
ctx.meter.Counter('name').add(value, { label: 'value' })
ctx.meter.Gauge('name').record(value, { label: 'value' })

// Sparse gauge (high cardinality — many pools/tokens)
const vol = Gauge.register("vol", {
  sparse: true,
  aggregationConfig: {
    intervalInMinutes: [60],
    discardOrigin: true,  // only keep aggregated vol_count, vol_sum
  },
})

// Event logging
ctx.eventLogger.emit('EventName', {
  distinctId: address,     // Primary entity ID (used for user-level analytics)
  message: 'description',
  // ... arbitrary key-value attributes
})

// Exporter (export to external systems via webhook)
ctx.exporter.emit({ key: "value" })
```

**Label cardinality warning:** ~10,000 unique series limit per metric. NEVER use wallet addresses, tx hashes, or token IDs as labels. Use event logs or entities for high-cardinality data instead.

## Store (Database) API

Define entities in `schema.graphql`, generated by `sentio gen`:

```graphql
type AccountSnapshot @entity {
  id: ID!
  timestampMilli: BigInt!
  balance: BigInt!
}
```

```typescript
import { AccountSnapshot } from "./schema/store.js"

await ctx.store.get(AccountSnapshot, id)
await ctx.store.upsert(new AccountSnapshot({ id, timestampMilli, balance }))
await ctx.store.list(AccountSnapshot, [])  // List all
await ctx.store.delete(AccountSnapshot, id)
```

**Critical:** Enable sequential execution when using store to prevent race conditions:
```typescript
import { GLOBAL_CONFIG } from "@sentio/runtime"
GLOBAL_CONFIG.execution = { sequential: true }
```

See [references/store-and-points.md](references/store-and-points.md) for points system patterns.

## Price Feeds

```typescript
import { getPriceByType, token } from "@sentio/sdk/utils"

const info = await token.getERC20TokenInfo(EthChainId.ETHEREUM, tokenAddr)
const price = await getPriceByType(EthChainId.ETHEREUM, tokenAddr, ctx.timestamp) || 0
const usdValue = amount.scaleDown(info.decimal).multipliedBy(price)
```

See [references/defi-patterns.md](references/defi-patterns.md) for caching patterns and DeFi examples.

## Testing

```typescript
import { TestProcessorServer, firstCounterValue } from '@sentio/sdk/testing'

const service = new TestProcessorServer(() => import('./processor.js'))
before(async () => { await service.start() })

// Ethereum
const resp = await service.eth.testLog(mockTransferLog('0x...', { from, to, value }))
assert.equal(firstCounterValue(resp.result, 'transfer_count'), 1n)
```

| Chain | Facet | Key Methods |
|-------|-------|-------------|
| Ethereum | `service.eth` | `testLog()`, `testBlock()`, `testTransaction()`, `testTrace()` |
| Aptos | `service.aptos` | `testEvent()`, `testCall()`, `testResourceChange()` |
| Sui | `service.sui` | `testEvent()`, `testObjectChange()` |
| Solana | `service.solana` | `testInstruction()` |

```bash
sentio test                              # All tests
sentio test --test-name-pattern="swap"   # Filter by name
```

## Project Structure

```
my-project/
  sentio.yaml              # Project config
  schema.graphql           # Store entity definitions (optional)
  package.json             # @sentio/sdk + @sentio/cli deps
  tsconfig.json
  abis/{chain}/            # Contract ABIs
  src/
    processor.ts           # Main processor code
    processor.test.ts      # Tests
    types/                 # Auto-generated (sentio gen)
    schema/                # Auto-generated store entities
  dist/lib.js              # Bundled output (sentio build)
```

## Best Practices

1. **Use `sequential: true`** when handlers share state via entity store
2. **Do NOT use global variables** for persistent state — use `ctx.store`
3. **Use `startBlock`** to skip irrelevant history and reduce backfill cost
4. **Prefer Metrics over Logs/Entities** for aggregated numerical data (lower cost)
5. **Never use addresses/hashes as metric labels** — ~10k series limit per metric
6. **Use event filters** (`Processor.filters`) to reduce unnecessary handler executions
7. **Increase backfill intervals** to the largest acceptable value for cost reduction
8. **Use `listIterator()`** instead of `list()` for large entity datasets
9. **Use `Promise.all`** for parallel contract calls and entity processing
10. **Use `BigDecimal`** and `scaleDown()` for precision — avoid floating point
11. **Filter null addresses** with `isNullAddress()` for mint/burn events
12. **Skip self-transfers** — `if (from == to) return;`

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Writing processor before `sentio gen` | Always `sentio add` then `sentio gen` first |
| Wrong import path for generated types | Use `./types/{chain}/{name}.js` (`.js` extension for ESM) |
| Forgetting `.scaleDown(decimals)` | `.scaleDown(18)` for ETH, `.scaleDown(6)` for USDC |
| Not setting `startBlock` / `startCheckpoint` | Processor starts from genesis without it |
| Store entities without sequential execution | Add `GLOBAL_CONFIG.execution = { sequential: true }` |
| Deploying without login | Run `sentio login` first |
| Caching resolved values instead of Promises | Cache the `Promise` to prevent duplicate concurrent RPC calls |
