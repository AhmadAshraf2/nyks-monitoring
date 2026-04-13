# Nyks Monitoring Stack

Grafana + Loki + Prometheus monitoring for the Nyks (Twilight) chain.

```
Validator EC2                         Monitoring EC2
┌──────────────────────┐              ┌──────────────────────┐
│ nyksd (:26660,:1317) │◄── scrape ──│ Prometheus (:9090)   │
│ zkos, zkpass, indexer│              │ Loki       (:3100)   │
│ Promtail ────────────┼── push ────►│ Grafana    (:3000)   │
└──────────────────────┘              └──────────────────────┘
```

## Quick Start — Monitoring Server

```bash
git clone git@github.com:AhmadAshraf2/nyks-monitoring.git
cd nyks-monitoring

# 1. Configure
cp .env.example .env
# Edit .env with your GRAFANA_PASSWORD

# 2. Set your validator IP in prometheus.yml
sed -i 's/VALIDATOR_IP/<your-validator-ip>/g' prometheus.yml

# 3. Start
docker compose up -d

# 4. Open Grafana
# http://<monitoring-ip>:3000 (admin / your password)
```

## Quick Start — Validator Server

```bash
# Copy the promtail/ directory to your validator
scp -r promtail/ ubuntu@<validator-ip>:~/promtail/

# On the validator:
cd ~/promtail

# 1. Set monitoring server IP
sed -i 's/MONITORING_IP/<your-monitoring-ip>/g' promtail-config.yaml

# 2. Start
docker compose up -d
```

## Prerequisites — Enable Prometheus on the Validator

Edit these files on the validator before starting:

**config.toml** — under `[instrumentation]`:
```toml
prometheus = true
prometheus_listen_addr = ":26660"
```

**app.toml** — under `[telemetry]`:
```toml
enabled = true
prometheus-retention-time = 60
```

Restart nyksd after these changes.

## AWS Security Groups

| Port  | Direction          | Allow From       | Purpose              |
|-------|--------------------|------------------|----------------------|
| 26660 | Validator inbound  | Monitoring IP    | Tendermint metrics   |
| 1317  | Validator inbound  | Monitoring IP    | Cosmos SDK metrics   |
| 3100  | Monitoring inbound | Validator IP     | Promtail → Loki     |
| 3000  | Monitoring inbound | Your IP / VPN    | Grafana UI           |

## What's Included

### Dashboard: Nyks Chain Overview
Auto-provisioned at startup with panels for:
- **Chain Health**: block height, block time, peers, mempool, tx rate
- **Consensus**: rounds, validators, missing/byzantine validators
- **Twilight Metrics**: minted sats, mint rate, oracle BTC heights, oracle divergence
- **Node Resources**: goroutines, memory, open file descriptors
- **Logs**: error stream, error rate by module, per-service logs, database logs

### Log Collection (Promtail)
- Auto-discovers all Docker containers via Docker socket
- Parses nyksd structured JSON logs (extracts `level` and `module` labels)
- Collects systemd journal for non-Docker services
- Services covered: nyks, zkos, zkpass, twilight-indexer, all databases

### Metrics (Prometheus)
Two scrape targets:
- `:26660` — Tendermint consensus metrics + custom `promauto` metrics (MintedSatsCounter, OracleBlockGauge)
- `:1317` — Cosmos SDK application telemetry (go-metrics)

## Alert Rules to Configure in Grafana

| Alert               | Query                                                        | For  |
|---------------------|--------------------------------------------------------------|------|
| Chain halted        | `increase(tendermint_consensus_height[5m]) == 0`             | 5m   |
| No peers            | `tendermint_p2p_peers == 0`                                  | 2m   |
| Oracle divergence   | `max(oracle_block_number) - min(oracle_block_number) > 3`    | 15m  |
| High error rate     | error log rate > 0.1/s                                       | 5m   |
| Container down      | `absent_over_time({container="zkos"}[5m])`                   | 5m   |
