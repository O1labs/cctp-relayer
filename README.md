# CCTP Relayer

<p align="center"><img src=".github/assets/portal.png"></p>

Lightweight service that listens and forwards Cross Chain Transfer Protocol (CCTP) events. Easily extensible with more chains.

## 🚀 Quick Start

```bash
git clone https://github.com/O1ahmad/cctp-relayer
cd cctp-relayer
make install
cctp-relayer start --config ./config/sample-config.yaml
```

Sample configs: [`config/`](config)

## ⚙️ Configuration

### Chain Configuration

Each chain requires specific settings. Below are the key fields:

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `rpc` | string | RPC endpoint URL | `https://eth-mainnet.g.alchemy.com/v2/...` |
| `ws` | string | WebSocket endpoint (EVM chains) | `wss://...` |
| `chain-id` | string/int | Chain identifier | `"grand-1"` (Noble), `5` (Goerli) |
| `domain` | int | CCTP domain ID | `0` (Ethereum), `4` (Noble), `5` (Solana) |
| `start-block` | uint64 | Starting block height (0 = latest) | `0` |
| `lookback-period` | uint64 | Historical blocks to look back on launch | `5` |
| `workers` | int | Number of concurrent workers | `8` |
| `broadcast-retries` | int | Number of broadcast retry attempts | `5` |
| `broadcast-retry-interval` | int | Seconds between retries | `10` |
| `min-mint-amount` | uint64 | Minimum amount to relay (in base units) | `10000000` ($10) |
| `minter-private-key` | string | Hex-encoded private key | `0xabc123...` |
| `metrics-denom` | string | Denomination for balance metrics | `"ETH"`, `"USDC"` |
| `metrics-exponent` | int | Exponent for unit conversion | `18` (Wei→ETH) |

### Supported Chains

| Chain | Domain | Example Config Key |
|-------|--------|-------------------|
| Noble | 4 | `noble` |
| Ethereum | 0 | `ethereum` |
| Optimism | 2 | `optimism` |
| Arbitrum | 3 | `arbitrum` |
| Avalanche | 1 | `avalanche` |
| Solana | 5 | `solana` |

### Enabled Routes

Define which source→destination routes are active:

```yaml
enabled-routes:
  0: [4, 5]    # ethereum → noble, solana
  1: [4]       # avalanche → noble
  2: [4]       # optimism → noble
  3: [4]       # arbitrum → noble
  4: [0,1,2,3,5] # noble → ethereum, avalanche, optimism, arbitrum, solana
```

### Circle API Settings

| Field | Type | Description | Default |
|-------|------|-------------|---------|
| `attestation-base-url` | string | Base URL for attestation API | `"https://iris-api-sandbox.circle.com/attestations/"` |
| `api-version` | string | API version (`v1` or `v2`) | `"v1"` |
| `fetch-retries` | int | Additional attestation fetch attempts | `30` |
| `fetch-retry-interval` | int | Seconds between fetch retries | `3` |
| `enable-fast-transfer-monitoring` | bool | Enable v2 allowance monitoring | `false` |
| `reattest-max-retries` | int | Max re-attestation attempts (v2) | `3` |
| `expiration-buffer-blocks` | int | Blocks before expiry to re-attest | `100` |
| `allowance-monitor-token` | string | Token to monitor (v2) | `"USDC"` |
| `allowance-monitor-interval` | int | Polling interval in seconds | `30` |

### Filter Plugins

Filters apply validation rules to incoming messages:

| Filter | Description | Config Example |
|--------|-------------|----------------|
| `depositor-whitelist` | Only allow transfers from whitelisted depositors (EVM chains) | See below |
| `low-transfer` | Filter transfers below minimum mint amounts | Auto-configured from chain configs |
| `destination-caller` | Validate destination caller addresses | Controlled by `destination-caller-only` flag |

**Depositor Whitelist Configuration:**
```yaml
filters:
  - name: "depositor-whitelist"
    enabled: true
    config:
      provider: "quicknode-kv"
      provider_config:
        api_key: "your-api-key"
      kv_key: "cctp-depositor-whitelist"
      refresh_interval: 300  # seconds
```

## 🔧 Operational Modes

### Flush Interval Mode

Run periodic flushes with `--flush-interval <duration>` (e.g., `5m`).

- First flush per chain starts at `latest height - (2 × lookback period)`
- Flush ends at `latest height - lookback period`
- Subsequent flushes continue from last flushed block
- Configure `lookback-period` based on block time and flush interval

**Example (30‑minute flush interval):**
- Ethereum (12s blocks): `150` blocks
- Polygon (2s blocks): `900` blocks
- Arbitrum (0.26s blocks): `~6950` blocks

### Flush‑Only Mode

Run `--flush-only-mode` for a secondary relayer that only retries failed transactions.

- Starts at `latest height - (4 × lookback period)`
- Ends at `latest height - (3 × lookback period)`
- Uses same config as primary relayer to avoid overlap

## 📊 Prometheus Metrics

Metrics are exposed at `http://localhost:2112/metrics` (customize with `--metrics-port`).

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `cctp_relayer_wallet_balance` | Gauge | `chain`, `address`, `denom` | Current wallet balance in Wei (Noble balances not exported) |
| `cctp_relayer_chain_latest_height` | Gauge | `chain`, `domain` | Current chain height |
| `cctp_relayer_broadcast_errors_total` | Counter | `chain`, `domain` | Total failed broadcasts (after retries) |
| `cctp_relayer_fast_transfer_allowance` | Gauge | `chain`, `domain`, `token` | Fast Transfer allowance (v2 only) |
| `cctp_relayer_attestation_total` | Counter | `src_chain`, `dest_chain`, `status`, `source_domain`, `dest_domain` | Attestation state transitions |
| `cctp_relayer_attestation_pending` | Gauge | `src_chain`, `dest_chain`, `source_domain`, `dest_domain` | Number of pending attestations |

## 🔑 Minter Private Keys

Private keys can be set in config or via environment variables.

### Environment Variables
Format: `<CHAIN_NAME_UPPER>_PRIV_KEY`

Example for Noble: `NOBLE_PRIV_KEY=0x...`

### Noble Private Key Format
Noble private keys must be hex‑encoded. Export with:
```bash
nobled keys export <KEY_NAME> --unarmored-hex --unsafe
```

## 🌐 API

Simple HTTP API to query message state cache:

```bash
# All messages for a source transaction hash
curl localhost:8000/tx/0x...

# Messages filtered by domain
curl localhost:8000/tx/0x...?domain=0
```

## 📈 State

Message states stored in cache:

| IrisLookupId | Status | SourceDomain | DestDomain | SourceTxHash | DestTxHash | MsgSentBytes | Created | Updated |
|--------------|--------|--------------|------------|--------------|------------|--------------|---------|---------|
| `0x123...` | Created | 0 | 4 | `0x...` | `ABC...` | `bytes...` | timestamp | timestamp |
| `0x123...` | Pending | 0 | 4 | `0x...` | `ABC...` | `bytes...` | timestamp | timestamp |
| `0x123...` | Attested | 0 | 4 | `0x...` | `ABC...` | `bytes...` | timestamp | timestamp |
| `0x123...` | Complete | 0 | 4 | `0x...` | `ABC...` | `bytes...` | timestamp | timestamp |
| `0x123...` | Failed | 0 | 4 | `0x...` | `ABC...` | `bytes...` | timestamp | timestamp |
| `0x123...` | Filtered | 0 | 4 | `0x...` | `ABC...` | `bytes...` | timestamp | timestamp |

## 🔗 Useful Links

- [Relayer Flow Charts](./docs/flows.md)
- [USDC Faucet](https://faucet.circle.com)
- [Circle Docs & Contract Addresses](https://developers.circle.com/cctp/references/contract-addresses)
- [Generating Go ABI Bindings](./README.md#generating-go-abi-bindings)

## 📄 License

MIT