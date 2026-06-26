---
name: sui-grpc-data-format
description: Use when writing or upgrading a Sentio Sui processor on SDK 4 that touches RAW transaction / object / event structures — e.g. reading fields off `ctx.transaction`, walking programmable-transaction commands, decoding an object from `ctx.client.getObject`, or parsing Move type strings. On SDK 4 Sui chain data is fetched over gRPC, so these raw shapes follow the Sui gRPC protobuf schema and differ from the JSON-RPC shapes used by earlier SDK versions (field names, nesting, protobuf oneof unions, and type-string formatting). Not needed for typed handlers that only use `event.data_decoded`.
---

# Sui gRPC vs JSON-RPC raw data shapes (SDK 4)

## Overview

On **Sentio SDK 4**, Sui processors fetch on-chain data over **gRPC**. The decoded values from typed handlers are unchanged — the SDK normalizes them. But any code that reaches into **raw** transaction, event, or object structures now sees the **gRPC protobuf shapes**, which differ from the older JSON-RPC shapes in field names, nesting, `oneof` unions, and string formatting.

If you are upgrading a processor from an earlier SDK and it parses raw transactions / objects / Move type strings, those are the spots that need changes. A processor that only uses typed decoded values usually needs no code change (just regenerate bindings).

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

1. **No space after commas** in generic args. gRPC emits `Pool<A,B>`; JSON-RPC emitted `Pool<A, B>`. Regexes that relied on whitespace will mis-split.
   - Split top-level type args by **bracket depth** (only the comma at depth 0), which also handles nested generics.
2. **Address representation differs.** gRPC renders every address in the **full 32-byte (64-hex) form** — `0x` + 64 hex chars (Sui guarantees address outputs are the full 66-char string). Earlier JSON-RPC strings could spell the same address with leading zeros stripped (e.g. `0x2::sui::SUI` vs `0x0000…0002::sui::SUI`). So the **same coin can appear with different address spellings across SDK versions / data sources**, and exact string equality is not reliable.
   - If you need to compare type strings, or match against data produced under an older path, **normalize first** to a single canonical form. `@mysten/sui` provides `normalizeStructTag` / `normalizeSuiAddress` (both pad addresses to the full 64-hex form); run both sides through the same normalizer before comparing.

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
