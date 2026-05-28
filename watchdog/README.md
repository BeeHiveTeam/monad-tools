# monad-watchdog

Detects and recovers a **stuck Monad full-node**. Single bash script, runs from cron, zero dependencies beyond `curl` + `python3`-free (pure shell + `awk`).

It targets one specific, nasty failure mode we hit in production (see [The problem](#the-problem)) and recovers it with a guarded `systemctl restart` — while refusing to touch a node that looks like an active validator.

```
$ /usr/local/bin/monad-watchdog
2026-05-28T17:47:25Z block=34681120 head=34681125 gap=5 advanced=812/run frozen=0 transport_sessions=19
2026-05-28T17:47:25Z OK — advancing 812/run, gap=5
```

## The problem

A Monad full-node that drops below its `NUM_UPSTREAM_VALIDATORS` target stops
receiving valid proposals and loops on `local timeout` forever. Because its
block-tree root stops advancing, the runtime's auto-statesync trigger (gated
behind a root advancing past a threshold) **never fires** — so the node neither
catches up nor restarts itself. A process restart is the only recovery.

What made it hard to even *notice*: the node's discovery routing table keeps
reporting hundreds of **known** peers while the count of **established** sessions
collapses. Our Grafana "Active peers" panel was wired to
`monad_peer_disc_num_peers` (≈350, discovery-known) instead of
`monad_wireauth_udp_state_transport_sessions` (≈2, actually connected), so the
dashboard looked healthy through the entire outage. We fixed the panel in
[monad-grafana](https://github.com/BeeHiveTeam/monad-grafana) and wrote this
watchdog so the recovery is automatic next time.

## Decision model (designed to avoid false restarts)

- **Only RPC-derived signals trigger a restart:** local head FROZEN (gained
  fewer than `MIN_ADVANCE` blocks since last run) or a large `GAP` behind the
  network head. The local RPC is read on the host and guarded — if it is
  unreachable we **alert and do nothing** (never restart on a blind `0`).
- **Statesync-aware:** when the RPC is down *and* `monad_statesync_syncing=1`,
  the node is legitimately recovering — we log quietly and take no action
  instead of alert-spamming every run.
- **Session count is context only.** It is read locally from otelcol
  (`:8889/metrics`, no Prometheus dependency) using
  `monad_wireauth_udp_state_transport_sessions` (established sessions, **not**
  `*_total_sessions`, which counts in-progress handshakes too). It is logged but
  **never triggers a restart by itself.**
- **Cooldown + escalation:** after a restart it waits `RESTART_COOLDOWN` before
  acting again; if still stuck it escalates to an alert (a node that far behind
  needs a Hard Reset / snapshot restore — which this script intentionally does
  **not** do automatically).
- **Validator safety gate:** if `node.toml` has a non-burn `beneficiary` the
  node looks like an active validator — a restart in your proposal slot means
  missed blocks. The script **refuses to restart** unless
  `ALLOW_VALIDATOR_RESTART=1` is set explicitly.

## Install

```bash
sudo install -m 0755 watchdog/monad-watchdog /usr/local/bin/monad-watchdog

# every 5 minutes
( sudo crontab -l 2>/dev/null | grep -v monad-watchdog
  echo '*/5 * * * * /usr/local/bin/monad-watchdog >> /var/log/monad-watchdog.log 2>&1'
) | sudo crontab -
```

## Configuration (override via env)

| Var | Default | Meaning |
|---|---|---|
| `RPC` | `http://localhost:8080` | local node RPC |
| `NET_RPC` | `https://testnet-rpc.monad.xyz` | network-head reference |
| `OTEL` | `http://127.0.0.1:8889/metrics` | local otelcol exporter (session context) |
| `SERVICES` | `monad-bft monad-execution monad-rpc` | services to restart |
| `MIN_ADVANCE` | `50` | min blocks/run the local head must gain to count as alive |
| `GAP_ALERT` | `2000` | blocks behind network head that is clearly unhealthy |
| `RESTART_COOLDOWN` | `1800` | min seconds between restarts |
| `ALLOW_VALIDATOR_RESTART` | `0` | set `1` to allow restarting a node with a non-burn beneficiary |
| `NODE_TOML` | `/home/monad/monad-bft/config/node.toml` | path used for the validator gate |
| `ALERT_WEBHOOK` | _(empty)_ | optional Slack/Discord/Telegram webhook for alerts |

## Exit codes

`0` healthy / no action · `1` restarted this run · `2` alerted, manual action needed.

> Tune `MIN_ADVANCE` to your cron interval and network block rate (testnet ≈ 2.5
> blocks/s ≈ 750 blocks per 5-min run). Full-nodes only by default — read the
> validator gate above before running on a staked validator.
