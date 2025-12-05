# SV2 Full Stack Docker Setup

Complete Stratum V2 mining stack with custom template support for solo mining with SV1 hardware (BitAxe).

## Quick Start

```bash
cd docker
docker-compose up -d

# Wait for both Bitcoin Core nodes to sync (this takes time)
docker-compose logs -f sv2-node-miner
docker-compose logs -f sv2-node-pool

# Once synced, connect your BitAxe to: stratum+tcp://localhost:34255
```

## Architecture

```
[MINER-SIDE]
sv2-node-miner (Bitcoin Core + sv2-tp)
    ↓ Template Distribution Protocol
JDC (Job Declarator Client)
    ↓ Job Declaration Protocol
    
[POOL-SIDE]  
JDS (Job Declarator Server) ←─ RPC ─→ sv2-node-pool (Bitcoin Core + Pool)
    ↓ Mining Protocol                        ↓ IPC
Pool ←──────────────────────────────────────┘
    ↓ SV2 Protocol
Translator Proxy
    ↓ SV1 Protocol
BitAxe / SV1 Miners
```

### Why Two Bitcoin Core Instances?

**IPC socket limitation**: Only one process can connect to Bitcoin Core's IPC socket at a time.

- **Miner-side**: sv2-tp needs IPC for custom templates
- **Pool-side**: Pool needs IPC for block submission and validation

Since they can't share the socket, we run two separate Bitcoin Core instances.

## Components

| Service | Container | IP | Purpose |
|---------|-----------|----|---------| 
| **sv2-node-miner** | Bitcoin Core + sv2-tp | 172.30.0.10 | Custom template generation |
| **jd-client** | JDC | 172.30.0.12 | Declares custom jobs |
| **sv2-node-pool** | Bitcoin Core + Pool | 172.30.0.13 | Mining coordination |
| **jd-server** | JDS | 172.30.0.11 | Validates custom jobs |
| **translator-proxy** | Translator | 172.30.0.14 | SV1 ↔ SV2 bridge |

### Ports

- `8442` - sv2-tp Template Provider
- `34254` - Pool (SV2)
- `34255` - Translator (SV1 miners connect here)
- `34264` - Job Declarator Server
- `48332/48432` - Bitcoin Core RPC
- `48333/48433` - Bitcoin Core P2P

## How Custom Templates Work

1. **sv2-tp** creates custom templates from miner-side Bitcoin Core
2. **JDC** receives templates and declares them as custom jobs
3. **JDS** validates custom jobs against pool-side Bitcoin Core mempool
4. **JDC** sends validated jobs to **Pool** via Mining Protocol
5. **Pool** distributes work (mix of standard + custom jobs)
6. **Translator** bridges SV1 miners to Pool
7. **BitAxe** mines the work

### Important: Pool Template Provider

The Pool's `template_provider_type` config is **ONLY for standard fallback templates**:

```toml
# pool-config.toml
[template_provider_type.BitcoinCoreIpc]
unix_socket_path = "/root/.bitcoin/testnet4/node.sock"
```

- ✅ Pool gets standard templates: Bitcoin Core IPC
- ✅ Pool gets custom jobs: JDC via Mining Protocol  
- ❌ `JobDeclarator` template provider type doesn't exist in v0.1.0

Custom jobs are NOT configured via `template_provider_type` - they come through the Mining Protocol from JDC.

## Configuration

All config files are in `docker/config/`:

- **jdc-config.toml** - Connects to sv2-tp (172.30.0.10:8442) and JDS
- **jds-config.toml** - Uses RPC to pool-side Bitcoin Core (172.30.0.13:48332)
- **pool-config.toml** - Uses IPC to local Bitcoin Core socket
- **translator-config.toml** - Connects to Pool (172.30.0.13:34254)

### Key Settings

**Coinbase Rewards**: Update the `coinbase_reward_script` in pool/jdc/jds configs with your own address:
```toml
coinbase_reward_script = "wpkh([00000000/0'/0'/0']YOUR_PUBKEY_HERE)"
```

**Authority Keys**: The configs use example keys. For production, generate your own.

## Troubleshooting

### Services keep restarting

**Bitcoin Core not synced yet**. Check IBD status:
```bash
docker-compose logs sv2-node-pool | grep -i "ibd\|sync"
```

Pool and JDS will retry until Bitcoin Core is ready.

### "JobDeclarator variant does not exist"

This error means you're trying to use `JobDeclarator` as a template provider type in pool-config.toml. This variant doesn't exist - use `BitcoinCoreIpc` instead.

### Translator connection refused

Pool isn't running yet. Wait for pool-side Bitcoin Core to complete IBD.

### JDS RPC connection errors

Pool-side Bitcoin Core RPC not ready. JDS will retry automatically.

## Development

### Rebuild
```bash
docker-compose down
docker-compose build
docker-compose up -d
```

### View Logs
```bash
# All services
docker-compose logs -f

# Specific service  
docker-compose logs -f sv2-node-pool

# Errors only
docker-compose logs | grep -iE "error|panic|fatal"
```

### Clean Restart (⚠️ Deletes blockchain data)
```bash
docker-compose down -v
docker-compose up -d
```

### Check Service Status
```bash
docker-compose ps
docker exec sv2-node-pool ps aux
```

## Converting to Your Fork

This is a clone of `fossatmara/sv2-tp`. To convert to your fork:

```bash
# 1. Fork fossatmara/sv2-tp on GitHub

# 2. Update remote
git remote set-url origin https://github.com/YOUR_USERNAME/sv2-tp.git

# 3. Commit your changes
git add .
git commit -m "Add full SV2 stack Docker orchestration

Assisted-by: GitHub Copilot
Assisted-by: OpenAI GPT-4"

# 4. Push
git push -u origin master
```

## Files

```
docker/
├── README.md                    # This file
├── docker-compose.yml           # Full stack orchestration
├── Dockerfile                   # Miner-side: Bitcoin Core + sv2-tp
├── Dockerfile.pool              # Pool-side: Bitcoin Core + Pool
└── config/
    ├── jdc-config.toml
    ├── jds-config.toml
    ├── pool-config.toml
    └── translator-config.toml
```

## Requirements

- Docker & Docker Compose
- ~50GB disk space (two Bitcoin Core testnet4 nodes)
- Patience (IBD takes hours)

## References

- [SV2 Specification](https://github.com/stratum-mining/sv2-spec)
- [SV2 Apps](https://github.com/stratum-mining/sv2-apps)  
- [Job Declaration Protocol](https://github.com/stratum-mining/sv2-spec/blob/main/06-Job-Declaration-Protocol.md)
- [Bitcoin Core 30.0](https://bitcoincore.org/en/releases/30.0/)

## License

Same as upstream sv2-tp project.
