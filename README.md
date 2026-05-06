# monad-tools

Operator tooling for Monad validator nodes, by [BeeHive](https://monad-tech.com). Three single-file bash scripts, zero external dependencies for the diagnostic ones, opinionated defaults that match what we run in production.

| Tool | What | Lines |
|---|---|---|
| **[monad-doctor](doctor/)** | Pre-flight readiness check — hardware/OS/network/security/monad. 32 checks, 30 seconds. | ~500 |
| **[monad-validator-setup](validator-setup/)** | One-shot host configuration — kernel tuning, IO scheduler, chrony, monad apt, UFW, optional monitoring. Idempotent, reversible. | ~400 |
| **[monad-authudp-check](authudp-check/)** | Verify Auth UDP active for the 0.14.3 cutover; optional POST to public compliance tracker. | ~200 |

Companion repo: [BeeHiveTeam/monad-grafana](https://github.com/BeeHiveTeam/monad-grafana) — Prometheus + Grafana monitoring stack with 47-panel dashboard, installs in one command.

---

## Quick start — new validator from zero

```bash
# 1. Clone tools
git clone https://github.com/BeeHiveTeam/monad-tools.git
cd monad-tools

# 2. Check the server is ready
sudo ./doctor/monad-doctor

# 3. If READY (no FAIL): configure host
sudo ./validator-setup/monad-validator-setup --with-monitoring

# 4. Place your validator keys
sudo cp bls_priv_key secp_priv_key validator_id /home/monad/monad-bft/config/
sudo chmod 600 /home/monad/monad-bft/config/{bls,secp}_priv_key
sudo chown -R monad:monad /home/monad/monad-bft/config/

# 5. Start services
sudo systemctl enable --now monad-execution monad-bft monad-rpc

# 6. Verify Auth UDP active (timely with 0.14.3 cutover)
sudo ./authudp-check/monad-authudp-check

# 7. Open Grafana via SSH tunnel
ssh -L 3000:127.0.0.1:3000 -L 9090:127.0.0.1:9090 user@your.server
# → http://localhost:3000  (admin / pass in /opt/monad-grafana/.env)
```

---

## Why we built this

Setting up a Monad validator means walking through 30+ steps from the official docs, copy-pasting configurations and hoping you didn't miss a sysctl tunable. Most operators eventually learn the hard way that:

- `vm.swappiness=60` causes vote_delay spikes when your server has 100+ GB RAM.
- The triedb device must be on `mq-deadline` or random reads tank to ~10k IOPS.
- `systemd-timesyncd` drifts ~50ms and inflates p99 vote_delay metrics.
- Default `unattended-upgrades` will randomly restart your node mid-epoch.
- A consumer NVMe shows fine in `lsblk` but only does 80k random IOPS instead of the 500k+ datacenter SSDs hit.

These tools encode the lessons learned running a real Monad validator into runnable scripts. They don't replace the docs — they enforce them.

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

- BeeHive team — operators of [monad-tech.com](https://monad-tech.com)

## License

MIT — see [LICENSE](LICENSE).
