# Points & Position Tracking Templates

Protocol-specific templates for points/rewards programs that track user positions over time.

## Architecture Overview

All points processors follow the same pattern:

1. **Discover users** via token Transfer events (ERC20/ERC721)
2. **Snapshot balances** in entity store on each event + periodic time interval
3. **Calculate points** from old snapshot: `balance * timeDelta * dailyPoints * multiplier`
4. **Emit event logs** with before/after state for off-chain aggregation
5. **Update snapshot** with current balance and timestamp

---

## Core Framework

### Required Setup

```typescript
import { GLOBAL_CONFIG } from "@sentio/runtime";
import { BigDecimal } from "@sentio/sdk";
import { EthContext, isNullAddress } from "@sentio/sdk/eth";
import { ERC20Processor } from "@sentio/sdk/eth/builtin";
import { AccountSnapshot } from "./schema/store.js";

GLOBAL_CONFIG.execution = { sequential: true };
```

### Core processAccount Template

```typescript
const MILLISECOND_PER_DAY = 60 * 60 * 1000 * 24;

async function processAccount(
  ctx: EthContext,
  account: string,
  snapshot: AccountSnapshot | undefined,
  triggerEvent: string
): Promise<AccountSnapshot> {
  if (!snapshot) {
    snapshot = await ctx.store.get(AccountSnapshot, account);
  }

  const points = snapshot ? calcPoints(ctx, snapshot) : new BigDecimal(0);
  const newBalance = await getBalance(ctx, account);

  const newSnapshot = new AccountSnapshot({
    id: account,
    timestampMilli: BigInt(ctx.timestamp.getTime()),
    balance: newBalance,
  });

  ctx.eventLogger.emit("point_update", {
    account,
    points,
    triggerEvent,
    snapshotTimestampMilli: snapshot?.timestampMilli ?? 0n,
    snapshotBalance: snapshot?.balance.scaleDown(TOKEN_DECIMALS) ?? 0,
    newTimestampMilli: newSnapshot.timestampMilli,
    newBalance: newSnapshot.balance.scaleDown(TOKEN_DECIMALS),
    multiplier: MULTIPLIER,
  });

  return newSnapshot;
}
```

### Points Calculation

```typescript
function calcPoints(ctx: EthContext, snapshot: AccountSnapshot): BigDecimal {
  const nowMilli = ctx.timestamp.getTime();
  if (nowMilli <= Number(snapshot.timestampMilli)) {
    return new BigDecimal(0);
  }
  const deltaDay =
    (nowMilli - Number(snapshot.timestampMilli)) / MILLISECOND_PER_DAY;

  return snapshot.balance
    .scaleDown(TOKEN_DECIMALS)
    .multipliedBy(deltaDay)
    .multipliedBy(DAILY_POINTS)
    .multipliedBy(MULTIPLIER);
}
```

### Standard Binding Pattern

```typescript
ERC20Processor.bind({ address: TOKEN_ADDRESS, network: NETWORK })
  .onEventTransfer(async (event, ctx) => {
    const { from, to } = event.args;
    if (from == to) return;
    const accounts = [from, to].filter((a) => !isNullAddress(a));
    const snapshots = await Promise.all(
      accounts.map((a) => processAccount(ctx, a, undefined, event.name))
    );
    await ctx.store.upsert(snapshots);
  })
  .onTimeInterval(
    async (_, ctx) => {
      await updateAll(ctx, "TimeInterval");
    },
    60,  // intervalInMinutes (real-time)
    60   // backfillIntervalInMinutes (historical)
  );

async function updateAll(ctx: EthContext, triggerEvent: string) {
  const snapshots = await ctx.store.list(AccountSnapshot, []);
  const newSnapshots = await Promise.all(
    snapshots.map((s) =>
      processAccount(ctx, s.id.toString(), s, triggerEvent)
    )
  );
  await ctx.store.upsert(newSnapshots);
}
```

---

## Protocol-Specific Templates

### A. Simple Holding (ERC20 Balance Tracking)

For: direct holding of any ERC20 token

**store.graphql:**
```graphql
type AccountSnapshot @entity {
  id: String!
  timestampMilli: BigInt!
  balance: BigInt!
}
```

**config.ts:**
```typescript
import { EthChainId } from "@sentio/sdk/eth";
export const NETWORK = EthChainId.ETHEREUM;
export const TOKEN_ADDRESS = "0x...";
export const TOKEN_DECIMALS = 18;
export const DAILY_POINTS = 1000;
export const MULTIPLIER = 1;
```

**processor.ts:**
```typescript
import { GLOBAL_CONFIG } from "@sentio/runtime";
import { BigDecimal } from "@sentio/sdk";
import { EthContext, isNullAddress } from "@sentio/sdk/eth";
import { ERC20Processor } from "@sentio/sdk/eth/builtin";
import { AccountSnapshot } from "./schema/store.js";
import { NETWORK, TOKEN_ADDRESS, TOKEN_DECIMALS, DAILY_POINTS, MULTIPLIER } from "./config.js";

GLOBAL_CONFIG.execution = { sequential: true };
const MILLISECOND_PER_DAY = 60 * 60 * 1000 * 24;

ERC20Processor.bind({ address: TOKEN_ADDRESS, network: NETWORK })
  .onEventTransfer(async (event, ctx) => {
    const { from, to } = event.args;
    if (from == to) return;
    const accounts = [from, to].filter((a) => !isNullAddress(a));
    const snapshots = await Promise.all(
      accounts.map((a) => processAccount(ctx, a, undefined, event.name))
    );
    await ctx.store.upsert(snapshots);
  })
  .onTimeInterval(async (_, ctx) => {
    await updateAll(ctx, "TimeInterval");
  }, 60, 60);

async function updateAll(ctx: EthContext, triggerEvent: string) {
  const snapshots = await ctx.store.list(AccountSnapshot, []);
  const newSnapshots = await Promise.all(
    snapshots.map((s) => processAccount(ctx, s.id.toString(), s, triggerEvent))
  );
  await ctx.store.upsert(newSnapshots);
}

async function processAccount(
  ctx: EthContext, account: string,
  snapshot: AccountSnapshot | undefined, triggerEvent: string
) {
  if (!snapshot) snapshot = await ctx.store.get(AccountSnapshot, account);
  const points = snapshot ? calcPoints(ctx, snapshot) : new BigDecimal(0);
  const newBalance = await ctx.contract.balanceOf(account);

  const newSnapshot = new AccountSnapshot({
    id: account,
    timestampMilli: BigInt(ctx.timestamp.getTime()),
    balance: newBalance,
  });

  ctx.eventLogger.emit("point_update", {
    account, points, triggerEvent,
    snapshotTimestampMilli: snapshot?.timestampMilli ?? 0n,
    snapshotBalance: snapshot?.balance.scaleDown(TOKEN_DECIMALS) ?? 0,
    newTimestampMilli: newSnapshot.timestampMilli,
    newBalance: newSnapshot.balance.scaleDown(TOKEN_DECIMALS),
    multiplier: MULTIPLIER,
  });
  return newSnapshot;
}

function calcPoints(ctx: EthContext, snapshot: AccountSnapshot): BigDecimal {
  const nowMilli = ctx.timestamp.getTime();
  if (nowMilli <= Number(snapshot.timestampMilli)) return new BigDecimal(0);
  const deltaDay = (nowMilli - Number(snapshot.timestampMilli)) / MILLISECOND_PER_DAY;
  return snapshot.balance.scaleDown(TOKEN_DECIMALS)
    .multipliedBy(deltaDay).multipliedBy(DAILY_POINTS).multipliedBy(MULTIPLIER);
}
```

### B. Lending Protocol (Aave / Aave Fork)

For: Aave, Zerolend, Seamless, etc. using aToken + debtToken

**Key difference:** Track both supply (aToken) and borrow (debtToken). Net position = supply - borrow.

**store.graphql:**
```graphql
type AccountSnapshot @entity {
  id: String!
  timestampMilli: BigInt!
  supplyBalance: BigInt!
  borrowBalance: BigInt!
}
```

**config.ts:**
```typescript
import { EthChainId } from "@sentio/sdk/eth";
export const NETWORK = EthChainId.ETHEREUM;
export const ATOKEN = "0x...";      // aToken address (supply receipt)
export const DEBT_TOKEN = "0x...";  // variable debt token address
export const TOKEN_DECIMALS = 8;
export const DAILY_POINTS = 1000;
export const MULTIPLIER = 3;
```

**processor.ts key parts:**
```typescript
// Bind Transfer events on both tokens
ERC20Processor.bind({ address: ATOKEN, network: NETWORK })
  .onEventTransfer(async (event, ctx) => {
    const { from, to } = event.args;
    if (from == to) return;
    const accounts = [from, to].filter(a => !isNullAddress(a));
    await ctx.store.upsert(
      await Promise.all(accounts.map(a => processAccount(ctx, a, undefined, event.name)))
    );
  })
  .onTimeInterval(async (_, ctx) => {
    await updateAll(ctx, "TimeInterval");
  }, 60, 60);

ERC20Processor.bind({ address: DEBT_TOKEN, network: NETWORK })
  .onEventTransfer(async (event, ctx) => {
    const { from, to } = event.args;
    if (from == to) return;
    const accounts = [from, to].filter(a => !isNullAddress(a));
    await ctx.store.upsert(
      await Promise.all(accounts.map(a => processAccount(ctx, a, undefined, event.name)))
    );
  });

// Query both contracts for balance
async function getAccountSnapshot(ctx: EthContext, account: string) {
  const [supply, borrow] = await Promise.all([
    getATokenContractOnContext(ctx, ATOKEN).balanceOf(account),
    getVariableDebtTokenContractOnContext(ctx, DEBT_TOKEN).balanceOf(account),
  ]);
  return new AccountSnapshot({
    id: account,
    timestampMilli: BigInt(ctx.timestamp.getTime()),
    supplyBalance: supply,
    borrowBalance: borrow,
  });
}

// Points: net = supply - borrow
function calcPoints(ctx: EthContext, snapshot: AccountSnapshot): BigDecimal {
  const nowMilli = ctx.timestamp.getTime();
  if (nowMilli <= Number(snapshot.timestampMilli)) return new BigDecimal(0);
  const deltaDay = (nowMilli - Number(snapshot.timestampMilli)) / MILLISECOND_PER_DAY;
  const netBalance = snapshot.supplyBalance - snapshot.borrowBalance;
  return (netBalance > 0n ? netBalance : 0n)
    .scaleDown(TOKEN_DECIMALS)
    .multipliedBy(deltaDay).multipliedBy(DAILY_POINTS).multipliedBy(MULTIPLIER);
}
```

### C. Morpho (Isolated Lending Markets)

For: Morpho Blue markets, each with its own marketId

**Key difference:** Query positions via Morpho main contract `position(marketId, account)`, not via token transfer balances.

```typescript
// Use collateral token Transfer to discover users, but balance comes from main contract
ERC20Processor.bind({ address: COLLATERAL_TOKEN, network: NETWORK })
  .onEventTransfer(async (event, ctx) => { ... })
  .onTimeInterval(async (_, ctx) => { ... }, 60, 60);

async function getAccountSnapshot(ctx: EthContext, account: string) {
  const morpho = getMorphoContractOnContext(ctx, MORPHO_ADDRESS);
  const position = await morpho.position(MARKET_ID, account);
  return new AccountSnapshot({
    id: account,
    timestampMilli: BigInt(ctx.timestamp.getTime()),
    supplyBalance: position.supplyShares,  // or use collateral
    borrowBalance: position.borrowShares,
  });
}
```

### D. Vault / LP Token (Curve / ERC4626 / BoringVault)

For: Curve pools, ERC4626 vaults, BoringVault, etc. where LP token represents a share of underlying assets.

**Key difference:** User holds LP token. Underlying balance = (LP balance / LP total supply) * target token in pool.

```typescript
ERC20Processor.bind({ address: LP_TOKEN, network: NETWORK })
  .onEventTransfer(...)
  .onTimeInterval(..., 60, 60);

async function getBalance(ctx: EthContext, account: string): Promise<bigint> {
  const [lpBalance, lpTotalSupply, underlyingTotal] = await Promise.all([
    ctx.contract.balanceOf(account),
    ctx.contract.totalSupply(),
    getERC20ContractOnContext(ctx, TARGET_TOKEN).balanceOf(POOL_ADDRESS),
  ]);
  if (lpTotalSupply === 0n) return 0n;
  return (underlyingTotal * lpBalance) / lpTotalSupply;
}
```

**Multi-token pool (Curve multi-asset):**
```typescript
async function getBalance(ctx: EthContext, account: string) {
  const [lpBalance, lpTotalSupply, token0InPool, token1InPool] = await Promise.all([
    ctx.contract.balanceOf(account),
    ctx.contract.totalSupply(),
    getERC20ContractOnContext(ctx, TOKEN0).balanceOf(POOL),
    getERC20ContractOnContext(ctx, TOKEN1).balanceOf(POOL),
  ]);
  if (lpTotalSupply === 0n) return 0n;
  const share0 = (token0InPool * lpBalance) / lpTotalSupply;
  const share1 = (token1InPool * lpBalance) / lpTotalSupply;
  return share0 + share1; // assumes same-asset 1:1 peg
}
```

### E. Concentrated Liquidity (Uniswap V3 / Aerodrome CL)

For: any NFT-position-based concentrated liquidity AMM

**Key differences:**
- Use tokenId (NFT) as snapshot key instead of address
- Requires Uniswap V3 SDK to calculate token amounts in position
- Track gauge staking ownership changes

**store.graphql:**
```graphql
type PositionSnapshot @entity {
  id: String!
  owner: String!
  tickLower: BigInt!
  tickUpper: BigInt!
  timestampMilli: BigInt!
  token0Balance: BigDecimal!
  token1Balance: BigDecimal!
}

type StakedPositionSnapshot @entity {
  id: String!
  owner: String!
}
```

**processor.ts key parts:**
```typescript
import { Pool, Position } from "@uniswap/v3-sdk";
import { Token } from "@uniswap/sdk-core";

NonfungiblePositionManagerProcessor.bind({
  network: NETWORK,
  address: NFT_MANAGER,
  startBlock: START_BLOCK,
})
  .onEventIncreaseLiquidity(async (event, ctx) => {
    const tokenId = event.args.tokenId.toString();
    if (!(await isTargetNFT(ctx, tokenId))) return;
    await processPosition(ctx, tokenId, event.name);
  })
  .onEventDecreaseLiquidity(async (event, ctx) => {
    const tokenId = event.args.tokenId.toString();
    const snapshot = await ctx.store.get(PositionSnapshot, tokenId);
    if (!snapshot) return;
    await processPosition(ctx, tokenId, event.name);
  })
  .onEventTransfer(async (event, ctx) => {
    if ([event.args.from, event.args.to].some(isNullAddress)) return;
    const tokenId = event.args.tokenId.toString();
    const snapshot = await ctx.store.get(PositionSnapshot, tokenId);
    if (!snapshot) return;
    await processPosition(ctx, tokenId, event.name);
  })
  .onTimeInterval(async (_, ctx) => {
    const snapshots = await ctx.store.list(PositionSnapshot, []);
    await Promise.all(
      snapshots.map((s) => processPosition(ctx, s.id, "TimeInterval"))
    );
  }, 60, 60);

// Gauge staking tracking (owner becomes gauge contract when staked)
CLGaugeProcessor.bind({ network: NETWORK, address: GAUGE_ADDRESS })
  .onEventDeposit(async (event, ctx) => {
    await ctx.store.upsert(new StakedPositionSnapshot({
      id: event.args.tokenId.toString(),
      owner: event.args.user,
    }));
  })
  .onEventWithdraw(async (event, ctx) => {
    await ctx.store.delete(StakedPositionSnapshot, event.args.tokenId.toString());
  });

// Calculate token amounts in position
async function getPositionAmounts(ctx: EthContext, tokenId: string) {
  const { tickLower, tickUpper, liquidity } = await getPositionInfo(ctx, tokenId);
  const pool = await getPool(ctx);
  const position = new Position({ pool, tickLower, tickUpper, liquidity });
  return {
    token0Balance: BigDecimal(position.amount0.toFixed()),
    token1Balance: BigDecimal(position.amount1.toFixed()),
  };
}

// Resolve position owner (account for staking)
async function getPositionOwner(ctx: EthContext, tokenId: string) {
  const staked = await ctx.store.get(StakedPositionSnapshot, tokenId);
  if (staked) return staked.owner;
  return await getNFTManager(ctx).ownerOf(tokenId);
}
```

### F. Pendle (Multi-layer Token Structure)

For: Pendle's SY / YT / LP / PT token system

**Key differences:**
- Multiple contracts bound simultaneously (SY, YT, LP, plus third-party liquid lockers)
- YT balance computed via implied holding
- LP shares must account for liquid locker boost
- Uses handler delegation to separate business logic

**processor.ts structure:**
```typescript
// SY token (Standardized Yield)
ERC20Processor.bind({ address: SY, network, startBlock: SY_START_BLOCK, name: "SY" })
  .onEventTransfer(async (evt, ctx) => await handleSYTransfer(evt, ctx));

// YT token (Yield Token)
PendleYieldTokenProcessor.bind({ address: YT, network, startBlock, name: "YT" })
  .onEventTransfer(async (evt, ctx) => await handleYTTransfer(evt, ctx))
  .onTimeInterval(async (_, ctx) => await processAllYTAccounts(ctx), 60, 60);

// LP token (Market)
PendleMarketProcessor.bind({ address: LP, network, startBlock, name: "LP" })
  .onEventTransfer(async (evt, ctx) => await handleLPTransfer(evt, ctx))
  .onEventSwap(async (evt, ctx) => await handleMarketSwap(evt, ctx));

// Third-party Liquid Locker (Equilibria / Penpie) staking events
EQBBaseRewardProcessor.bind({ address: EQB_RECEIPT_TOKEN, network, startBlock })
  .onEventStaked(async (evt, ctx) =>
    await processAllLPAccounts(ctx, [evt.args._user.toLowerCase()])
  )
  .onEventWithdrawn(async (evt, ctx) =>
    await processAllLPAccounts(ctx, [evt.args._user.toLowerCase()])
  );
```

**YT implied holding calculation:**
```typescript
async function getUnderlyingPerYt(ctx: EthContext, market: string): Promise<number> {
  const c = getPendleMarketV3ContractOnContext(ctx, market);
  const [storage, expiry] = await Promise.all([c._storage(), c.expiry()]);

  const lastLnImpliedRate = Number(storage.lastLnImpliedRate) / 1e18;
  const lastImpliedRate = Math.exp(lastLnImpliedRate);
  const timeToMaturity = Number(expiry) - Date.now() / 1000;
  const oneYear = 365 * 24 * 60 * 60;
  const assetToPtPrice = Math.pow(lastImpliedRate, timeToMaturity / oneYear);
  return 1 - 1 / assetToPtPrice;
}
```

### G. Compound Fork (cToken Model)

For: Compound V2, Moonwell, Benqi, Silo

**Key difference:** cToken uses exchangeRate to convert cToken amount to underlying asset amount.

```typescript
async function getBalance(ctx: EthContext, account: string): Promise<bigint> {
  const cToken = getCTokenContractOnContext(ctx, CTOKEN_ADDRESS);
  const [balance, rate] = await Promise.all([
    cToken.balanceOf(account),
    cToken.exchangeRateStored(),
  ]);
  // cToken balance * exchange rate / 1e18 = underlying amount
  return (balance * rate) / BigInt(1e18);
}
```

### H. ERC721 / NFT Vault (e.g. Fluid Smart Collateral)

For: protocols that manage positions via NFTs

**Key difference:** NFT transfer triggers user registration, time interval polls positions.

```typescript
ERC721Processor.bind({ network: NETWORK, address: NFT_ADDRESS })
  .onEventTransfer(async (event, ctx) => {
    const account = event.args.to;
    if (isNullAddress(account)) return;
    await ctx.store.upsert(new Account({ id: account }));
    await processAccount(ctx, account);
  })
  .onTimeInterval(async (_, ctx) => {
    const accounts = await ctx.store.list(Account, []);
    await Promise.all(
      accounts.map((a) => processAccount(ctx, a.id.toString()))
    );
  }, 60, 60);
```

### I. Uniswap V2 / Constant Product AMM

For: Uniswap V2, PancakeSwap V2, SushiSwap, or any constant product pair

**Key differences:**
- LP token is the pair contract itself
- Use `getReserves()` to get pool balances, user's share = `(reserve * lpBalance) / totalSupply`
- Listen to `Sync` and `Swap` events to update all positions when pool ratio changes
- Composite ID `{poolAddress}.{account}` when tracking multiple pools

**store.graphql:**
```graphql
type AccountSnapshot @entity {
  id: String!
  account: String!
  poolAddress: String!
  timestampMilli: BigInt!
  balance: BigDecimal!
}
```

**config.ts:**
```typescript
import { EthChainId } from "@sentio/sdk/eth";

export interface PoolInfo {
  address: string;
  token0: string;
  token0Decimals: number;
  token1: string;
  token1Decimals: number;
}

export const NETWORK = EthChainId.ETHEREUM;
export const TARGET_TOKEN = "0x...";
export const TOKEN_DECIMALS = 18;
export const DAILY_POINTS = 1000;
export const MULTIPLIER = 1;

const POOL_ADDRESSES = ["0x..."];

export const configs: PoolInfo[] = await Promise.all(
  POOL_ADDRESSES.map(async (address) => {
    const c = getPairContract(NETWORK, address);
    const [token0, token1] = await Promise.all([c.token0(), c.token1()]);
    return { address, token0, token0Decimals: 18, token1, token1Decimals: 18 };
  })
);

export function isTargetToken(address: string) {
  return address.toLowerCase() === TARGET_TOKEN.toLowerCase();
}
```

**processor.ts:**
```typescript
configs.forEach((config) =>
  PairProcessor.bind({ network: NETWORK, address: config.address })
    .onEventTransfer(async (event, ctx) => {
      const { from, to } = event.args;
      if (from == to) return;
      const accounts = [from, to].filter((a) => !isNullAddress(a));
      const snapshots = await ctx.store.list(AccountSnapshot, [
        { field: "poolAddress", op: "=", value: ctx.address },
      ]);
      const newSnapshots = await Promise.all([
        ...snapshots
          .filter((s) => !accounts.includes(s.account))
          .map((s) => processAccount(ctx, s.account, config, s, event.name)),
        ...accounts.map((a) => processAccount(ctx, a, config, undefined, event.name)),
      ]);
      await ctx.store.upsert(newSnapshots);
    })
    .onEventSync(async (event, ctx) => {
      await updateAllForPool(ctx, config, event.name);
    })
    .onEventSwap(async (event, ctx) => {
      await updateAllForPool(ctx, config, event.name);
    })
    .onTimeInterval(async (_, ctx) => {
      await updateAllForPool(ctx, config, "TimeInterval");
    }, 60, 60)
);

async function updateAllForPool(ctx: EthContext, config: PoolInfo, triggerEvent: string) {
  const snapshots = await ctx.store.list(AccountSnapshot, [
    { field: "poolAddress", op: "=", value: ctx.address },
  ]);
  const newSnapshots = await Promise.all(
    snapshots.map((s) => processAccount(ctx, s.account, config, s, triggerEvent))
  );
  await ctx.store.upsert(newSnapshots);
}

async function processAccount(ctx: EthContext, account: string, config: PoolInfo, snapshot: AccountSnapshot | undefined, triggerEvent: string) {
  const id = config.address + "." + account;
  if (!snapshot) snapshot = await ctx.store.get(AccountSnapshot, id);
  const points = snapshot ? calcPoints(ctx, snapshot) : new BigDecimal(0);

  const [lpBalance, totalSupply, reserves] = await Promise.all([
    ctx.contract.balanceOf(account),
    ctx.contract.totalSupply(),
    ctx.contract.getReserves(),
  ]);
  const reserveTarget = isTargetToken(config.token0) ? reserves._reserve0 : reserves._reserve1;
  const newBalance = (reserveTarget * lpBalance)
    .scaleDown(TOKEN_DECIMALS)
    .div(totalSupply.asBigDecimal());

  const newSnapshot = new AccountSnapshot({
    id,
    account,
    poolAddress: config.address,
    timestampMilli: BigInt(ctx.timestamp.getTime()),
    balance: newBalance,
  });

  ctx.eventLogger.emit("point_update", {
    account, points, triggerEvent,
    poolAddress: config.address,
    snapshotBalance: snapshot?.balance ?? new BigDecimal(0),
    newBalance,
  });
  return newSnapshot;
}
```

### J. Uniswap V4 (Singleton Pool with PositionManager)

For: Uniswap V4 pools using the singleton architecture

**Key differences:**
- Single `PositionManager` contract manages all positions (no per-pool NFT manager)
- Pool identified by `poolId` (bytes32), position info encoded in a packed uint256
- Use `StateView` contract to read pool state (slot0)
- Position info decoded from packed bitfield: `poolId(200) | tickUpper(24) | tickLower(24) | hasSubscriber(8)`

**store.graphql:**
```graphql
type PositionSnapshot @entity {
  id: String!
  owner: String!
  tickLower: BigInt!
  tickUpper: BigInt!
  timestampMilli: BigInt!
  token0Balance: BigDecimal!
  token1Balance: BigDecimal!
}
```

**config.ts:**
```typescript
import { EthChainId } from "@sentio/sdk/eth";
import { getPositionManagerContract } from "./types/eth/positionmanager.js";

export const NETWORK = EthChainId.ETHEREUM;
export const POOL_ID = "0x...";
export const POSITION_MANAGER = "0x...";
export const STATE_VIEW = "0x...";
export const START_BLOCK = 12345678;
export const DAILY_POINTS = 1000;
export const MULTIPLIER = 1;

export const {
  fee: FEE,
  tickSpacing: TICK_SPACING,
  currency0: TOKEN0,
  currency1: TOKEN1,
} = await getPositionManagerContract(NETWORK, POSITION_MANAGER).poolKeys(
  POOL_ID.slice(0, 52)
);
```

**util.ts:**
```typescript
export function decodePositionInfo(info: bigint) {
  const tickLower = BigInt.asIntN(24, (info >> 8n) & ((1n << 24n) - 1n));
  const tickUpper = BigInt.asIntN(24, (info >> 32n) & ((1n << 24n) - 1n));
  const poolId = "0x" + (info >> 56n).toString(16);
  return { poolId, tickUpper, tickLower };
}
```

**processor.ts:**
```typescript
import { Pool, Position } from "@uniswap/v3-sdk";
import { Token } from "@uniswap/sdk-core";

const NULL = "0x0000000000000000000000000000000000000000";

PositionManagerProcessor.bind({
  network: NETWORK,
  address: POSITION_MANAGER,
  startBlock: START_BLOCK,
})
  .onEventTransfer(async (event, ctx) => {
    const { info } = await ctx.contract.getPoolAndPositionInfo(event.args.id);
    const { poolId, tickUpper, tickLower } = decodePositionInfo(info);
    if (poolId != POOL_ID.slice(0, 52)) return;

    if (event.args.to == NULL) {
      await ctx.store.delete(PositionSnapshot, event.args.id.toString());
      return;
    }
    await ctx.store.upsert(new PositionSnapshot({
      id: event.args.id.toString(),
      owner: event.args.to,
      tickLower,
      tickUpper,
    }));
  })
  .onTimeInterval(async (_, ctx) => {
    await updateAll(ctx, "TimeInterval");
  }, 60, 60);

async function updateAll(ctx: EthContext, triggerEvent: string) {
  const tokenA = new Token(1, TOKEN0, 18, "token0");
  const tokenB = new Token(1, TOKEN1, 18, "token1");
  const { sqrtPriceX96, tick } = await getStateViewContractOnContext(ctx, STATE_VIEW)
    .getSlot0(POOL_ID);
  const pool = new Pool(tokenA, tokenB, Number(FEE), sqrtPriceX96.toString(), 0, Number(tick));

  const snapshots = await ctx.store.list(PositionSnapshot);
  await Promise.all(
    snapshots.map((snapshot) => processPosition(ctx, snapshot, pool, triggerEvent))
  );
}

async function processPosition(ctx: EthContext, snapshot: PositionSnapshot, pool: Pool, triggerEvent: string) {
  const liquidity = await getPositionManagerContractOnContext(ctx, POSITION_MANAGER)
    .getPositionLiquidity(snapshot.id.toString());
  const position = new Position({
    pool,
    liquidity: liquidity.toString(),
    tickLower: Number(snapshot.tickLower),
    tickUpper: Number(snapshot.tickUpper),
  });
  const token0Balance = BigDecimal(position.amount0.toFixed());
  const token1Balance = BigDecimal(position.amount1.toFixed());

  ctx.eventLogger.emit("point_update", {
    account: snapshot.owner,
    token0Balance,
    token1Balance,
    triggerEvent,
  });
}
```

---

## Points-Specific Best Practices

1. **Always `sequential: true`** -- prevent state races in snapshot updates
2. **Always filter null addresses** -- `isNullAddress()` check for mint/burn
3. **Always skip self-transfers** -- `if (from == to) return;`
4. **Always validate timestamps** -- prevent backwards time and same-block double counting
5. **Use `Promise.all`** -- parallel processing for multiple accounts and contract calls
6. **Use `BigDecimal`** -- avoid floating point precision loss
7. **Use `scaleDown()`** -- convert raw bigint to human-readable decimal
8. **Emit full before/after state** -- log both old and new snapshot values in eventLogger
9. **Filter protocol addresses** -- skip contracts, gauges, treasuries:
```typescript
function isProtocolAddress(address: string) {
  const PROTOCOL_ADDRESSES = new Set([
    "0x...", // gauge contract
    "0x...", // treasury
  ].map(a => a.toLowerCase()));
  return isNullAddress(address) || PROTOCOL_ADDRESSES.has(address.toLowerCase());
}
```
10. **Use `startBlock`** -- avoid processing irrelevant historical data
11. **`onTimeInterval(60, 60)`** -- standard 1-hour polling cycle
12. **Extract `updateAll` helper** -- keep `onTimeInterval` callbacks clean
