# SV2 Full Stack Docker Setup

Complete Stratum V2 mining stack with custom template support for solo mining with SV1 hardware (BitAxe).

**Single Bitcoin Core instance** - uses hosted TP for fallback templates.

## Quick Start

```bash
cd docker
docker-compose up -d

# Wait for Bitcoin Core to sync
docker-compose logs -f sv2-node

# Once synced, connect your BitAxe to: stratum+tcp://localhost:34255
```

## Architecture

```
[YOUR SETUP - Single Bitcoin Core]
Bitcoin Core + sv2-tp (your custom template provider)
    ↓ IPC                    ↓ RPC (mempool)
    ↓                        ↓
JDC ←─ (custom templates)  JDS (validates)
    ↓                        ↓
    └─→ Pool ←───────────────┘
         ↑
         └─ Hosted TP (fallback: vanilla blocks)
         ↓
    Translator
         ↓
      BitAxe
```

### How It Works

1. **Your sv2-tp** creates custom templates with your transaction selection
2. **JDC** receives templates and declares them as custom jobs
3. **JDS** validates custom jobs against Bitcoin Core mempool (RPC)
4. **JDC** sends validated custom jobs to **Pool** via Mining Protocol
5. **Pool** uses **hosted TP** (community.stratum.org) for fallback templates
6. **Miners work on your custom templates** when available, vanilla blocks as fallback
7. **Translator** bridges SV1 miners to Pool
8. **BitAxe** mines the work

### Why Single Node?

- **sv2-tp** uses IPC (exclusive access to Bitcoin Core socket)
- **Pool** uses hosted TP via TCP (no local Bitcoin Core needed!)
- **JDS** uses RPC (shared access, no conflict)

**Result**: Only ONE Bitcoin Core instance needed (~25GB instead of 50GB)

## Components

| Service | Container | IP | Purpose |
|---------|-----------|----|---------| 
| **sv2-node** | Bitcoin Core + sv2-tp | 172.30.0.10 | Your custom template generation |
| **jd-client** | JDC | 172.30.0.12 | Declares custom jobs |
| **jd-server** | JDS | 172.30.0.11 | Validates custom jobs |
| **pool** | Pool | 172.30.0.13 | Mining coordination |
| **translator-proxy** | Translator | 172.30.0.14 | SV1 ↔ SV2 bridge |

### Ports

- `8442` - sv2-tp Template Provider
- `34254` - Pool (SV2)
- `34255` - Translator (SV1 miners connect here)
- `34264` - Job Declarator Server
- `48332` - Bitcoin Core RPC
- `48333` - Bitcoin Core P2P

## Configuration

All config files are in `docker/config/`:

- **jdc-config.toml** - Connects to sv2-tp (172.30.0.10:8442) and JDS
- **jds-config.toml** - Uses RPC to Bitcoin Core (172.30.0.10:48332)
- **pool-config.toml** - Uses hosted TP (community.stratum.org:8442) for fallback
- **translator-config.toml** - Connects to Pool (172.30.0.13:34254)

### Key Settings

**Pool uses hosted TP for fallback**:
```toml
# pool-config.toml
[template_provider_type.Sv2Tp]
address = "community.stratum.org"
port = 8442
public_key = "EguTM8URcZDQVeEBsM4B5vg9weqEUnufA8pm85fG4bZd"
```

**Custom templates come from JDC**, not from the template provider!

**Coinbase Rewards**: Update the `coinbase_reward_script` in pool/jdc/jds configs with your own address:
```toml
coinbase_reward_script = "wpkh([00000000/0'/0'/0']YOUR_PUBKEY_HERE)"
```

## Modifying Transaction Selection

Your custom transaction selection logic goes in **sv2-tp**. The Pool's hosted TP is only for fallback - your custom templates take priority via JDC custom jobs.

To modify sv2-tp:
1. Edit `/Users/ericprice/repos/sv2-tp/src/` (your transaction selection logic)
2. Rebuild: `docker-compose build sv2-node`
3. Restart: `docker-compose up -d sv2-node`

## Troubleshooting

### Services keep restarting

**Bitcoin Core not synced yet**. Check IBD status:
```bash
docker-compose logs sv2-node | grep -i "ibd\|sync"
```

JDS and Pool will retry until Bitcoin Core is ready.

### Pool connection to hosted TP fails

Check internet connectivity and that `community.stratum.org:8442` is reachable:
```bash
docker exec sv2-pool nc -zv community.stratum.org 8442
```

### JDS RPC connection errors

Bitcoin Core RPC not ready. JDS will retry automatically.

### Custom jobs not being used

Check JDC logs:
```bash
docker-compose logs jd-client | grep -i "custom\|declare"
```

Ensure sv2-tp is running and providing templates.

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
docker-compose logs -f pool

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
```

## Converting to Your Fork

This is a fork of `fossatmara/sv2-tp`. To push changes:

```bash
# Already configured as your fork
git remote -v  # Should show gimballock/sv2-tp

# Commit your changes
git add .
git commit -m "Your commit message

Assisted-by: GitHub Copilot
Assisted-by: OpenAI GPT-4"

# Push
git push origin master
```

## Files

```
docker/
├── README.md                    # This file
├── docker-compose.yml           # Single-node orchestration
├── Dockerfile                   # Bitcoin Core + sv2-tp
└── config/
    ├── jdc-config.toml          # Job Declarator Client config
    ├── jds-config.toml          # Job Declarator Server config
    ├── pool-config.toml         # Pool config (uses hosted TP)
    └── translator-config.toml   # SV1↔SV2 translator config
```

## Requirements

- Docker & Docker Compose
- ~25GB disk space (single Bitcoin Core testnet4 node)
- Internet connection (for hosted TP fallback)
- Patience (IBD takes hours)

## Benefits vs Two-Node Setup

✅ **50% less disk space** (25GB vs 50GB)  
✅ **Faster sync** (one IBD instead of two)  
✅ **Simpler architecture** (one node to manage)  
✅ **Your custom templates** via JDC  
✅ **Vanilla fallback** from community TP  
✅ **No second node maintenance**

## References

- [SV2 Specification](https://github.com/stratum-mining/sv2-spec)
- [SV2 Apps](https://github.com/stratum-mining/sv2-apps)  
- [Job Declaration Protocol](https://github.com/stratum-mining/sv2-spec/blob/main/06-Job-Declaration-Protocol.md)
- [Bitcoin Core 30.0](https://bitcoincore.org/en/releases/30.0/)
- [Community Hosted TP](https://github.com/stratum-mining/sv2-apps#hosted-template-provider)

## License

Same as upstream sv2-tp project.
