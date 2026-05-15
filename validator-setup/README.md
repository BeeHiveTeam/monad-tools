# monad-validator-setup

One-shot host preparation for a Monad validator. Tunes the kernel, configures the firewall, installs the `monad` package, downloads the right config templates for your network — all idempotent, all reversible.

```bash
curl -fsSL https://raw.githubusercontent.com/BeeHiveTeam/monad-tools/main/validator-setup/monad-validator-setup \
  | sudo bash -s -- --network=testnet --with-monitoring
```

Replaces ~30 minutes of copy-pasting from docs with a single command. After it finishes, edit `node.toml`, place keys, start services, validator is online.

## What it does (24 steps — many optional)

Aligned with [docs.monad.xyz/node-ops/full-node-installation](https://docs.monad.xyz/node-ops/full-node-installation) (verified 2026-05-15).

**Mandatory steps run on every install. "Optional" steps prompt the operator interactively (default: skip) or activate via a flag — they cover destructive/operator-specific operations the script can't safely auto-decide.**

### End-to-end install in one command

```bash
sudo MONAD_PKG_VERSION=0.14.3 ./monad-validator-setup --non-interactive \
  --network=testnet \
  --triedb-partition=nvme1n1 --triedb-format \
  --beneficiary=0x<your-address> --node-name=BeeHive-1 \
  --generate-keys --keys-password=auto \
  --sign-name-record --start-services
```
Zero prompts, fresh-server-to-running-validator in a single invocation.

1. **Pre-flight** — runs [`monad-doctor`](../doctor/) and aborts on FAIL (`--skip-preflight` overrides; `--non-interactive` + FAIL = hard abort)
2. **Dependencies** — `apt install -y curl gnupg nvme-cli aria2 jq ethtool ufw iptables` (`iptables-persistent` excluded — Conflicts with `ufw` on Ubuntu 24.04)
3. **Monad user + dirs** — `useradd -m -s /bin/bash monad` + 4 directories under `/home/monad/monad-bft/`
4. **🔧 OPTIONAL — Triedb partitioning** — lists free NVMe disks, operator picks (or `--triedb-partition=DEV`). Checks LBA, offers `nvme format --lbaf=0` if not 512-byte. Runs `parted mklabel gpt` + `mkpart triedb 0% 100%`, writes `/etc/udev/rules.d/99-triedb.rules` with PARTUUID, triggers udev → `/dev/triedb` symlink appears. **DESTRUCTIVE: confirm() before each step.**
5. **`/etc/sysctl.d/99-monad-validator.conf`** — `rmem_max`, `wmem_max`, `netdev_max_backlog`, `file-max=2097152`, `swappiness=1`, `tcp_fastopen=3`
6. **`/etc/security/limits.d/99-monad-validator.conf`** — `nofile=1048576`
7. **IO scheduler** — `mq-deadline` on triedb device. Priority: `--triedb-dev=` → `/dev/triedb` symlink → unused NVMe. Persisted via udev rule (attribute-match, not by name).
8. **TriedB SYMLINK udev rule** — `ENV{ID_PART_ENTRY_UUID}==<PARTUUID>, MODE="0666", SYMLINK+="triedb"`. Idempotent: skipped if step 4 already created it.
9. **chrony** — replaces `systemd-timesyncd` (prompts before).
10. **Monad apt package** — repo + key (auto-dearmor). **Auto-detects version**: GitHub releases/latest (stable only, не RC) → apt-cache fallback (refuses pre-releases). After install: **`apt-mark hold monad`**.
11. **🔧 OPTIONAL — Triedb format** — `systemctl start monad-mpt` (one-shot, per docs). Prompts unless `--triedb-format`. Skipped if `/dev/triedb` missing or monad-mpt already ran successfully.
12. **Bootstrap configs** — `.env` + `node.toml` from `$MF_BUCKET/config/<network>/latest/`. Preserves existing files.
13. **🔧 OPTIONAL — node.toml customize** — prompts for `beneficiary` (0x-validated) + `node_name` (default `BeeHive-<hostname>`), sed-replaces in node.toml. Or `--beneficiary=` / `--node-name=` flags. Auto-skipped if values already set.
14. **🔧 OPTIONAL — Validator key generation** — `monad-keystore create` for SECP + BLS keys. Password mode prompts: **1) auto-generate** (32-byte random, shown ONCE — must back up) or **2) keyboard input** (typed twice, masked). Or `--generate-keys --keys-password=auto|prompt`. Writes `KEYSTORE_PASSWORD` to `/home/monad/.env` (mode 0600) + backup in `/opt/monad/backup/`.
15. **🔧 OPTIONAL — Sign name-record** — `monad-sign-name-record` with auto-detected public IP (`ifconfig.me`). Injects `self_address`, `self_auth_port`, `self_record_seq_num`, `self_name_record_sig` into `[peer_discovery]`. Skipped if signature already present (avoids `self_record_seq_num` bump).
16. **`monad-cruft.timer`** — auto-enables hourly cleanup
17. **otelcol install** — otelcol .deb from GitHub releases, **sha256-verified** against pinned hash. Skips on `otelcol-contrib` setups.
18. **otelcol enable** — `systemctl enable --now otelcol` (if config present)
19. **UFW** — auto-detected SSH port(s) + 8000/tcp+udp + 8001/udp. Prompts before `ufw enable`.
20. **iptables UDP length filter** — persisted via `netfilter-persistent` OR `/etc/ufw/before.rules`. Prompts before insert.
21. **(optional)** `--with-monitoring` → BeeHive monad-grafana installer
22. **(optional)** `--with-vdp-otel` → MF VDP setup script (sha256-verified)
23. **🔧 OPTIONAL — Start services** — `systemctl enable --now monad-execution monad-bft monad-rpc`. Prompts with explicit warnings: "node.toml correct? keys placed? triedb formatted?". Or `--start-services`.
24. **JSON post-install report** at `/var/lib/monad-validator-setup/report-<ts>.json`

🔧 = optional steps (default skip / prompts for opt-in). Six of them turn the script from "configure host" into "fresh-server-to-running-validator in one command."

## What it does NOT do

- **Does not modify any file without backup.** Existing files are copied to `<file>.bak.<timestamp>` before edit.
- **Does not auto-bump `self_record_seq_num`.** If signature already in node.toml, step 15 is a no-op. Re-signing with bumped seq breaks peers that cached the old signature.
- **Does not install monad-grafana / VDP OTel push by default.** Those are opt-in via `--with-monitoring` / `--with-vdp-otel`.

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
journalctl -u monad-bft -f
```

## Interactive prompts at risky steps

Three steps interrupt and ask for confirmation, since each can hurt an operator who has a custom setup:

| Step | What it asks | Why |
|---|---|---|
| `step_chrony` | "Replace systemd-timesyncd with chrony?" / "Install and enable chrony?" | You may already run `ntpd`, OpenNTPD, or a customized chrony build — silent replacement breaks that |
| `step_firewall` | "Enable UFW now?" | A wrong rule (e.g. missing `22/tcp`) would lock you out of SSH |
| `step_iptables_ddos` | "Add this iptables rule?" | An `iptables -I INPUT` inserts ahead of existing rules — operators with custom netfilter setups (frr/bird/k8s/wireguard) should review first |

Answering `n` makes the step exit cleanly with a WARN — re-run later with `--non-interactive` or manually.

`--non-interactive` auto-answers Yes to every prompt (intended for CI).

`--dry-run` doesn't ask at all — it prints what would happen and exits.

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
| `MONAD_APT_KEY_URL` | `${MONAD_APT_REPO}keys/public-key.asc` | ASCII-armored GPG signing key (dearmored on install) |
| `MONAD_APT_KEYRING` | `/etc/apt/keyrings/category-labs.gpg` | local keyring path |
| `MONAD_PKG_VERSION_TESTNET` | (empty — auto) | optional pin if testnet should diverge from mainnet |
| `MONAD_PKG_VERSION_MAINNET` | (empty — auto) | optional pin if mainnet should diverge from testnet |
| `MONAD_PKG_VERSION` | unset (auto-detect) | hard pin a specific version. Without it: GitHub releases/latest → apt-cache fallback. Pre-release suffixes refused unless explicit. |
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
