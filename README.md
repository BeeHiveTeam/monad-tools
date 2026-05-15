# monad-tools

Operator tooling for Monad validator nodes, by [BeeHive](https://bee-hive.work). Two single-file bash scripts, zero external dependencies for the diagnostic one, opinionated defaults that match what we run in production.

| Tool | What | Lines |
|---|---|---|
| **[monad-doctor](doctor/)** | Pre-flight readiness check — hardware/OS/network/security/monad/VDP. **55 checks** in 30 seconds, JSON output, exits 0/1/2. Extended fix hints (Quick fix / Auto / Verify / Why) on every FAIL & WARN. | ~1380 |
| **[monad-validator-setup](validator-setup/)** | One-shot host configuration — **18 steps** matching docs.monad.xyz/node-ops/full-node-installation verbatim: deps → user → tuning → triedb → chrony → monad apt (with hold) → bootstrap configs → otelcol install (sha256-verified) → otelcol enable → UFW → iptables → (optional) VDP OTel push. Idempotent, `--dry-run`, `--network=testnet\|mainnet`. | ~1100 |

Companion repo: [BeeHiveTeam/monad-grafana](https://github.com/BeeHiveTeam/monad-grafana) — Prometheus + Grafana monitoring stack with 47-panel dashboard, installs in one command.

---

## Quick start — new validator from zero

```bash
# 1. Clone tools
git clone https://github.com/BeeHiveTeam/monad-tools.git
cd monad-tools

# 2. Check the server is ready (55 checks, 30 seconds)
sudo ./doctor/monad-doctor

# 3. If READY (no FAIL): configure host. Asks testnet/mainnet interactively,
#    or pass --network=testnet|mainnet. Adds --with-monitoring for Grafana,
#    --with-vdp-otel for MF metrics push compliance (testnet first).
sudo ./validator-setup/monad-validator-setup --network=testnet --with-monitoring

# 4. node.toml has been downloaded from $MF_BUCKET/config/<network>/latest/
#    Edit it to set: beneficiary, node_name, Auth UDP keys
sudo -u monad nano /home/monad/monad-bft/config/node.toml

# 5. Place your validator keys
sudo cp bls_priv_key secp_priv_key validator_id /home/monad/monad-bft/config/
sudo chmod 600 /home/monad/monad-bft/config/{bls,secp}_priv_key
sudo chown -R monad:monad /home/monad/monad-bft/config/

# 6. Start services
sudo systemctl enable --now monad-execution monad-bft monad-rpc

# 7. Verify everything healthy (incl. Auth UDP runtime + config, VDP push)
sudo ./doctor/monad-doctor

# 8. (with --with-monitoring) open Grafana via SSH tunnel
ssh -L 3000:127.0.0.1:3000 -L 9090:127.0.0.1:9090 user@your.server
# → http://localhost:3000  (admin / pass in /opt/monad-grafana/.env)
```

---

## Why we built this

Setting up a Monad validator means walking through 30+ steps from the official docs, copy-pasting configurations and hoping you didn't miss a sysctl tunable. Most operators eventually learn the hard way that:

- `vm.swappiness=60` causes vote_delay spikes when your server has 100+ GB RAM.
- The triedb device must be on `mq-deadline` AND **512-byte LBA** or random reads tank.
- `systemd-timesyncd` drifts ~50ms and inflates p99 vote_delay metrics.
- Default `unattended-upgrades` will randomly restart your node mid-epoch (now configurable: `Automatic-Reboot=false`).
- Kernel `6.8.0-{56..59}` has a known hang bug — operators have lost weeks debugging it.
- **SMT/HyperThreading must be disabled in BIOS** per docs — easy to miss.
- The validator node **must be on bare metal** — Foundation rejects KVM/cloud per VDP eligibility.

These tools encode the lessons learned running a real Monad validator into runnable scripts. They don't replace the docs — they enforce them, with fix commands included for every WARN.

For issues that `monad-validator-setup` cannot fix automatically (BIOS access, destructive operations, reinstall-required cases), see [`docs/MANUAL_FIXES.md`](docs/MANUAL_FIXES.md).

---

## Project status

This is **opinionated, production-tested code** running on the BeeHive validator — a peer in the 200-validator testnet active set. PRs welcome, especially for:

- Additional hardware checks (specific NVMe model warnings, RAID configurations)
- Non-Ubuntu support (Debian, RHEL — currently Ubuntu-only)
- Translations of error messages and docs

If you're running the BeeHive Grafana stack on the same host, you may also want our [monad-grafana](https://github.com/BeeHiveTeam/monad-grafana) repo.

---

## Verifying scripts before running them

These tools take `sudo` — read them first. They're each one bash file, deliberately. Don't pipe random scripts from the internet through `sudo` without at least skimming the source:

```bash
curl -fsSL https://raw.githubusercontent.com/BeeHiveTeam/monad-tools/main/doctor/monad-doctor | less
```

Or clone first, review the diff vs HEAD, then run locally.

---

## Submitting validator info to Monad Foundation

Once your validator is live and stable for ≥1 week, submit metadata to the [`monad-developers/validator-info`](https://github.com/monad-developers/validator-info) repo. This is the canonical channel for Foundation visibility (delegation program, network rewards, etc.).

```yaml
# in PR to validator-info/validators/<your-pubkey>.yaml
moniker: BeeHive
website: https://monad-tech.com
description: Independent validator focused on operator tooling and network observability.
contact:
  email: ops@bee-hive.work
  twitter: <handle>
```

---

## Authors

- BeeHive team — [bee-hive.work](https://bee-hive.work) · operators of [monad-tech.com](https://monad-tech.com)

## License

MIT — see [LICENSE](LICENSE).
