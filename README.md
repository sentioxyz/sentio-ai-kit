# managing-sentio-processors

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin for building blockchain data processors with the [Sentio SDK](https://docs.sentio.xyz/).

## Install

### Claude Code Plugin (recommended)

```bash
# Add this repo as a marketplace source
/plugin marketplace add sentioxyz/sentio-skills

# Install the plugin
/plugin install managing-sentio-processors
```

### ClawHub

```bash
npx clawhub@latest install managing-sentio-processors
```

## What It Does

This plugin teaches Claude Code how to work with Sentio — a TypeScript blockchain data indexing platform. When you ask Claude Code to build a processor, it knows the full lifecycle: project setup, contract binding, handler patterns, testing, and deployment.

### Supported Chains

Ethereum, Aptos, Sui, Solana, Starknet, Bitcoin, Cosmos, Fuel, and IOTA.

### What It Covers

- **Project lifecycle** — `sentio create` → `sentio add` → `sentio gen` → write processor → `sentio test` → `sentio upload`
- **Processor patterns** — Event handlers, block/time intervals, transaction tracing for each chain
- **Metrics & events** — Counters, gauges, event logging with proper labeling
- **Store (database) API** — Entity definitions with `schema.graphql`, CRUD operations, sequential execution
- **Price feeds** — Token price lookups, USD value calculations, caching patterns
- **Testing** — `TestProcessorServer` with chain-specific test facets
- **DeFi patterns** — DEX/AMM, lending protocols, TVL tracking, points systems
- **Advanced patterns** — Multi-contract binding, GlobalProcessor, lazy caching, view calls

## Usage

Once installed, Claude Code automatically activates the skill when you work on Sentio projects. Just describe what you want:

```
"Create a Sentio processor that tracks USDC transfers on Ethereum"
"Add a Sui DEX swap tracker with volume metrics"
"Set up a points system for staking rewards"
```

## Plugin Structure

```
sentio-skills/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── managing-sentio-processors/
│       ├── SKILL.md
│       └── references/
│           ├── advanced-patterns.md
│           ├── defi-patterns.md
│           ├── store-and-points.md
│           └── production-examples.md
└── README.md
```

## Links

- [Sentio Documentation](https://docs.sentio.xyz/)
- [100+ Production Processor Examples](https://github.com/sentioxyz/sentio-processors)
- [ClawHub Page](https://clawhub.ai/Poytr1/managing-sentio-processors)
