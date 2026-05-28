# monad-tools

Operator tooling for Monad validator nodes, by [BeeHive](https://bee-hive.work). Single-file bash scripts, zero external dependencies for the diagnostic one, opinionated defaults that match what we run in production.

| Tool | What | Lines |
|---|---|---|
| **[monad-doctor](doctor/)** | Pre-flight readiness check — hardware/OS/network/security/monad/VDP. **55 checks** in 30 seconds, JSON output, exits 0/1/2. Extended fix hints (Quick fix / Auto / Verify / Why) on every FAIL & WARN. | ~1380 |
| **[monad-validator-setup](validator-setup/)** | **Fresh-server-to-running-validator** in one command — **25 steps**, including snapshot restore (default ON) and Foundation CVE mitigation (`algif_aead` blacklist + `unattended-upgrades` no-auto-reboot). Defaults to **full-node template** (per docs validator-installation — validators are promoted from a full node after on-chain stake); `--validator` opts into validator template. Auto-detects monad version from GitHub releases/latest, otelcol sha256-verified, UFW SSH-port auto-detect, end-to-end verified on a fresh Hetzner box. | ~2500 |
| **[monad-watchdog](watchdog/)** | Cron recovery for the **`local timeout` deadlock** — a full-node that drops below its upstream-validator target freezes and can't self-recover. Detects it via RPC freeze/gap (statesync-aware, never restarts on a blind 0), restarts services with cooldown + escalation, and **refuses to restart an active validator** (non-burn beneficiary gate). | ~150 |

Companion repo: [BeeHiveTeam/monad-grafana](https://github.com/BeeHiveTeam/monad-grafana) — Prometheus + Grafana + node_exporter monitoring stack (4 containers, 49-panel dashboard, `--public` flag for browser access). Installs in one command.

---

## Quick start — new validator from zero

```bash
# 1. Clone tools
git clone https://github.com/BeeHiveTeam/monad-tools.git
cd monad-tools

# 2. Check the server is ready (55 checks, 30 seconds)
sudo ./doctor/monad-doctor

# 3. If READY (no FAIL): configure host end-to-end.
#    Default = full-node template (per docs validator-installation flow).
#    Snapshot restore is ON by default (--no-snapshot-restore to skip).
#    --with-vdp-otel is auto-enabled for testnet (VDP eligibility starts there).
sudo ./validator-setup/monad-validator-setup --network=testnet \
  --triedb-partition=nvme1n1 --start-services

# 4. After node is fully synced AND you have ≥100k MON stake on the
#    beneficiary address, PROMOTE to validator on the same machine:
sudo ./validator-setup/monad-validator-setup --network=testnet --validator \
  --beneficiary=0x<your_real_rewards_address> --node-name=<your_moniker> \
  --generate-keys --keys-password=auto
# The script detects the full→validator transition, backs up the existing
# node.toml, swaps in the validator template, generates SECP+BLS keys, and
# signs the Auth UDP name-record.

# 5. Verify everything healthy (incl. sync gap, Auth UDP, VDP push)
sudo ./doctor/monad-doctor

# 6. Install monitoring (separate repo, one command, optional --public for
#    browser access). See https://github.com/BeeHiveTeam/monad-grafana
curl -fsSL https://raw.githubusercontent.com/BeeHiveTeam/monad-grafana/main/install.sh \
  | sudo bash -s -- --public
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
