# monad-validator-setup

One-shot host preparation for a Monad validator. Tunes the kernel, configures the firewall, installs the `monad` package and (optionally) the BeeHive monitoring stack — all idempotent, all reversible.

```bash
curl -fsSL https://raw.githubusercontent.com/BeeHiveTeam/monad-tools/main/validator-setup/monad-validator-setup | sudo bash -s -- --with-monitoring
```

Replaces 30+ minutes of copy-pasting from docs with a single command. After it finishes, place your keys, start the services, and your validator is online.

## What it does

1. **Pre-flight** — runs [`monad-doctor`](../doctor/) and aborts on FAIL (override with `--skip-preflight`)
2. **`/etc/sysctl.d/99-monad-validator.conf`** — `rmem_max`, `wmem_max`, `netdev_max_backlog`, `file-max`, `swappiness=1`, `tcp_fastopen`
3. **`/etc/security/limits.d/99-monad-validator.conf`** — `nofile=1048576` for all users + root
4. **IO scheduler** — `mq-deadline` on the largest NVMe (or `--triedb-dev=nvmeXnY`), persisted via udev rule
5. **chrony** — replaces `systemd-timesyncd` if active (sub-millisecond NTP for accurate `vote_delay`)
6. **Monad apt package** — adds the official repository (signed-by) and installs `monad`
7. **UFW** — opens `22/tcp` (SSH), `8000/8001 tcp+udp` (Monad P2P + raptorcast)
8. **(optional)** Calls the [BeeHive monad-grafana](https://github.com/BeeHiveTeam/monad-grafana) installer when `--with-monitoring` is set

## What it does NOT do

- **Does not generate or import validator keys.** Keys are sensitive — you place them yourself in `/home/monad/monad-bft/config/`.
- **Does not start `monad-bft.service`.** Without keys this would just crash-loop. Operator starts services after key placement.
- **Does not modify any file without backup.** Every existing file is copied to `<file>.bak.<timestamp>` before edit.

## Run

```bash
# Interactive (recommended first time)
sudo ./monad-validator-setup

# Show what would change without doing it
sudo ./monad-validator-setup --dry-run

# Non-interactive (CI / automation)
sudo ./monad-validator-setup --non-interactive

# With Grafana monitoring stack
sudo ./monad-validator-setup --with-monitoring

# Pin a specific NVMe as triedb device
sudo ./monad-validator-setup --triedb-dev=nvme1n1

# Skip preflight (you already ran monad-doctor)
sudo ./monad-validator-setup --skip-preflight
```

## After install — next steps

The installer prints these explicitly. Summarized:

```bash
# 1. Place keys
sudo cp bls_priv_key secp_priv_key validator_id /home/monad/monad-bft/config/
sudo chmod 600 /home/monad/monad-bft/config/{bls,secp}_priv_key
sudo chown -R monad:monad /home/monad/monad-bft/config/

# 2. Configure node.toml — peer_discovery_servers, external_address

# 3. Start services
sudo systemctl enable --now monad-execution monad-bft monad-rpc

# 4. Verify
sudo /opt/monad-tools/doctor/monad-doctor
journalctl -u monad-bft -f
```

## Idempotency

Safe to re-run any number of times. Each step:
- Checks current state first (e.g. "is chrony already active?")
- Skips changes if state matches desired
- Backs up before modifying
- Files written are deterministic — re-running produces identical content

## Uninstall

```bash
sudo ./monad-validator-setup --uninstall
```

Removes the three config files written by the installer:
- `/etc/sysctl.d/99-monad-validator.conf`
- `/etc/security/limits.d/99-monad-validator.conf`
- `/etc/udev/rules.d/60-monad-triedb-scheduler.rules`

It does NOT remove:
- The `monad` apt package — `sudo apt remove monad` if you want
- chrony, ufw, monad-grafana — independent of this tool
- Your validator keys, configs, chain data — these are yours, your responsibility

## Logs and reports

- Per-run log: `/tmp/monad-validator-setup-YYYYMMDD-HHMMSS.log`
- Post-install JSON report: `/var/lib/monad-validator-setup/report-YYYYMMDD-HHMMSS.json`

The report captures hostname, kernel, OS, monad version, paths to written config files. Useful for support tickets and for verifying drift over time.

## Configurable values

Override via environment variable:

| Variable | Default | What it controls |
|---|---|---|
| `SYSCTL_RMEM` / `SYSCTL_WMEM` | `16777216` | UDP receive/send buffer max (raptorcast) |
| `SYSCTL_FILE_MAX` | `2097152` | system-wide file descriptor limit |
| `SYSCTL_SWAPPINESS` | `1` | discourage swapping |
| `ULIMIT_NOFILE` | `1048576` | per-user file descriptor limit |
| `MONAD_APT_REPO` | `https://apt.monad.xyz` | apt source URL |

Example:
```bash
sudo SYSCTL_FILE_MAX=4194304 ./monad-validator-setup --non-interactive
```

## License

MIT — see [LICENSE](../LICENSE).
