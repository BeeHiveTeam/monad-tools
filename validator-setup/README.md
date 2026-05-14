# monad-validator-setup

One-shot host preparation for a Monad validator. Tunes the kernel, configures the firewall, installs the `monad` package, downloads the right config templates for your network — all idempotent, all reversible.

```bash
curl -fsSL https://raw.githubusercontent.com/BeeHiveTeam/monad-tools/main/validator-setup/monad-validator-setup \
  | sudo bash -s -- --network=testnet --with-monitoring
```

Replaces ~30 minutes of copy-pasting from docs with a single command. After it finishes, edit `node.toml`, place keys, start services, validator is online.

## What it does (14 steps)

1. **Pre-flight** — runs [`monad-doctor`](../doctor/) and aborts on FAIL (override with `--skip-preflight`)
2. **Dependencies** — `apt install -y curl nvme-cli aria2 jq ethtool ufw iptables iptables-persistent` (per docs)
3. **Monad user + dirs** — `useradd -m -s /bin/bash monad` and the 4 directories under `/home/monad/monad-bft/` listed in docs (`config`, `ledger`, `config/forkpoint`, `config/validators`)
4. **`/etc/sysctl.d/99-monad-validator.conf`** — `rmem_max`, `wmem_max`, `netdev_max_backlog`, `file-max=2097152`, `swappiness=1`, `tcp_fastopen=3`
5. **`/etc/security/limits.d/99-monad-validator.conf`** — `nofile=1048576`
6. **IO scheduler** — `mq-deadline` on the triedb device. Selection priority: `--triedb-dev=` flag → `/dev/triedb` symlink → unused NVMe (largest, never picks an OS-RAID member). Persisted via udev rule that matches by attribute (`rotational=0`, `nvme*`) so swapping the disk doesn't silently regress.
7. **TriedB SYMLINK udev rule** — writes `/etc/udev/rules.d/99-triedb.rules` with `ENV{ID_PART_ENTRY_UUID}==<PARTUUID>, MODE="0666", SYMLINK+="triedb"` exactly per docs format. Skips gracefully if no triedb partition exists yet.
8. **chrony** — replaces `systemd-timesyncd` (sub-millisecond NTP for accurate `vote_delay`)
9. **Monad apt package** — adds `https://pkg.category.xyz/` (Category Labs, official) using deb822 `.sources` format with signing key at `/etc/apt/keyrings/category-labs.gpg`. Installs `monad=$MONAD_PKG_VERSION`. Default is **per-network per docs**: testnet → `0.14.3`, mainnet → `0.14.2`. Override with `MONAD_PKG_VERSION=` env var (empty = install latest available).
10. **Bootstrap configs** — downloads `.env` and `node.toml` from `$MF_BUCKET/config/<network>/latest/` per docs. Validator gets `node.toml`; full-node gets `full-node-node.toml` (toggle with `--full-node`). Idempotent: preserves existing files (no overwrite).
11. **`monad-cruft.timer`** — auto-enables hourly cleanup (was previously left to operator)
12. **UFW** — `22/tcp` (SSH), `8000/tcp+udp` (P2P), `8001/udp` (Auth UDP only — not TCP)
13. **iptables UDP DDoS filter** — `iptables -I INPUT -p udp --dport 8000 -m length --length 0:1400 -j DROP` per docs, persisted via `netfilter-persistent save`
14. **(optional)** `--with-monitoring` runs the BeeHive monad-grafana installer
15. **(optional)** `--with-vdp-otel` runs the MF VDP setup script after sha256 verification (pinned default; override with `VDP_OTEL_SETUP_SHA256=` env if MF rotates). **Bails cleanly if you have `otelcol-contrib` instead of plain `otelcol`** (different config path) — apply the equivalent manually per [docs/vdp-otel-push.md](../docs/vdp-otel-push.md). Requires `KEYSTORE_PASSWORD` in `/home/monad/.env` (set when generating keys), so put your keys in place first.
16. JSON post-install report at `/var/lib/monad-validator-setup/report-<ts>.json`

## What it does NOT do

- **Does not generate or import validator keys.** Keys are sensitive — you place them yourself in `/home/monad/monad-bft/config/`.
- **Does not start `monad-bft.service`.** Without keys this would just crash-loop.
- **Does not modify any file without backup.** Existing files are copied to `<file>.bak.<timestamp>` before edit.
- **Does not install OTEL collector by default.** Monad's apt package installs plain `otelcol` itself. With `--with-vdp-otel` (added 2026-05-14), the MF setup script is fetched + sha256-verified + run after key placement to satisfy VDP push compliance. Plain-otelcol setups get full automation; `otelcol-contrib` setups are detected and skipped with a pointer to manual instructions.

## Run

```bash
# Interactive (asks testnet/mainnet)
sudo ./monad-validator-setup

# Skip the prompt
sudo ./monad-validator-setup --network=testnet
sudo ./monad-validator-setup --network=mainnet

# Full-node template instead of validator
sudo ./monad-validator-setup --network=testnet --full-node

# Show what would change without doing it
sudo ./monad-validator-setup --network=testnet --dry-run

# Non-interactive (CI / automation) — --network is REQUIRED
sudo ./monad-validator-setup --network=testnet --non-interactive

# With Grafana monitoring stack
sudo ./monad-validator-setup --network=testnet --with-monitoring

# Pin a specific NVMe as triedb device (overrides /dev/triedb autodetect)
sudo ./monad-validator-setup --network=testnet --triedb-dev=nvme1n1

# Skip preflight if you already ran monad-doctor
sudo ./monad-validator-setup --network=testnet --skip-preflight
```

## After install — next steps

The installer prints these explicitly. Summarized:

```bash
# 1. Edit node.toml (template was downloaded to /home/monad/monad-bft/config/node.toml)
sudo -u monad nano /home/monad/monad-bft/config/node.toml
#    Set: beneficiary, node_name, Auth UDP keys
#    For Auth UDP: run monad-sign-name-record to generate self_name_record_sig

# 2. Place keys
sudo cp bls_priv_key secp_priv_key validator_id /home/monad/monad-bft/config/
sudo chmod 600 /home/monad/monad-bft/config/{bls,secp}_priv_key
sudo chown -R monad:monad /home/monad/monad-bft/config/

# 3. Start services
sudo systemctl enable --now monad-execution monad-bft monad-rpc

# 4. Verify health
sudo /opt/monad-tools/doctor/monad-doctor
sudo /opt/monad-tools/authudp-check/monad-authudp-check
journalctl -u monad-bft -f
```

## Idempotency

Safe to re-run any number of times. Each step:
- Checks current state first ("is chrony already active?")
- Skips changes if state matches desired
- Backs up before modifying
- Files written are deterministic — re-running produces identical content
- `step_config_files` preserves existing `.env` / `node.toml` (delete first to re-download)

## Error handling

If any step fails, `do_install` collects the failed names and exits 1 with a summary instead of pretending success:

```
═══════════════════════════════════════════════════
  Setup INCOMPLETE — failed steps: monad-install firewall
═══════════════════════════════════════════════════
  See log for details: /tmp/monad-validator-setup-...log
  Re-run after fixing, or open an issue.
```

## Uninstall

```bash
sudo ./monad-validator-setup --uninstall
```

Removes the config files written by the installer:
- `/etc/sysctl.d/99-monad-validator.conf`
- `/etc/security/limits.d/99-monad-validator.conf`
- `/etc/udev/rules.d/60-monad-triedb-scheduler.rules`

It does NOT remove:
- `/etc/udev/rules.d/99-triedb.rules` (TriedB SYMLINK — required for monad to find triedb at boot)
- The `monad` apt package — `sudo apt-mark unhold monad && sudo apt remove monad` if you want
- chrony, ufw, monad-grafana, iptables rules — independent of this tool
- Validator keys, configs, chain data — your responsibility

## Logs and reports

- Per-run log: `/tmp/monad-validator-setup-YYYYMMDD-HHMMSS.log`
- Post-install JSON report: `/var/lib/monad-validator-setup/report-YYYYMMDD-HHMMSS.json`

## Configurable values

Override via environment variable:

| Variable | Default | What it controls |
|---|---|---|
| `MONAD_APT_REPO` | `https://pkg.category.xyz/` | Category Labs apt source URL (per docs) |
| `MONAD_APT_SUITE` | `noble` | apt suite (only `noble` exists upstream) |
| `MONAD_APT_KEY_URL` | `${MONAD_APT_REPO}/category-labs.gpg` | GPG signing key |
| `MONAD_APT_KEYRING` | `/etc/apt/keyrings/category-labs.gpg` | local keyring path |
| `MONAD_PKG_VERSION_TESTNET` | `0.14.3` | pinned testnet version (per docs) |
| `MONAD_PKG_VERSION_MAINNET` | `0.14.2` | pinned mainnet version (per docs) |
| `MONAD_PKG_VERSION` | unset (auto-pick by `--network=`) | overrides both defaults; set empty for `latest` |
| `MF_BUCKET` | `https://bucket.monadinfra.com` | Foundation config bucket |
| `SYSCTL_RMEM` / `SYSCTL_WMEM` | `16777216` | UDP receive/send buffer max (raptorcast) |
| `SYSCTL_FILE_MAX` | `2097152` | system-wide file descriptor limit |
| `SYSCTL_SWAPPINESS` | `1` | discourage swapping (validator must not swap) |
| `ULIMIT_NOFILE` | `1048576` | per-user file descriptor limit |

Example:
```bash
sudo MONAD_PKG_VERSION=0.14.3 ./monad-validator-setup --network=testnet --non-interactive
```

## License

MIT — see [LICENSE](../LICENSE).
