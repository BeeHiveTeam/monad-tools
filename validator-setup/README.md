# monad-validator-setup

One-shot host preparation for a Monad node. Goes fresh-server-to-running-validator in a single command — tunes the kernel, configures the firewall, installs the `monad` package, partitions the TrieDB NVMe, downloads the right config templates, applies the Foundation snapshot, generates keys, signs the Auth UDP name-record, and (opt-in) starts services.

All idempotent, all reversible.

```bash
curl -fsSL https://raw.githubusercontent.com/BeeHiveTeam/monad-tools/main/validator-setup/monad-validator-setup \
  | sudo bash -s -- --network=testnet
```

Replaces ~30 minutes of copy-pasting from docs with a single command.

## What it does (25 steps — most automatic)

Aligned with [docs.monad.xyz/node-ops/full-node-installation](https://docs.monad.xyz/node-ops/full-node-installation) (verified 2026-05-15).

**Default mode is `full-node` template**, per docs validator-installation: *"Start by configuring a node according to the full node installation instructions"* — validators are promoted from a synced full-node after on-chain stake is in. Pass `--validator` to opt into the validator template (only useful once you have ≥100k MON ready).

### End-to-end install in one command

```bash
sudo ./monad-validator-setup --non-interactive \
  --network=testnet \
  --triedb-partition=nvme1n1 --allow-nvme-reformat \
  --start-services
```

Zero prompts. The script:
- partitions `nvme1n1` for `/dev/triedb`
- installs `monad` (auto-detect latest stable from GitHub releases)
- writes the full-node template with `beneficiary=0x000…000` (burn address)
- applies the Foundation snapshot (testnet < 5 min)
- starts `monad-bft` + `monad-execution` + `monad-rpc`
- pushes metrics to MF VDP endpoint (`--with-vdp-otel` is auto-enabled for testnet)

### Step list

1. **Pre-flight** — runs [`monad-doctor`](../doctor/), aborts on FAIL (`--skip-preflight` overrides; `--non-interactive` + FAIL = hard abort)
2. **Dependencies** — `apt install -y curl gnupg nvme-cli aria2 jq ethtool ufw iptables parted python3` (`iptables-persistent` excluded — Conflicts with `ufw` on Ubuntu 24.04)
3. **Monad user + dirs** — `useradd -m -s /bin/bash monad` + 4 directories under `/home/monad/monad-bft/`
4. **🔧 OPTIONAL — Triedb partitioning** — lists free NVMe disks, operator picks (or `--triedb-partition=DEV`). Verifies LBA = 512 bytes (offers `nvme format --lbaf=0` if not). Runs `parted mklabel gpt` + `mkpart triedb 0% 100%`, writes udev rule with PARTUUID → `/dev/triedb` symlink. **DESTRUCTIVE: confirm() each step; `--allow-nvme-reformat` for non-interactive.**
5. **ulimits** — `/etc/security/limits.d/99-monad-validator.conf` with `nofile=1048576`. Note: we **do NOT** write a sysctl drop-in for network buffers — the `monad` apt package ships `/etc/sysctl.d/90-monad-network-buffer.conf` with `rmem_max=62500000` itself; overriding it caused monad-bft panics in older script versions.
6. **IO scheduler** — `mq-deadline` on triedb device (per docs). Persisted via udev rule (attribute-match, not by name).
7. **TriedB SYMLINK udev rule** — `ENV{ID_PART_ENTRY_UUID}==<PARTUUID>, MODE="0666", SYMLINK+="triedb"`. Idempotent: skipped if step 4 already created it.
8. **CVE mitigations (Foundation advisory)** — blacklists `algif_aead` (CVE-2026-31431, Foundation Discord 2026-04-30) + sets `Unattended-Upgrade::Automatic-Reboot "false"` (so apt security updates don't reboot mid-block-proposal). Both idempotent, confirm-prompted.
9. **chrony** — replaces `systemd-timesyncd` (prompts before — you may already run `ntpd` / OpenNTPD)
10. **Monad apt package** — repo + key (auto-dearmor). **Auto-detects version**: GitHub releases/latest (stable only, not RC) → apt-cache fallback (refuses pre-releases). After install: **`apt-mark hold monad`**.
11. **Triedb format** — `systemctl start monad-mpt` (one-shot, per docs). Auto-runs if `/dev/triedb` present and monad-mpt hasn't completed yet.
12. **Bootstrap configs** — `.env` + `node.toml` from `$MF_BUCKET/config/<network>/latest/`. **Detects existing template mismatch**: if a validator-template node.toml is present but you asked for full-node (or vice versa), backs up + replaces. This catches the recovery flow when a validator without on-chain stake gets refused by bootstraps and needs to revert to full-node mode.
13. **node.toml customize** — auto-injects `beneficiary` + `node_name` if provided via flags (`--beneficiary=0x...`, `--node-name=...`). For `--full-node` mode the beneficiary is forced to `0x000…000`. Skipped if already set.
14. **Validator key generation** — runs `monad-keystore create` for SECP + BLS keys when `--generate-keys` is set. Password mode: `--keys-password=auto` (32-byte random, shown once + backed up in `/opt/monad/backup/`) or `prompt` (typed twice).
15. **Sign name-record** — `monad-sign-name-record` with auto-detected public IP. Injects `self_address`, `self_auth_port=8001`, `self_record_seq_num`, `self_name_record_sig` into `[peer_discovery]`. Skipped if signature already present (avoids `self_record_seq_num` bump that would break peer caches).
16. **`monad-cruft.timer`** — auto-enables hourly cleanup
17. **otelcol install** — otelcol v0.139.0 `.deb` from GitHub releases, **sha256-verified** against pinned hash. Plus `apt-mark hold otelcol`. Skipped on `otelcol-contrib` setups (operator runs custom collector).
18. **otelcol enable** — `systemctl enable --now otelcol` (if config present)
19. **UFW** — auto-detected SSH port(s) + 8000/tcp+udp + 8001/udp. Prompts before `ufw enable`.
20. **iptables UDP length filter** — `iptables -I INPUT -p udp --dport 8000 -m length --length 0:1400 -j DROP` per docs. Persisted via `netfilter-persistent` OR `/etc/ufw/before.rules`. Prompts before insert.
21. **(optional)** `--with-monitoring` → [BeeHive monad-grafana](https://github.com/BeeHiveTeam/monad-grafana) installer
22. **Snapshot restore (default ON)** — `curl $MF_BUCKET/scripts/<network>/restore-from-snapshot.sh | bash` per docs hard-reset. Auto-skips when ledger already has ≥100 MB of data and no role transition happened. Use `--no-snapshot-restore` to skip even on fresh installs.
23. **🔧 OPTIONAL — Start services** — `systemctl enable --now monad-bft monad-execution monad-rpc`. Prompts with explicit warnings: "node.toml correct? keys placed? triedb formatted?". Or `--start-services`.
24. **VDP OTel push** — runs MF setup script (`bucket.monadinfra.com/tmp/otel-validator-setup.sh`, sha256-verified). Auto-enabled for `--network=testnet` (VDP eligibility track starts on testnet). Runs **after** start-services so the MF script's `secp_key` verify against `:8889/metrics` has data to read. Soft-fail: if MF script verify fails, the install is not marked failed — config still applied, operator re-runs after node syncs.
25. **JSON post-install report** at `/var/lib/monad-validator-setup/report-<ts>.json`

🔧 = the two truly opt-in steps. Step 4 is destructive, step 23 is the "go live" gate. Everything else either runs automatically or is harmless on re-run.

## What it does NOT do

- **Does not modify any file without backup.** Existing files are copied to `<file>.bak.<timestamp>` before edit.
- **Does not auto-bump `self_record_seq_num`.** If signature already in node.toml, the sign step is a no-op. Re-signing with bumped seq breaks peers that cached the old signature.
- **Does not write a sysctl drop-in for network buffers.** The `monad` apt package's `/etc/sysctl.d/90-monad-network-buffer.conf` is authoritative; the script removes legacy `/etc/sysctl.d/99-monad-validator.conf` if it survived from an older install.
- **Does not call `addValidator()` on-chain.** That's a separate off-server step requiring 100k MON in your wallet — see the install banner for next steps.

## Run

```bash
# Interactive (asks testnet/mainnet, picks safer defaults)
sudo ./monad-validator-setup

# Skip the prompt
sudo ./monad-validator-setup --network=testnet
sudo ./monad-validator-setup --network=mainnet

# Default = full-node template. Promote to validator after on-chain stake:
sudo ./monad-validator-setup --network=testnet --validator \
  --beneficiary=0x<real_rewards_addr> --node-name=<unique_moniker> \
  --generate-keys --keys-password=auto

# Show what would change without doing it
sudo ./monad-validator-setup --network=testnet --dry-run

# Non-interactive (CI / automation) — --network is REQUIRED
sudo ./monad-validator-setup --network=testnet --non-interactive

# With Grafana monitoring stack (4 containers, 49-panel dashboard)
sudo ./monad-validator-setup --network=testnet --with-monitoring

# Pin a specific NVMe as triedb device (overrides /dev/triedb autodetect)
sudo ./monad-validator-setup --network=testnet --triedb-dev=nvme1n1

# Skip preflight if you already ran monad-doctor separately
sudo ./monad-validator-setup --network=testnet --skip-preflight

# Skip snapshot restore (will sync from genesis — hours on testnet, weeks on mainnet)
sudo ./monad-validator-setup --network=testnet --no-snapshot-restore
```

## Flags

```
--network=testnet|mainnet           Which Monad network. Asked interactively if missing.
--full-node                         Full-node template (DEFAULT).
--validator                         Validator template (use only after on-chain stake).
                                    Without on-chain registration, bootstraps refuse
                                    to share peer-list (anti-Sybil), so the node never
                                    finds peers. Promote a full-node after stake is in.
--triedb-partition=DEV              DESTRUCTIVE: partition this NVMe for triedb.
--triedb-dev=DEV                    IO-scheduler target (no partitioning).
--allow-nvme-reformat               Allow nvme format --lbaf=0 under --non-interactive.
--beneficiary=0x...                 Rewards address; written into node.toml.
--node-name=NAME                    Unique moniker; written into node.toml.
--generate-keys                     monad-keystore create SECP + BLS keys.
--keys-password=auto|prompt         auto = generate + show; prompt = type twice.
--snapshot-restore                  Force docs hard-reset snapshot (DEFAULT ON).
--no-snapshot-restore               Skip snapshot restore (sync from genesis).
--with-monitoring                   Install BeeHive monad-grafana stack.
--with-vdp-otel                     Run MF VDP OTel push setup. AUTO-ENABLED for testnet.
--no-vdp-otel                       Opt out of VDP push setup (override testnet default).
--start-services                    systemctl enable+start monad-* at end.
--skip-preflight                    Skip monad-doctor pre-check.
--non-interactive                   No prompts; missing flags use safe defaults.
--dry-run                           Show what would change; no system modifications.
--uninstall                         Remove our config files (apt pkg + user stay).
--uninstall-deep                    Full wipe: apt purge monad + userdel monad + configs.
                                    Internally orders apt purge BEFORE userdel because
                                    monad's postrm runs `crontab -u monad -r` which
                                    fails if the user is already gone.
```

## Interactive prompts at risky steps

The script confirms before each of these — each can hurt an operator with a custom setup:

| Step | What it asks | Why |
|---|---|---|
| `step_triedb_partition` | "Wipe this NVMe and create a TriedB partition?" | Destructive — destroys existing data on the disk |
| `step_cve_mitigation` | "Blacklist algif_aead kernel module?" + "Disable unattended-upgrades auto-reboot?" | Reversible but operator-visible — Foundation advisory mitigation |
| `step_chrony` | "Replace systemd-timesyncd with chrony?" | You may already run `ntpd` / OpenNTPD; silent replacement breaks that |
| `step_firewall` | "Enable UFW now?" | Wrong rule (e.g. missing SSH port) would lock you out |
| `step_iptables_ddos` | "Add this iptables rule?" | `iptables -I INPUT` inserts ahead of existing rules — review if you run frr/bird/k8s/wireguard |
| `step_snapshot_restore` | "Run snapshot restore now?" | Tens of GB download (mainnet) — auto-skipped if ledger already populated |
| `step_start_services` | "Start services now?" | Goes live on the network — node.toml + keys must be correct |

Answering `n` makes the step exit cleanly with a WARN — re-run later or take over manually.

`--non-interactive` auto-answers Yes to every prompt (intended for CI).
`--dry-run` doesn't ask at all — prints what would happen and exits.

## Idempotency

Safe to re-run any number of times. Each step:
- Checks current state first ("is chrony already active?")
- Skips changes if state matches desired
- Backs up before modifying
- Files written are deterministic — re-running produces identical content
- `step_config_files` preserves existing `.env` / `node.toml`, detects mismatched template (validator vs full) and offers backup+swap
- `step_snapshot_restore` skips when ledger has data unless a role transition happened

The `ROLE_TRANSITION` mechanism makes promote/demote safe: on re-run with a different `--full-node` / `--validator` choice than what's currently on disk, the script stops services, backs up the existing config, swaps templates, and forces a fresh snapshot.

## Error handling

If any step fails, `do_install` collects the failed names and exits 1 with a summary instead of pretending success:

```
═══════════════════════════════════════════════════
  Setup INCOMPLETE — failed steps: monad-install firewall
═══════════════════════════════════════════════════
  See log for details: /tmp/monad-validator-setup-...log
  Re-run after fixing, or open an issue.
```

The VDP OTel push step is a deliberate exception: if the MF verify step fails (no `secp_key` in metrics yet because monad-bft just started), the install is NOT marked failed — the config was applied, operator just re-runs `--with-vdp-otel` after sync.

## Uninstall

```bash
sudo ./monad-validator-setup --uninstall        # configs only
sudo ./monad-validator-setup --uninstall-deep   # configs + apt purge + userdel
```

`--uninstall` removes the config files written by the installer:
- `/etc/security/limits.d/99-monad-validator.conf`
- `/etc/udev/rules.d/60-monad-triedb-scheduler.rules`
- `/etc/modprobe.d/blacklist-algif_aead.conf`
- Also cleans up legacy `/etc/sysctl.d/99-monad-validator.conf` if it survived from older script versions.

It does NOT remove:
- `/etc/udev/rules.d/99-triedb.rules` (TriedB SYMLINK — required for monad to find triedb at boot)
- The `monad` apt package — use `--uninstall-deep` if you want full wipe
- chrony, ufw, monad-grafana, iptables rules — independent of this tool
- Validator keys, configs, chain data — your responsibility

`--uninstall-deep` additionally: stops services, `apt purge monad`, removes `/etc/apt/sources.list.d/category-labs.sources` + keyring, `userdel monad`, removes `/home/monad`. **Order matters**: `apt purge` runs **before** `userdel` because monad's postrm executes `crontab -u monad -r` which fails if the user is already gone.

## Logs and reports

- Per-run log: `/tmp/monad-validator-setup-YYYYMMDD-HHMMSS.log`
- Post-install JSON report: `/var/lib/monad-validator-setup/report-YYYYMMDD-HHMMSS.json` (includes detected versions, generated public keys, services started, sha256 of MF scripts run)

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
| `MONAD_PKG_VERSION` | unset (auto-detect) | hard pin. Without it: GitHub releases/latest (stable only) → apt-cache fallback. Pre-release suffixes (`~rc/~alpha/~beta`) refused unless explicit. |
| `MF_BUCKET` | `https://bucket.monadinfra.com` | Foundation config bucket |
| `OTEL_VERSION` | `0.139.0` | otelcol pinned version (matches docs.monad.xyz) |
| `OTEL_CHECKSUM` | `1a1576dde7d51fa7094f4963ceaff37c91ac7b9c9593ba735a3a328ec6f8acd9` | otelcol .deb sha256 (matches docs.monad.xyz) |
| `UFW_SSH_PORT` | (empty — auto-detect) | space-separated list of SSH ports to whitelist (override if sshd on non-22) |
| `ULIMIT_NOFILE` | `1048576` | per-user file descriptor limit |

Example:
```bash
sudo MONAD_PKG_VERSION=0.14.3 ./monad-validator-setup --network=testnet --non-interactive
```

## License

MIT — see [LICENSE](../LICENSE).
