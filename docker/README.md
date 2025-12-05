# SV2 Solo Mining Stack with Custom Templates

Complete Stratum V2 mining infrastructure for solo mining with custom block templates. Supports both SV1 miners (BitAxe, etc.) via translation proxy and native SV2 miners (CPU miner included for testing).

## Quick Start

```bash
cd docker
docker-compose up -d

# Monitor Bitcoin Core sync (required before mining)
docker-compose logs -f sv2-node

# Once synced, connect your miner:
# - SV1 miners (BitAxe): stratum+tcp://localhost:34255
# - SV2 miners: 172.30.0.13:34254

# Optional: Enable CPU miner for testing (uncomment in docker-compose.yml)
docker-compose up -d cpu-miner
```

## Architecture

### Single Bitcoin Core Design

```
Local Bitcoin Core + sv2-tp (template generation)
    ↓ IPC
    ↓
Job Declarator Client ──→ Job Declarator Server
    ↓                         ↓ RPC
    └────→ Pool ←─────────────┘ (validates via SRI hosted node)
            ↑
            └─ Hosted TP (fallback templates)
            ↓
      Translation Proxy
            ↓
        SV1 Miners
```

**Architecture Notes**:
- **Local sv2-tp**: Generates your custom templates via IPC to local Bitcoin Core
- **JDS validation**: Uses hosted testnet4 node from SRI (Stratum Reference Implementation) for mempool validation
- **Pool fallback**: Uses hosted TP from SRI for vanilla block templates when custom jobs unavailable
- **Result**: Template generation is local (your custom logic), validation is remote (shared infrastructure)

### Data Flow

1. **Bitcoin Core** syncs testnet4 blockchain
2. **Template Provider** generates custom block templates via IPC
3. **JDC** receives templates from sv2-tp and declares custom jobs
4. **JDS** validates custom jobs against Bitcoin Core mempool (RPC)
5. **Pool** coordinates mining:
   - Custom jobs from JDC (priority)
   - Vanilla blocks from hosted TP (fallback)
6. **Translation Proxy** bridges SV1 miners to SV2 Pool
7. **Miners** work on custom templates (when available) or fallback blocks

## Components

| Service | Container | IP | Port(s) | Purpose |
|---------|-----------|----|---------|---------| 
| **sv2-node** | Bitcoin Core + sv2-tp | 172.30.0.10 | 8442, 48332-48333 | Custom template generation |
| **jd-client** | JDC | 172.30.0.12 | - | Declares custom jobs to Pool |
| **jd-server** | JDS | 172.30.0.11 | 34264 | Validates custom jobs |
| **pool** | Pool | 172.30.0.13 | 34254 | Mining coordination (SV2) |
| **translator-proxy** | Translator | 172.30.0.14 | 34255 | SV1 ↔ SV2 bridge |
| **cpu-miner** (optional) | CPU Miner | 172.30.0.15 | - | Test mining without hardware |

### Port Reference

- **8442** - sv2-tp Template Provider (TCP)
- **34254** - Pool SV2 port (native SV2 miners connect here)
- **34255** - Translator SV1 port (SV1 miners connect here)
- **34264** - Job Declarator Server
- **48332** - Bitcoin Core RPC (testnet4)
- **48333** - Bitcoin Core P2P (testnet4)

## Configuration Files

All configs are in `docker/config/`:

### jdc-config.toml
Job Declarator Client configuration:
- Connects to sv2-tp at `172.30.0.10:8442` for custom templates
- Connects to JDS at `172.30.0.11:34264` for job declaration
- Declares custom jobs to Pool
- Manages mining job tokens

### jds-config.toml
Job Declarator Server configuration:
- Validates custom jobs against Bitcoin Core RPC
- Currently configured to use SRI's hosted testnet4 node
- This separates template generation (local) from validation (remote)
- Allocates mining job tokens
- Ensures custom templates are valid before Pool uses them
- Coinbase reward script (update with your address)

**Why use the hosted node?**
- No need to wait for local Bitcoin Core to sync for validation
- Reduces local resource usage
- Your custom templates still come from your local sv2-tp
- Hosted node provided by Stratum Reference Implementation (SRI) organization

**To use your local Bitcoin Core instead**, edit `jds-config.toml`:
```toml
core_rpc_url = "http://172.30.0.10"
core_rpc_port = 48332
core_rpc_user = "user"
core_rpc_pass = "password"
```

### pool-config.toml
Pool configuration:
- Listens on `0.0.0.0:34254` for SV2 miners
- Uses hosted TP at `75.119.150.111:8442` for fallback templates (provided by SRI)
- Receives custom jobs from JDC via Mining Protocol
- Coinbase reward script (update with your address)
- Authority public key for authentication

**Note**: Both the Pool's fallback TP and JDS's validation node are hosted by the Stratum Reference Implementation (SRI) organization on testnet4.

### translator-config.toml
Translation Proxy configuration:
- Listens on `0.0.0.0:34255` for SV1 miners (BitAxe, etc.)
- Connects to Pool at `172.30.0.13:34254` (upstream SV2)
- Translates SV1 ↔ SV2 protocol
- Manages difficulty adjustment for SV1 miners

**Important**: Update `coinbase_reward_script` in pool, jdc, and jds configs with your own Bitcoin address:
```toml
coinbase_reward_script = "wpkh([00000000/0'/0'/0']YOUR_PUBKEY_HERE)"
```

## Connecting Miners

### SV1 Miners (BitAxe, CGMiner, etc.)

SV1 miners are fully supported via the Translation Proxy:

```
Miner Configuration:
- URL: stratum+tcp://localhost:34255
- User: any (e.g., your_address)
- Password: x
```

The translator automatically:
- Converts SV1 protocol to SV2
- Manages difficulty adjustments
- Forwards shares to Pool
- Translates job notifications

### SV2 Native Miners

Native SV2 miners connect directly to the Pool:

```
Pool Address: 172.30.0.13:34254
Authority Key: 9auqWEzQDVyd2oe1JVGFLMLHZtCo2FFqZwtKA5gd9xbuEu7PH72
```

**CPU Miner (included for testing)**:
1. Uncomment the `cpu-miner` section in `docker-compose.yml`
2. Start: `docker-compose up -d cpu-miner`
3. Monitor: `docker-compose logs -f cpu-miner`

The CPU miner is configured with:
- 1 core (adjustable via `--cores`)
- 5ms handicap to prevent overheating (adjustable via `--handicap`)
- Direct SV2 connection to Pool

## Monitoring

### Check All Services

```bash
# Service status
docker-compose ps

# All logs
docker-compose logs -f

# Specific service
docker-compose logs -f sv2-node
docker-compose logs -f pool
docker-compose logs -f jd-client
docker-compose logs -f jd-server
docker-compose logs -f translator-proxy
docker-compose logs -f cpu-miner
```

### Bitcoin Core Sync Status

```bash
# Check IBD progress
docker-compose logs sv2-node | grep -i "progress\|ibd"

# Check current block height
docker exec sv2-node bitcoin-cli -testnet4 getblockchaininfo
```

### Mining Activity

```bash
# Watch for shares (SV1 miners via translator)
docker-compose logs -f translator-proxy | grep -i "submit\|share\|accept"

# Watch for shares (SV2 miners / CPU miner)
docker-compose logs -f cpu-miner | grep -i "submit\|share"

# Pool activity
docker-compose logs -f pool | grep -i "share\|submit"
```

### Custom Template Activity

```bash
# JDC declaring custom jobs
docker-compose logs -f jd-client | grep -i "template\|job\|token"

# JDS validating jobs
docker-compose logs -f jd-server | grep -i "allocate\|token\|success"

# sv2-tp generating templates
docker-compose logs -f sv2-node | grep -i "template\|sv2"
```

### Resource Usage

```bash
# Container stats
docker stats

# Disk usage
docker-compose exec sv2-node du -sh /root/.bitcoin
```

## Development

### Modifying Transaction Selection

Your custom transaction selection logic is in **sv2-tp** (not the Pool). The Pool's hosted TP is only for fallback.

1. Edit transaction selection in `/Users/ericprice/repos/sv2-tp/src/`
2. Rebuild the sv2-node container:
   ```bash
   docker-compose build sv2-node
   ```
3. Restart:
   ```bash
   docker-compose up -d sv2-node
   ```

Your custom templates will automatically flow through: `sv2-tp → JDC → JDS → Pool → Miners`

### Testing Changes

1. **With CPU Miner** (fast feedback):
   ```bash
   # Enable CPU miner
   docker-compose up -d cpu-miner
   
   # Watch it mine your custom templates
   docker-compose logs -f cpu-miner jd-client pool
   ```

2. **With BitAxe** (real hardware):
   - Point BitAxe to `stratum+tcp://YOUR_IP:34255`
   - Monitor: `docker-compose logs -f translator-proxy pool`

### Rebuilding Components

```bash
# Rebuild sv2-node (Bitcoin Core + sv2-tp)
docker-compose build sv2-node

# Rebuild CPU miner
docker-compose build cpu-miner

# Rebuild all
docker-compose build

# Force rebuild without cache
docker-compose build --no-cache
```

## File Structure

```
docker/
├── README.md                    # This file
├── docker-compose.yml           # Single-node orchestration
├── Dockerfile                   # Bitcoin Core + sv2-tp
├── Dockerfile.mining-device     # CPU miner (optional)
└── config/
    ├── jdc-config.toml          # Job Declarator Client
    ├── jds-config.toml          # Job Declarator Server
    ├── pool-config.toml         # Pool (uses hosted TP fallback)
    └── translator-config.toml   # SV1↔SV2 translator
```

## Troubleshooting

### Services Keep Restarting

**Cause**: Bitcoin Core not synced yet (IBD in progress)

**Solution**: Wait for sync to complete
```bash
docker-compose logs sv2-node | grep -i "progress"
```

JDS and Pool will automatically retry until Bitcoin Core is ready.

### Pool Can't Connect to Hosted TP

**Cause**: Network connectivity or hosted TP down

**Solution**: 
1. Check internet connection
2. Verify hosted TP is accessible:
   ```bash
   telnet 75.119.150.111 8442
   ```
3. Pool will continue using custom jobs from JDC even if hosted TP is unavailable

### JDC Can't Connect to sv2-tp

**Cause**: sv2-tp not running or IPC socket issue

**Solution**:
```bash
# Check sv2-tp is running
docker-compose logs sv2-node | grep sv2-tp

# Restart sv2-node
docker-compose restart sv2-node
```

### JDS Can't Validate Jobs

**Cause**: Bitcoin Core RPC not accessible

**Solution**:
```bash
# Check Bitcoin Core RPC
docker exec sv2-node bitcoin-cli -testnet4 getblockchaininfo

# Verify JDS config has correct RPC settings
grep core_rpc docker/config/jds-config.toml

# Restart JDS
docker-compose restart jd-server
```

### Translator Proxy Connection Issues

**Cause**: Pool not ready or configuration mismatch

**Solution**:
```bash
# Verify Pool is running
docker-compose logs pool | tail -20

# Check translator config matches Pool IP
grep upstream_address docker/config/translator-config.toml
# Should be: 172.30.0.13

# Restart translator
docker-compose restart translator-proxy
```

### No Shares Being Submitted

**Cause**: Difficulty too high or miner not connected

**Solution**:
```bash
# Check miner connection (SV1)
docker-compose logs translator-proxy | grep -i "downstream\|connected"

# Check miner connection (SV2)
docker-compose logs cpu-miner | grep -i "connection\|channel"

# Verify Pool is sending jobs
docker-compose logs pool | grep -i "job\|mining"
```

### CPU Miner High CPU Usage

**Cause**: Handicap set too low

**Solution**: Increase handicap in `docker-compose.yml`:
```yaml
"--handicap", "10000",  # 10ms delay (slower but cooler)
```

### Disk Space Issues

**Cause**: Bitcoin Core blockchain data (~25GB for testnet4)

**Solution**:
```bash
# Check disk usage
df -h

# Prune old data (if needed)
docker-compose exec sv2-node bitcoin-cli -testnet4 pruneblockchain 100000
```

## Requirements

- Docker & Docker Compose
- ~25GB disk space (single Bitcoin Core testnet4 node)
- Internet connection (for hosted TP fallback and IBD)
- Patience (IBD takes hours on first run)



## Advanced Configuration

### Using a Remote Bitcoin Core for JDS

If you have a remote Bitcoin Core node, you can configure JDS to use it instead of the local one:

1. Edit `docker/config/jds-config.toml`:
   ```toml
   core_rpc_url = "http://remote.node.ip"
   core_rpc_port = 8332  # or your RPC port
   core_rpc_user = "your_rpc_user"
   core_rpc_pass = "your_rpc_password"
   ```

2. Restart JDS:
   ```bash
   docker-compose restart jd-server
   ```

This allows you to:
- Use a more powerful remote node for validation
- Share a single Bitcoin Core node across multiple JDS instances
- Separate template generation (sv2-tp) from validation (JDS)

### Adjusting CPU Miner Performance

Edit `docker-compose.yml`:
```yaml
command: [
  "/app/mining_device",
  "--address-pool", "172.30.0.13:34254",
  "--cores", "2",           # Use more cores
  "--handicap", "1000",     # Faster (1ms delay)
  "--nominal-hashrate-multiplier", "0.1"  # Advertise lower hashrate for faster shares
]
```

### Using Different Bitcoin Network

Edit `docker/Dockerfile` and change:
```dockerfile
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

Update supervisord.conf to use `-regtest` or `-signet` instead of `-testnet4`

### Custom Hosted TP

Edit `docker/config/pool-config.toml`:
```toml
[template_provider_type.Sv2Tp]
address = "YOUR_TP_IP:PORT"
```

## Support

- **SV2 Spec**: https://stratumprotocol.org
- **sv2-apps**: https://github.com/stratum-mining/sv2-apps
- **sv2-tp**: https://github.com/gimballock/sv2-tp (this fork)

## License

See parent repository for license information.
