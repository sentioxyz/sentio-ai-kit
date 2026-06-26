# Migrating a Sui processor to SDK 4: gRPC vs JSON-RPC data shapes

Use this when **upgrading a Sui processor to Sentio SDK 4**, or writing one that touches **raw** transaction / object / event structures (reading fields off `ctx.transaction`, walking programmable-transaction commands, decoding an object from `ctx.client.getObject`, or parsing Move type strings).

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
