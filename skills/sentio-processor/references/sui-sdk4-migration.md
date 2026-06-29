# Migrating a Sui processor to SDK 4: gRPC vs JSON-RPC data shapes

Use this when **upgrading a Sui processor to Sentio SDK 4**, or writing one that touches **raw** transaction / object / event structures (reading fields off `ctx.transaction`, walking programmable-transaction commands, decoding an object from `ctx.client.getObject`, or parsing Move type strings).

## SDK 4 is a multi-chain major — not just Sui

Most of this doc covers the Sui gRPC raw-data shapes, which are the largest source of processor code changes. But SDK 4 is a breaking major across the board, so an upgrade also involves:

- **Bump deps to `^4.0.0`.** Set `@sentio/sdk`, `@sentio/cli` (and `@sentio/action` if used) to `^4.0.0` (the `latest` npm tag is now `4.x`), then `sentio gen` to regenerate bindings.
- **TypeScript 6 / `strictFunctionTypes` — the silent store-killer.** SDK 4's `sentio create` template pins `typescript@^6.0.3`, where `strict` (hence `strictFunctionTypes`) is on by default. The generated entity classes rely on `strictFunctionTypes` for `ctx.store.get(MyEntity, id)` to infer the concrete entity type; without it, `get`'s generic falls back to its `AbstractEntity` constraint, so `(await ctx.store.get(MyEntity, id))?.someField` fails to type-check in **every** store-using project (`Property 'someField' does not exist on type 'AbstractEntity'`). If you bump the SDK but stay on TypeScript 5.x, add `"strictFunctionTypes": true` to each project's `tsconfig.json`. (Going straight to TS 6 without `"strict": false` also flips on `strictPropertyInitialization` → TS2564 on generated entity fields, and `useUnknownInCatchVariables` → `catch (e) { e.message }` breaks; enabling only `strictFunctionTypes` on TS 5.x is the lowest-churn fix.)
- **Drop the `@sentio/ethers` fork.** Remove any `resolutions`/`overrides` pinning `@sentio/ethers`; SDK 4 uses upstream `ethers` 6. Side effect: some non-standard receipt fields are gone from ethers' types — e.g. an OP-stack L2 `ctx.transactionReceipt.l1Fee` now needs a cast (`(ctx.transactionReceipt as any)?.l1Fee`).
- **Solana migrated to `@anchor-lang/core` + `@solana/kit`.** If your processor decodes instructions with an Anchor coder, import `BorshInstructionCoder` / `Idl` / `Instruction` from `@sentio/sdk/solana` instead of `@coral-xyz/anchor` (`import { BorshInstructionCoder, type Idl } from '@sentio/sdk/solana'`).
- **Proto/RPC stack swapped to protobuf-es + connect-es** (from ts-proto + nice-grpc). This is internal to the runtime, but if you import directly from `@sentio/protos` (e.g. constructing proto messages in tests), the API changed: `create(FooSchema, init)` instead of `Foo.fromPartial`, `toBinary`/`fromBinary`/`fromJson(FooSchema, …)` for (de)serialization, and `oneof` fields are discriminated unions.
- **The v3 back-compat layer is gone.** The SDK 4 runtime no longer patches old (v3) processor shapes and drops deprecated proto fields, so a v3 processor won't run unmodified on a v4 driver — you must actually migrate, not just rely on the runtime papering over differences.

The rest of this doc is the Sui-specific deep dive; the cross-chain gotchas below were verified across a ~250-project monorepo migration.

## Cross-chain gotchas (verified in a large migration)

### Re-fetch Sui (and Fuel) ABIs — the cache is the wrong format
SDK 4 Sui codegen consumes ABIs in the new gRPC **array** format. Old cached `abis/sui/*.json` (the JSON-RPC normalized-module shape — often an object keyed by module name) make `sentio gen` throw `TypeError: modules.map is not a function` or `Cannot read properties of undefined (reading 'datatypes')`. Fix:
1. `rm -f abis/sui/*.json`, then `sentio gen` — re-downloads every `sentio.yaml` address (plus dependents) in the new format. Needs network.
2. **Named modules** added via `sentio add --name foo <addr>` live in `abis/sui/foo.json`, are imported as `./types/sui/foo.js`, and are usually **not** listed in `sentio.yaml` — so step 1 won't regenerate them. Re-add each: `sentio add --chain sui_mainnet --name foo <package-address>` (the package address is inside the old cached ABI). Codegen runs `modules.map` over *every* file in `abis/sui`, so it keeps failing until all stale files are refreshed.

Fuel has the analogous problem: an old Fuel ABI JSON lacking `specVersion` / `encodingVersion` / `programType` / `concreteTypes` / `metadataTypes` breaks Fuel codegen and must be regenerated from the Sway source (there may be no on-chain address to re-fetch from).

### Aptos: `MoveFetchConfig` is a protobuf-es message now
A literal like `const cfg: MoveFetchConfig = { resourceChanges: true, resourceConfig: { moveTypePrefix } }` fails because the nested `ResourceConfig` is a protobuf-es `Message` requiring `$typeName`. Either drop the explicit `: MoveFetchConfig` annotation and add `$typeName: "processor.ResourceConfig"` to the nested object (handler params accept `HandlerOptions<MoveFetchConfig, T>`, which already omits the *top-level* `$typeName`), or build it with `create(MoveFetchConfigSchema, {…})` (`create` from `@bufbuild/protobuf`, `MoveFetchConfigSchema` from `@sentio/protos`).

### Tests: `processBindings` data is a protobuf-es oneof
Raw test bindings change from the old ts-proto key style to the discriminated-union style:
```ts
// before
data: { aptEvent: { rawTransaction, rawEvent, eventIndex } }
// after
data: { value: { case: "aptEvent", value: { rawTransaction, rawEvent, eventIndex } } }
```

### More Sui `ctx.client` changes (beyond the tables below)
- `getCoinMetadata({ coinType })` now returns `{ coinMetadata: CoinMetadata | null }` — read `.coinMetadata?.symbol` / `.decimals` / `.name` (the metadata was returned directly before).
- `getObject(...)` returns `{ object }`; read `obj.object.type` and `obj.object.json` (see the Object section below).
- `multiGetObjects({ ids, options })` → `getObjects({ objectIds, include })`, returning `{ objects }`.
- `getDynamicFieldObject` → `getDynamicField`.
- `@mysten/sui/client` no longer exports `SuiMoveObject` / `SuiObjectChange` / `SuiTransactionBlockResponse`; the typed object input `SuiMoveObjectInput` has no `.fields` (use `.json`, or `data_decoded` for typed handlers).

### Watch out for `// @ts-nocheck`
A file with `// @ts-nocheck` passes `tsc` even when its raw-data access is still in the old JSON-RPC shape — so it compiles but returns `undefined` / throws at runtime. When migrating, grep the whole repo for `getObject({ id`, `.data.content.fields`, and `.parsedJson` — not just the files `tsc` flags.

## Overview

On **SDK 4**, Sui processors fetch on-chain data over **gRPC**. The decoded values from typed handlers are unchanged — the SDK normalizes them. But any code that reaches into **raw** transaction, event, or object structures now sees the **gRPC protobuf shapes**, which differ from the older JSON-RPC shapes in field names, nesting, `oneof` unions, and string formatting.

If your processor parses raw transactions / objects / Move type strings, those are the spots that need changes. A processor that only uses typed decoded values usually needs no code change (just regenerate bindings).

## Authoritative schema — read the proto

The gRPC shapes are defined by the Sui RPC v2 protobufs. Treat the proto, not this doc, as the source of truth:

- Sentio's copy: https://github.com/sentioxyz/sui-apis/tree/main/proto/sui/rpc/v2
- Forked from upstream: https://github.com/MystenLabs/sui-apis

This is a fork of an upstream schema that evolves, so field names/shapes can change across versions — re-verify against the proto after an SDK bump.

## What does NOT change

- Typed event handlers: `onEventXxx((event, ctx) => …)` and `event.data_decoded.*`.
- Logic that only uses decoded values (amounts, metrics, store entities).

You only adapt code that reads **raw** structures (`ctx.transaction.*` beyond `data_decoded`, objects from `ctx.client`, type strings).

**Exception — `onObjectChange`:** the `changes` array passed to object-change handlers (`SuiObjectTypeProcessor.bind(...).onObjectChange((changes, ctx) => …)`) is no longer the JSON-RPC object shape — on SDK 4 it is the unified `SuiClientTypes.ChangedObject[]` from `@mysten/sui`. If your handler reads fields off those entries (object type, owner, id), re-check them against the unified type; `onEventXxx` decoded payloads are unaffected.

## protobuf `oneof` pattern

gRPC messages use protobuf `oneof` (discriminated unions). In TypeScript they appear as `{ oneofKind: "<case>", <case>: <value> }`. Always branch on `oneofKind` before reading the value:

```ts
const data = ctx.transaction.transaction?.kind?.data
const ptb = data?.oneofKind === "programmableTransaction" ? data.programmableTransaction : undefined
const moveCalls = (ptb?.commands ?? []).flatMap((c) =>
  c.command.oneofKind === "moveCall" ? [c.command.moveCall] : []
)
```

## Transaction shape: JSON-RPC → gRPC

| You want | JSON-RPC (earlier SDK) | gRPC (SDK 4) |
|---|---|---|
| PTB command list | `ctx.transaction.transaction.data.transaction.transactions` | `ctx.transaction.transaction.kind.data` where `oneofKind === "programmableTransaction"` → `.programmableTransaction.commands` |
| a MoveCall command | element `.MoveCall` (may be absent) | element `.command.oneofKind === "moveCall"` → `.moveCall` |
| MoveCall fields | `.package` / `.module` / `.function` / `.type_arguments` | `.package` / `.module` / `.function` / `.typeArguments` (camelCase) |
| events on the tx | `ctx.transaction.events` | `ctx.transaction.events?.events` |
| event type | `ev.type` | `ev.eventType` |
| event parsed payload | `ev.parsedJson` | `ev.json` |
| tx digest | `ctx.transaction.digest` | `ctx.transaction.digest` — may be `undefined`; guard before use |

## Object shape

Reading an object via `ctx.client.getObject(...)` returns the gRPC object:

```ts
const obj = await ctx.client.getObject({ objectId: id, include: { json: true } })
const type = obj.object.type        // full Move type string
const fields = obj.object.json      // decoded contents, e.g. obj.object.json?.fee_rate
```

Request decoded contents explicitly (`include: { json: true }`); read fields off `obj.object.json`, and the object's Move type off `obj.object.type`.

## Move type strings — parse structurally, not by text format

Raw type strings carry two gRPC-vs-JSON-RPC differences. Do **not** write parsers that assume the JSON-RPC text layout:

1. **Comma separator differs.** Between generic args, gRPC uses `,` (no space) → `Pool<A,B>`, while JSON-RPC uses `, ` (comma + space) → `Pool<A, B>`. Regexes that rely on whitespace will mis-split. This comes from the same two formatters as the address difference below: move's `StructTag::to_canonical_display` joins type args with `","` ([`language_storage.rs`](https://github.com/MystenLabs/sui/blob/main/external-crates/move/crates/move-core-types/src/language_storage.rs)), whereas Sui's `to_sui_struct_tag_string` joins with `", "` ([`sui_serde.rs`](https://github.com/MystenLabs/sui/blob/main/crates/sui-types/src/sui_serde.rs)).
   - Split top-level type args by **bracket depth** (only the comma at depth 0), which also handles nested generics.
2. **Address form differs by path — and the short-form set is a hardcoded list, not a numeric range.**
   - **gRPC** renders *every* type-tag address in **full 64-hex** (`StructTag::to_canonical_string(true)` — see [`crates/sui-rpc-api/src/grpc/v2/render.rs`](https://github.com/MystenLabs/sui/blob/main/crates/sui-rpc-api/src/grpc/v2/render.rs)). So SUI is `0x0000…0002::sui::SUI`.
   - **JSON-RPC** short-forms (leading zeros stripped) **only** the addresses in a hardcoded `SUI_ADDRESSES` set and renders all others full 64-hex — see `to_sui_struct_tag_string` in [`crates/sui-types/src/sui_serde.rs`](https://github.com/MystenLabs/sui/blob/main/crates/sui-types/src/sui_serde.rs):
     ```rust
     let address = if SUI_ADDRESSES.contains(&value.address) {
         value.address.short_str_lossless()                          // framework addrs → short (0x2)
     } else {
         value.address.to_canonical_string(/* with_prefix */ false)  // everything else → full 64-hex
     };
     ```
     `SUI_ADDRESSES` is exactly 7 framework/system addresses: `0x0`, `0x1`, `0x2` (framework), `0x3` (system), `0x5` (system state), `0x6` (clock), `0xdee9` (DeepBook). This is an explicit list — note DeepBook `0xdee9` is short despite being far above `0xf`, so it is **not** the "addresses 0x0–0xf are special" rule from other Move chains.

   So the same coin's type can be spelled differently across paths — `0x2::sui::SUI` (JSON-RPC) vs `0x0000…0002::sui::SUI` (gRPC) — and exact string equality is unreliable.
   - To compare or match type strings across paths, **normalize both sides** with `@mysten/sui`'s `normalizeStructTag` / `normalizeSuiAddress`, which pad every address to the full 64-hex form.
   - Do **not** hand-strip leading zeros to bridge the gap (it corrupts ordinary addresses, e.g. `0x027792…` → `0x27792…`), and don't assume a numeric "special range" — Sui uses the explicit `SUI_ADDRESSES` list above.

```ts
// Extract top-level generic type args from a struct type string,
// robust to comma spacing and nested generics.
function topLevelTypeArgs(type: string): string[] {
  const open = type.indexOf("<")
  const close = type.lastIndexOf(">")
  if (open < 0 || close < 0 || close < open) return []
  const inner = type.slice(open + 1, close)
  const args: string[] = []
  let depth = 0
  let start = 0
  for (let i = 0; i < inner.length; i++) {
    const ch = inner[i]
    if (ch === "<") depth++
    else if (ch === ">") depth--
    else if (ch === "," && depth === 0) {
      args.push(inner.slice(start, i).trim())
      start = i + 1
    }
  }
  args.push(inner.slice(start).trim())
  return args
}
```

## Upgrade checklist

- [ ] Run `sentio gen` to regenerate type bindings under SDK 4; commit the regenerated files.
- [ ] Find every place that reads **raw** `ctx.transaction` / events / objects (not `data_decoded`) and adapt to the gRPC shapes above (`oneof` branching, camelCase fields, `events.events`, `eventType`, `json`).
- [ ] Replace whitespace/format-dependent type-string parsing with depth-based splitting + explicit address normalization.
- [ ] Guard against `undefined` digests / optional fields.
- [ ] After any SDK bump, re-check field names against the proto — the schema is a fork of an evolving upstream.
