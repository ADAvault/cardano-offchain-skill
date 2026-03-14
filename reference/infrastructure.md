# Ogmios, Kupo, and Node Infrastructure

This document covers the chain infrastructure components used for dApp development:
Ogmios (WebSocket bridge to cardano-node), Kupo (chain indexer), Docker deployment,
and operational patterns learned from production use.

## Component Overview

```
cardano-node  <-->  Ogmios (WebSocket API, port 1337)
                       |
                       +-->  Kupo (chain indexer, port 1442)
                       |
                       +-->  MeshJS / off-chain code
```

- **cardano-node**: Runs the Cardano consensus protocol, maintains chain state
- **Ogmios**: Lightweight WebSocket bridge providing a JSON-RPC interface to the node
- **Kupo**: Fast chain indexer that tracks UTxOs matching specified patterns
- **HAProxy**: Load balancer for failover between primary and DR infrastructure

## Ogmios

### Deployment (Docker)

```yaml
# docker-compose.yml
services:
  ogmios:
    image: cardanosolutions/ogmios:latest
    ports:
      - "1337:1337"
    volumes:
      - /opt/cardano/cnode/db:/db:ro
      - /opt/cardano/cnode/files:/config:ro
    command:
      - --node-socket /db/node.socket
      - --node-config /config/config.json
    restart: unless-stopped
```

### WebSocket API

Ogmios provides a WebSocket interface at `ws://localhost:1337`. Key capabilities:

- **Chain sync** -- stream blocks from any point
- **Transaction submission** -- submit signed transactions
- **State queries** -- protocol parameters, UTxO queries, stake distribution
- **Transaction evaluation** -- estimate execution units for Plutus scripts

### Transaction Evaluation

The Ogmios evaluator estimates execution units (memory + CPU steps) for Plutus
script transactions. MeshJS integrates with it for automatic `exUnits` calculation:

```typescript
// MeshTxBuilder with Ogmios evaluator
const txBuilder = new MeshTxBuilder({
  fetcher: blockchainProvider,
  evaluator: ogmiosEvaluator,
});
```

When the evaluator is unreliable or you need deterministic results, bypass it
and set manual `exUnits`:

```typescript
// Manual exUnits -- remove evaluator from builder
const txBuilder = new MeshTxBuilder({
  fetcher: blockchainProvider,
  // No evaluator
});

// Pass exUnits as 3rd argument to redeemer
txBuilder.mintRedeemerValue(redeemer, 'Mesh', { mem: 500000, steps: 200000000 });
```

## Kupo

### Deployment (Docker)

```yaml
# docker-compose.yml
services:
  kupo:
    image: cardanosolutions/kupo:latest
    ports:
      - "1442:1442"
    volumes:
      - kupo-db:/db
    command:
      - --ogmios-host ogmios
      - --ogmios-port 1337
      - --since origin
      - --workdir /db
      - --match "0027a6fe95fffb1a6a5a5282b98a754a9f0d7402db0a4e40cd466e5b/*"
    restart: unless-stopped

volumes:
  kupo-db:
```

### Targeted vs Full Indexing

The `--match` flag controls what Kupo indexes. This is the single most important
configuration decision:

| Approach | `--match` | DB Size | Use Case |
|----------|-----------|---------|----------|
| Full indexing | `"*"` | ~50 GB | Public explorer, multi-tenant |
| Targeted | `"<credential>/*"` | ~50 MB | Single-dApp, known addresses |

Always start targeted. You can add patterns later. Starting with `"*"` means
50+ GB of disk and hours of initial sync.

### Adding Patterns

When deploying new contracts, add their script credentials to Kupo. Two approaches:

#### 1. Docker `--match` Flags (Clean Restart)

Add all patterns to the Docker command:

```bash
docker run ... \
  --match "0027a6fe95fffb1a6a5a5282b98a754a9f0d7402db0a4e40cd466e5b/*" \
  --match "064b53addd46894f86faca65297c30df2ca8b26c58ebb1c320a98928/*" \
  --match "<new-script-credential>/*"
```

#### 2. Kupo PUT API (Runtime)

Add patterns dynamically without restart:

```bash
curl -X PUT http://localhost:1442/v1/patterns/new-credential/* \
  -H "Content-Type: application/json" \
  -d '{"rollback_to": {"slot_no": 12345678, "header_hash": "abc123..."}}'
```

### ConflictingOptionsException on Restart

When you add patterns via the PUT API, Kupo's internal DB tracks different patterns
than what the Docker `--match` flags specify. On container restart, Kupo detects
the mismatch and refuses to start with `ConflictingOptionsException`.

**Fix**: Stop the container, remove the DB volume, recreate with ALL patterns in
`--match` flags:

```bash
docker stop kupo
docker volume rm kupo-db
docker run ... --match "pattern1/*" --match "pattern2/*" --match "pattern3/*"
```

Fresh sync on preview with ~12 targeted patterns takes approximately 25 minutes.

### PUT Beyond Safe Zone

Adding patterns with a rollback point older than 36 hours requires the
`unsafe_allow_beyond_safe_zone` flag:

```bash
curl -X PUT http://localhost:1442/v1/patterns/new-credential/* \
  -H "Content-Type: application/json" \
  -d '{"rollback_to": {"slot_no": 12345678}, "limit": "unsafe_allow_beyond_safe_zone"}'
```

The PUT request blocks until re-indexing completes, which can take minutes
depending on chain history depth.

### PUT Stalling Sync

A long-running PUT re-index can prevent Kupo from advancing to new blocks. Symptoms:

- `most_recent_checkpoint` stops advancing
- Ogmios error 3117: "unknown UTxO references" because Kupo returns stale UTxOs
  the node has already consumed
- Queries return outdated data

**Fix**: Kill stale `curl` processes and restart the container:

```bash
# Find and kill stale PUT requests
ps aux | grep 'curl.*kupo' | grep -v grep
kill <pid>

# Restart Kupo
docker restart kupo
```

If you kill a stale curl and retry, Kupo may return 503 "too busy" until the
previous re-index drains.

## SSH Tunnels for Cross-Subnet Access

When the development machine is on a different subnet from the node infrastructure,
network firewalls may block direct access. Use SSH tunnels:

```bash
# Forward Ogmios (1337) and Kupo (1442) from node-host to localhost
ssh -N -L 1337:localhost:1337 -L 1442:localhost:1442 cardano@node-host
```

This is needed when your dev machine is on a different subnet from the node
infrastructure and the network firewall only allows SSH between subnets.

## HAProxy Failover

For production, HAProxy provides automatic failover between primary and DR
infrastructure:

```
HAProxy
  |-- primary:  ogmios-primary (Ogmios:1337, Kupo:1442)
  |-- backup:   ogmios-dr (Ogmios:1337, Kupo:1442)
```

HAProxy health checks the Ogmios and Kupo endpoints. If the primary goes down,
traffic automatically routes to DR.

### Health Check Endpoints

```bash
# Ogmios health
curl http://localhost:1337/health

# Kupo health
curl http://localhost:1442/v1/health

# Kupo most recent checkpoint (sync status)
curl http://localhost:1442/v1/checkpoints | jq '.[0]'
```

## socat for Node Socket Bridging

Ogmios needs access to the cardano-node Unix domain socket. When Ogmios runs on a
different machine than the node, use socat to bridge the socket over TCP:

```bash
# On the node machine: expose Unix socket as TCP
socat TCP-LISTEN:3002,reuseaddr,fork UNIX-CONNECT:/opt/cardano/cnode/db/node.socket

# Ogmios connects to the TCP bridge instead of a local socket
ogmios --node-socket tcp://node-host:3002
```

socat is deployed on relay nodes to bridge to Ogmios infrastructure.

## Version Information

Current production versions:

| Component | Version | Notes |
|-----------|---------|-------|
| cardano-node | 10.5.4 | All nodes, p2p-config.json |
| Ogmios | latest (Docker) | WebSocket on port 1337 |
| Kupo | latest (Docker) | HTTP on port 1442 |
| HAProxy | 2.8.16 | On Ogmios/Kupo VMs |
| Docker | 29.3.0 | On Ogmios/Kupo VMs |

## See Also

- [API Resilience](api-resilience.md) -- frontend failover patterns
- [Gotchas](gotchas.md) -- Kupo operational issues, infrastructure quirks
