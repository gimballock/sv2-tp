---
title: Hacking Bitcoin's Block Builder
author: Eric Price | eric.price@mara.com
sub_title: Pool Team @ Marathon Digital Holdings
---

## About This Talk

**Goal**: Show how Stratum v2 enables miner-provided custom templates

**Demo**: Testnet4 Docker playground for experimentation

**Audience**: Bitcoin developers curious about mining infrastructure

<!-- end_slide -->

## The Problem: Who Builds Blocks?

Traditionally, **pools** build block templates:

```
┌─────────────────────────────────────────────────────────┐
│                        POOL                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │  Mempool    │ -> │  Template   │ -> │    Jobs     │  │
│  │  Selection  │    │  Builder    │    │  to Miners  │  │
│  └─────────────┘    └─────────────┘    └─────────────┘  │
└─────────────────────────────────────────────────────────┘
                           ↓
                    ┌─────────────┐
                    │   Miners    │
                    │ (just hash) │
                    └─────────────┘
```

**Miners have no say in transaction selection!**

<!-- end_slide -->

## Why Does This Matter?

- **Censorship**: Pools can exclude transactions
- **MEV**: Pools can reorder for profit
- **Centralization**: Few entities control block content
- **Regulation**: Pools are easy targets for compliance pressure

<!-- end_slide -->

## Why Now? Recent Releases Made This Possible

**Bitcoin Core v30 (May 2025)**
- Mining interface via IPC (`-ipcbind=unix`)
- Clean separation of template generation

**Stratum v2 Reference Implementation v1.6 (Nov 2025)**
- Job Declaration protocol support added
- Pool, JDC, JDS, Translator all working together

**sv2-tp (this repo)**
- Standalone Template Provider for Bitcoin Core
- Docker playground for experimentation

**For the first time: all pieces are in place!**

<!-- end_slide -->

## Stratum v2: The Solution

Three sub-protocols working together:

```
┌────────────────────────────────────────────────────────┐
│                  STRATUM V2 PROTOCOLS                  │
├────────────────────────────────────────────────────────┤
│  1. Mining Protocol      - Jobs ↔ Shares               │
│  2. Template Distribution - Templates ↔ Jobs           │
│  3. Job Declaration      - Miner templates → Pool      │
└────────────────────────────────────────────────────────┘
```

<!-- end_slide -->

## Protocol 1: Mining Protocol

The basic mining loop - swapping jobs for proof-of-work shares

```
┌────────┐                                      ┌────────┐
│ MINER  │ <────────── jobs/shares ──────────>  │  POOL  │
└────────┘                                      └────────┘
```

<!-- end_slide -->

## Mining Protocol: With Translation

Legacy SV1 miners can participate via a translator proxy:

```
┌────────────┐          ┌────────────┐          ┌────────┐
│ OLD MINER  │ <- SV1 ->│ TRANSLATOR │ <- SV2 ->│  POOL  │
│   (SV1)    │          │   PROXY    │          │        │
└────────────┘          └────────────┘          └────────┘
```

<!-- end_slide -->

## Mining Protocol: Proxy Chains

Proxies can be chained for complex deployments:

```
┌────────┐   ┌───────┐   ┌───────┐   ┌───────┐   ┌────────┐
│ MINER  │<->│ PROXY │<->│ PROXY │<->│ PROXY │<->│  POOL  │
└────────┘   └───────┘   └───────┘   └───────┘   └────────┘
```

<!-- end_slide -->

## Mining Protocol: Fan-In Topology

Multiple miners aggregate through proxy hierarchy:

```
┌────────┐
│ MINER  │<->┐
└────────┘   │  ┌───────┐
┌────────┐   ├->│ PROXY │
│ MINER  │<->┘  └───────┘
└────────┘          |
                    │               ┌───────┐
┌────────┐          ├──────────────>│ PROXY │
│ MINER  │<->┐      │               └───────┘
└────────┘   │  ┌───────┐
┌────────┐   ├->│ PROXY │
│ MINER  │<->┘  └───────┘
└────────┘
```

<!-- end_slide -->

## Protocol 2: Template Distribution

Converts block templates into mining jobs

**Traditional model** (Stratum v1):

```
                    ┌────────────────────┐
                    │   BITCOIN CORE     │
                    │  (getblocktemplate)|
                    └────────┬───────────┘
                             │ RPC
                             ↓
┌────────┐           ┌───────────────┐
│ MINER  │ <-------> │     POOL      │
└────────┘           │ (builds jobs) │
                     └───────────────┘
```

Pool controls everything!

<!-- end_slide -->

## Template Distribution: Default (Pool-side)

Without custom templates, the pool runs its own TP:

```
                    ┌─────────────────────┐
                    │    BITCOIN CORE     │
                    └──────────┬──────────┘
                               │ IPC
                               ↓
                    ┌─────────────────────┐
                    │  TEMPLATE PROVIDER  │
                    └──────────┬──────────┘
                               │ Template Distribution
                               ↓
┌────────┐    Mining    ┌─────────────┐
│ MINER  │<------------>│    POOL     │
└────────┘   Protocol   └─────────────┘
```

Pool still controls templates, but now via a cleaner interface.

<!-- end_slide -->

## Why Separate the Template Provider?

Separating TP from Pool enables:

1. **Cleaner architecture** - Template logic isolated from pool coordination
2. **Push-based updates** - New templates sent immediately (not polled)
3. **Binary protocol** - No JSON serialization overhead
4. **Encrypted** - Noise Protocol instead of basic auth
5. **Foundation for miner templates** - The key step!

<!-- end_slide -->

## Protocol 3: Job Declaration

**The key innovation**: Miners can provide their own templates!

```
┌─────────────────────────────────────────────────────────────┐
│                      MINER SIDE                             │
│  ┌──────────────────┐    ┌─────────────────────────────┐    │
│  │ TEMPLATE PROVIDER│<-->│     JOB DECLARATOR CLIENT   │    │
│  │   (sv2-tp)       │    │  (declares custom templates)│    │
│  └──────────────────┘    └──────────────┬──────────────┘    │
│                                         │                   │
└─────────────────────────────────────────┼───────────────────┘
                                          │ Job Declaration
                                          ↓
┌─────────────────────────────────────────────────────────────┐
│                      POOL SIDE                              │
│  ┌──────────────────┐    ┌─────────────────────────────┐    │
│  │ TEMPLATE PROVIDER│<-->│     JOB DECLARATOR SERVER   │    │
│  │   (fallback)     │    │  (validates miner templates)│    │
│  └──────────────────┘    └──────────────┬──────────────┘    │
│                                         │                   │
│                          ┌──────────────┴──────────────┐    │
│                          │           POOL              │    │
│                          └─────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

<!-- end_slide -->

## Job Declaration: What Gets Negotiated?

The Job Declarator Server validates:

- **Credit attribution**: Proper payout addresses
- **Pool payout**: Pool's coinbase output is included
- **Block validity**: Transactions are valid, no double-spends

```
┌─────────────────────────────────────────────────────────┐
│              JOB DECLARATION MESSAGES                   │
├─────────────────────────────────────────────────────────┤
│  AllocateMiningJobToken    - Request to declare job     │
│  AllocateMiningJobToken.Success - Token granted         │
│  DeclareMiningJob          - Submit custom template     │
│  DeclareMiningJob.Success  - Template accepted          │
│  DeclareMiningJob.Error    - Template rejected          │
└─────────────────────────────────────────────────────────┘
```

<!-- end_slide -->

## Complete Architecture

```
              ┌──────────────┐              ┌──────────────┐
              │     TP       │              │     TP       │
              │  (primary)   │              │  (fallback)  │
              └──────┬───────┘              └──────┬───────┘
                     │ template                    │ template
                     │ distribution                │ distribution
                     ↓                             ↓
┌────────┐    ┌──────────────┐    mining    ┌──────────────┐
│ MINER  │<-->│    PROXY     │<------------>│    POOL      │
└────────┘    │    (JDC)     │              │    (JDS)     │
              └──────────────┘              └──────────────┘
                     ↑                             ↑
                     └──── job declaration ────────┘
```

**Miner controls template, Pool validates and coordinates**

<!-- end_slide -->

## The Template Provider (sv2-tp)

A standalone C++ application that:

1. Connects to Bitcoin Core via **IPC** (not RPC!)
2. Listens for Stratum v2 connections
3. Pushes new templates on:
   - New blocks
   - Fee increases (configurable threshold)

```cpp
// Key messages sent by Template Provider
Sv2NewTemplateMsg        // New block template
Sv2SetNewPrevHashMsg     // New tip found
```

<!-- end_slide -->

## Template Distribution Messages

```
┌─────────────────────────────────────────────────────────┐
│           TEMPLATE DISTRIBUTION PROTOCOL                │
├─────────────────────────────────────────────────────────┤
│  SetupConnection           - Handshake                  │
│  CoinbaseOutputConstraints - Space needed for payouts   │
│  NewTemplate               - Fresh block template       │
│  SetNewPrevHash            - New block found            │
│  RequestTransactionData    - Get full tx list           │
│  SubmitSolution            - Found a block!             │
└─────────────────────────────────────────────────────────┘
```

<!-- end_slide -->

## Why IPC Instead of RPC?

| Feature | getblocktemplate (RPC) | Template Provider (IPC) |
|---------|------------------------|-------------------------|
| Connection | Stateless polling | Stateful push |
| Encoding | JSON text | Binary |
| Security | Basic auth | Noise Protocol |
| Latency | Poll interval | Immediate push |
| Integration | Any language | Native C++ |

<!-- end_slide -->

## Demo: Testnet4 Docker Playground

```
docker/
├── docker-compose.yml      # Orchestration
├── Dockerfile              # Bitcoin Core + sv2-tp
└── config/
    ├── jdc-config.toml     # Job Declarator Client
    ├── jds-config.toml     # Job Declarator Server
    ├── pool-config.toml    # Pool
    └── translator-config.toml  # SV1 bridge
```

<!-- end_slide -->

## Docker Architecture

```
┌─────────────────────────────────────────────────────────┐
│  sv2-node (Bitcoin Core + sv2-tp)                       │
│    - Syncs testnet4, generates custom templates         │
└────────────────┬────────────────────────────────────────┘
                 │ Template Distribution
                 ↓
┌────────────────────────────┐    ┌───────────────────────┐
│  jd-client (JDC)           │    │  jd-server (JDS)      │
│  - Receives templates      │--->│  - Validates jobs     │
│  - Declares custom jobs    │    │  - RPC to sv2-node    │
└────────────────┬───────────┘    └───────────┬───────────┘
                 │                            │
                 └──────────┬─────────────────┘
                            ↓
              ┌─────────────────────────┐
              │  pool (SV2 Pool)        │
              │  - Coordinates mining   │
              │  - Fallback: hosted TP  │
              └─────────────┬───────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ↓                   ↓                   ↓
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  cpu-miner    │   │  translator   │   │  SV2 miners   │
│  (optional)   │   │  (for SV1)    │   │  (direct)     │
└───────────────┘   └───────┬───────┘   └───────────────┘
                            ↓
                    ┌───────────────┐
                    │ BitAxe / SV1  │
                    └───────────────┘
```

<!-- end_slide -->

## Quick Start

```bash
# Clone the repo
git clone https://github.com/gimballock/sv2-tp
cd sv2-tp/docker

# Start the stack
docker-compose up -d

# Wait for Bitcoin Core to sync (check progress)
docker-compose logs -f sv2-node

# Connect your miner!
# SV1 (BitAxe): stratum+tcp://localhost:34255
# SV2 native:   172.30.0.13:34254
```

<!-- end_slide -->

## Hacking the Template Builder

The magic happens in `src/sv2/template_provider.cpp`:

```cpp
// Template creation uses Bitcoin Core's CreateNewBlock()
block_template = m_mining.createNewBlock(block_create_options);

// You can modify transaction selection here!
// - Filter transactions
// - Reorder by your criteria
// - Add custom logic
```

<!-- end_slide -->

## How Core Selects Transactions

Bitcoin Core's `CreateNewBlock()` uses **ancestor feerate sorting**:

1. Calculate **ancestor feerate** for each mempool tx
   - (tx fee + ancestor fees) / (tx size + ancestor sizes)
2. Sort by ancestor feerate (highest first)
3. Add transactions until block is full (4M weight units)
4. Skip transactions that would exceed limits

```
┌─────────────────────────────────────────────┐
│  Mempool: ~300MB of unconfirmed txs         │
│     ↓ ancestor feerate sort                 │
│  Block: ~4MB (top ~3000-5000 txs)           │
└─────────────────────────────────────────────┘
```

<!-- end_slide -->

## Block Validity Constraints

Your custom template **MUST** satisfy:

| Constraint | Limit | Consequence |
|------------|-------|-------------|
| Block weight | ≤ 4,000,000 WU | Invalid block |
| Sigops | ≤ 80,000 | Invalid block |
| Coinbase maturity | Inputs ≥ 100 blocks old | Invalid block |
| No double-spends | Each UTXO spent once | Invalid block |
| Valid scripts | All inputs must validate | Invalid block |
| Topological order | Parents before children | Invalid block |

**JDS validates these before accepting your template!**

<!-- end_slide -->

## Constraint: Topological Ordering

Transactions must appear **after** their parents in the block:

```
WRONG:                      CORRECT:
┌─────────┐                 ┌─────────┐
│  Child  │  ← spends B     │ Parent  │  ← creates UTXO B
├─────────┤                 ├─────────┤
│ Parent  │  ← creates B    │  Child  │  ← spends B
└─────────┘                 └─────────┘
```

If you reorder transactions, you must maintain this!

<!-- end_slide -->

## Constraint: CPFP Packages

Child-Pays-For-Parent creates **dependencies**:

```
┌──────────────────────────────────────────┐
│ Parent tx: 1 sat/vB (too low alone)      │
│     ↓                                    │
│ Child tx: 100 sat/vB (pays for both)     │
│     = Package feerate: ~50 sat/vB        │
└──────────────────────────────────────────┘
```

**If you include the child, you MUST include the parent!**

Removing a child is safe. Removing a parent breaks the child.

<!-- end_slide -->

## Modification Difficulty: Easy

**Filtering transactions (removal only)**

- Remove inscriptions, OP_RETURN spam, specific patterns
- No reordering needed
- No dependency issues (just don't remove parents of included txs)

```cpp
// Easy: Skip transactions matching criteria
for (const auto& tx : block.vtx) {
    if (!isSpam(tx)) filtered.push_back(tx);
}
```

**Difficulty: ⭐ (1/5)**

<!-- end_slide -->

## Modification Difficulty: Medium

**Priority boosting (partial reorder)**

- Move specific transactions earlier in block
- Must maintain topological order
- Must recalculate if near weight limit

```cpp
// Medium: Boost priority of certain txs
std::stable_partition(txs.begin(), txs.end(), 
    [](const auto& tx) { return isPriority(tx); });
// Then verify topological order!
```

**Difficulty: ⭐⭐⭐ (3/5)**

<!-- end_slide -->

## Modification Difficulty: Hard

**Full re-selection from mempool**

- Query mempool directly, build your own selection
- Must track all dependencies (ancestor/descendant sets)
- Must respect weight, sigops, and ordering constraints
- Essentially rewriting `CreateNewBlock()`

```cpp
// Hard: Custom selection algorithm
auto mempool_txs = getMempool();
auto selected = mySelectionAlgorithm(mempool_txs);
// Must handle: dependencies, weight, sigops, ordering...
```

**Difficulty: ⭐⭐⭐⭐⭐ (5/5)**

<!-- end_slide -->

## Modification Difficulty: Summary

| Modification | Difficulty | Key Challenge |
|--------------|------------|---------------|
| Remove spam/inscriptions | ⭐ | Check parent dependencies |
| Exclude by size | ⭐ | Simple filter |
| Randomize order | ⭐⭐ | Maintain topology |
| Boost specific txs | ⭐⭐⭐ | Topology + weight |
| Add out-of-band txs | ⭐⭐⭐⭐ | Validation + deps |
| Custom fee algorithm | ⭐⭐⭐⭐⭐ | Full rewrite |

**Start simple! Filtering is powerful and easy.**

<!-- end_slide -->

## Example: Custom Transaction Filter

```cpp
// In template_provider.cpp, after getting block template:

// Get the block
CBlock block = block_template->getBlock();

// Filter transactions (example: exclude OP_RETURN > 80 bytes)
std::vector<CTransactionRef> filtered;
for (const auto& tx : block.vtx) {
    bool include = true;
    for (const auto& out : tx->vout) {
        if (out.scriptPubKey.IsUnspendable() && 
            out.scriptPubKey.size() > 80) {
            include = false;
            break;
        }
    }
    if (include) filtered.push_back(tx);
}
```

<!-- end_slide -->

## Rebuild and Test

```bash
# After modifying template_provider.cpp:

# Rebuild the container
docker-compose build sv2-node

# Restart
docker-compose up -d sv2-node

# Watch your custom templates flow
docker-compose logs -f jd-client | grep template
```

<!-- end_slide -->

## What Can You Build?

- **Transaction filtering**: Exclude spam, inscriptions, etc.
- **Priority ordering**: Favor certain transaction types
- **MEV protection**: Randomize ordering
- **Local policy**: Your node, your rules
- **Research**: Test new mempool policies

<!-- end_slide -->

## Current Limitations

- **Alpha software**: Expect bugs
- **Testnet only**: Don't use on mainnet yet
- **Single pool**: Limited pool support currently
- **No ASIC firmware**: Most ASICs need translator

<!-- end_slide -->

## The Ecosystem

```
┌─────────────────────────────────────────────────────────┐
│                    SRI PROJECT                          │
├─────────────────────────────────────────────────────────┤
│  stratum (Rust)     - Protocol libraries                │
│  sv2-apps (Rust)    - Pool, JDC, JDS, Translator        │
│  sv2-tp (C++)       - Template Provider (this repo)     │
└─────────────────────────────────────────────────────────┘

Website: https://stratumprotocol.org
Discord: https://discord.gg/fsEW23wFYs
```

<!-- end_slide -->

## Resources

- **Spec**: https://stratumprotocol.org/specification
- **sv2-tp**: https://github.com/gimballock/sv2-tp
- **sv2-apps**: https://github.com/stratum-mining/sv2-apps
- **Workshop**: https://github.com/Sjors/sv2-workshop
- **Getting Started**: https://stratumprotocol.org/blog/getting-started/

<!-- end_slide -->

## Summary

1. **Stratum v2** separates template creation from pool coordination
2. **Template Distribution** protocol enables push-based templates
3. **Job Declaration** lets miners propose their own templates
4. **sv2-tp** is a C++ Template Provider for Bitcoin Core
5. **Docker playground** lets you experiment on testnet4

**The future of mining is decentralized block building!**

<!-- end_slide -->

## Questions?

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   "Not your template, not your block"                   │
│                                                         │
│   - Every miner, soon                                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Try it**: `cd docker && docker-compose up -d`

<!-- end_slide -->

## Appendix: Key Configuration

### jdc-config.toml (connects to your sv2-tp)
```toml
[tp_config]
address = "172.30.0.10:8442"
```

### pool-config.toml (fallback to hosted TP)
```toml
[template_provider_type.Sv2Tp]
address = "75.119.150.111:8442"
```

<!-- end_slide -->

## Appendix: Port Reference

| Port | Service | Protocol |
|------|---------|----------|
| 8442 | sv2-tp Template Provider | SV2 |
| 34254 | Pool (SV2 miners) | SV2 |
| 34255 | Translator (SV1 miners) | SV1 |
| 34264 | Job Declarator Server | SV2 |
| 48332 | Bitcoin Core RPC | JSON-RPC |
| 48333 | Bitcoin Core P2P | Bitcoin |
