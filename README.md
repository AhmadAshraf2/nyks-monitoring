# Nyks Monitoring Stack

Grafana + Loki + Prometheus monitoring for the Nyks (Twilight) chain.

```
Validators / Sentries                 Monitoring Server
┌──────────────────────┐              ┌──────────────────────┐
│ nyksd     (:26660)   │◄── scrape ──│ Prometheus (:9090)   │
│ btcoracle            │              │ Loki       (:3100)   │
│ Promtail  ───────────┼── push ────►│ Grafana    (:3000)   │
└──────────────────────┘              └──────────────────────┘
  x7 validators + x2 sentries
```

## Servers

| Server | Services | Promtail |
|--------|----------|----------|
| validator-node-1 to 7 | nyksd + btcoracle | Yes |
| sentry-node-1 to 2 | nyksd only | Yes |
| monitoring server | Grafana + Loki + Prometheus | No |

---

## Step 1: Enable Prometheus on ALL 9 Nodes

SSH into each validator and sentry and edit:

**config.toml** — `[instrumentation]` section:
```toml
prometheus = true
prometheus_listen_addr = ":26660"
```

**app.toml** — `[telemetry]` section:
```toml
enabled = true
prometheus-retention-time = 60
```

Restart nyksd on each server after editing.

---

## Step 2: Deploy Monitoring Server

```bash
git clone git@github.com:AhmadAshraf2/nyks-monitoring.git
cd nyks-monitoring

# Configure
cp .env.example .env
nano .env  # set GRAFANA_PASSWORD

# Set all validator/sentry IPs in prometheus.yml
nano prometheus.yml  # replace VALIDATOR_NODE_X_IP and SENTRY_NODE_X_IP

# Start
docker compose up -d
```

Verify:
- http://<monitoring-ip>:9090/targets — all 9 nodes should show UP
- http://<monitoring-ip>:3000 — Grafana login (admin / your password)

---

## Step 3: Deploy Promtail on Each Validator/Sentry

For **each** of the 9 servers:

```bash
# From your local machine, copy promtail folder to the server
scp -r promtail/ ubuntu@<server-ip>:~/promtail/

# SSH into the server
ssh ubuntu@<server-ip>
cd ~/promtail

# Set the monitoring server IP
sed -i 's/MONITORING_IP/<monitoring-server-ip>/g' promtail-config.yaml

# Set the hostname (e.g. validator-node-1, sentry-node-2)
sed -i 's/HOSTNAME/<server-name>/g' promtail-config.yaml

# On sentry nodes: comment out the btcoracle section
# nano promtail-config.yaml

# Start
docker compose up -d
```

Verify: In Grafana → Explore → Loki → `{host="validator-node-1"}` should show logs.

---

## AWS Security Groups

| Port  | Direction          | Allow From        | Purpose            |
|-------|--------------------|-------------------|--------------------|
| 26660 | Validator inbound  | Monitoring IP     | Tendermint metrics |
| 1317  | Validator inbound  | Monitoring IP     | Cosmos SDK metrics |
| 3100  | Monitoring inbound | All validator IPs | Promtail → Loki   |
| 3000  | Monitoring inbound | Your IP / VPN     | Grafana UI         |

Do NOT open any of these to 0.0.0.0/0.

---

## Dashboard

A pre-built dashboard (**Nyks Chain Overview**) is auto-provisioned with:
- **Chain Health**: block height, block time, peers, mempool, tx rate
- **Consensus**: rounds, validators, missing/byzantine validators
- **Twilight Metrics**: minted sats, mint rate, oracle BTC heights, oracle divergence
- **Node Resources**: goroutines, memory, open file descriptors
- **Logs**: error stream, error rate by module, per-service logs

Use the `host` label to filter by server (e.g. `{host="validator-node-3"}`).
Use the `service` label to filter by service (e.g. `{service="btcoracle"}`).

---

## Alert Rules to Configure

| Alert               | Query                                                     | For  |
|---------------------|-----------------------------------------------------------|------|
| Chain halted        | `increase(tendermint_consensus_height[5m]) == 0`          | 5m   |
| No peers            | `tendermint_p2p_peers == 0`                               | 2m   |
| Oracle divergence   | `max(oracle_block_number) - min(oracle_block_number) > 3` | 15m  |
| High error rate     | error log rate > 0.1/s                                    | 5m   |

---

## Log File Paths

Promtail reads these files from `/var/log/` (adjust in `promtail-config.yaml` if your paths differ):

| Service | Path |
|---------|------|
| nyksd | `/var/log/nyksd*.log` |
| btcoracle | `/var/log/btcoracle*.log` |
| supervisor | `/var/log/supervisor/*.log` |
