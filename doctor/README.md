# monad-doctor

Pre-flight readiness check for Monad validator nodes. Single bash script, zero dependencies (except `fio` for the optional IOPS test), runs on any Ubuntu/Debian server.

Tells you in 30 seconds whether your server is ready to be a Monad validator — and exactly what to fix if not.

```
$ sudo ./monad-doctor

  ╔══════════════════════════════════════════════╗
  ║       monad-doctor — pre-flight check        ║
  ║       BeeHive monad-tools                    ║
  ╚══════════════════════════════════════════════╝
  Hostname: validator-1
  Date:     2026-05-06T17:08:00Z

━━ HARDWARE ━━
  ! cpu.cores                      16 cores — meets min (16), recommend 32+ (AMD EPYC 4585PX)
  ✓ cpu.avx                        AVX-512 supported (faster BLS)
  ! ram.total                      125 GB — meets min (64), recommend 128+
  ✓ disk.nvme                      4 NVMe drive(s), 5365 GB total
  ✓ disk.triedb-sched              at least one NVMe on mq-deadline (triedb-ready)
  ✓ disk.iops                      612000 4k random read IOPS

━━ OS ━━
  ✓ os.distro                      Ubuntu 24.04.4 LTS
  ✓ kernel.version                 6.8.0-110-generic
  ✓ ulimit.service                 monad-bft LimitNOFILE=1048576
  ! sysctl.swappiness              vm.swappiness=60 — high
    → Set vm.swappiness=1 in /etc/sysctl.conf to avoid validator stalls
  ✓ ntp                            chrony, |offset|=1µs

━━ NETWORK ━━
  ✓ net.public-ip                  198.51.100.42
  ✓ net.firewall                   ufw active, ports 8000 8001 allowed

━━ SECURITY ━━
  ✓ ssh.passwordauth               PasswordAuthentication=no
  ! auto-upgrades                  unattended-upgrades enabled — may auto-restart node mid-epoch
    → Disable for validator: systemctl disable --now unattended-upgrades

━━ MONAD ━━
  ✓ monad.authudp                  Auth UDP active (wireauth sessions in logs)

━━ Summary ━━
  PASS: 19   WARN: 8   FAIL: 0   INFO: 5

! Verdict: READY WITH WARNINGS
```

## Why use this

- **Before deploying a new validator**: catch hardware/OS issues before staking. NVMe with the wrong IO scheduler can take hours to debug; this script catches it in 30 seconds.
- **Audit existing validator**: find drift — kernel parameters that got reverted, services that turned themselves on, version upgrades you missed.
- **CI / monitoring**: `--json` output integrates into existing alerting.

Other validator install guides tell you the requirements. This script verifies you actually meet them.

## Run

### One-liner

```bash
curl -fsSL https://raw.githubusercontent.com/BeeHiveTeam/monad-tools/main/doctor/monad-doctor | sudo bash
```

### Local

```bash
git clone https://github.com/BeeHiveTeam/monad-tools.git
cd monad-tools/doctor
sudo ./monad-doctor
```

### Flags

```bash
sudo ./monad-doctor                       # full check
sudo ./monad-doctor --quick               # skip slow fio IOPS test
sudo ./monad-doctor --json                # machine-readable output
sudo ./monad-doctor --section hardware    # one section only
sudo ./monad-doctor --no-color            # plain text (CI)
```

## What it checks

| Section | Checks |
|---|---|
| **Hardware** | CPU cores · AVX2/AVX-512 · RAM · swap usage · NVMe count + total · IO scheduler per device · triedb-suitable disk · free space on / · NVMe IOPS via fio |
| **OS** | Distro (Ubuntu 22.04+) · kernel ≥5.15 · `LimitNOFILE` on monad-bft.service · `fs.file-max` · `net.core.rmem_max/wmem_max` · `vm.swappiness` · NTP daemon (chrony preferred) |
| **Network** | Public IP detection · NAT detection · ports 8000/8001 free or owned by monad · firewall (UFW) state and required port rules |
| **Security** | SSH `PasswordAuthentication` · `PermitRootLogin` · unattended-upgrades (warns — auto-restarts mid-epoch are bad) · fail2ban |
| **Monad** | Package state (incl. apt-mark hold) · current vs available version · Auth UDP active (wireauth log entries) |

## Exit codes

| Code | Meaning |
|---|---|
| `0` | All checks passed (or only INFO) |
| `1` | Warnings only — node will run, but suboptimal |
| `2` | One or more FAIL — do not deploy validator |

Useful for CI:
```bash
if ! sudo ./monad-doctor --quick --json > /tmp/doctor.json; then
  jq '.checks[] | select(.status=="FAIL")' /tmp/doctor.json
  exit 1
fi
```

## Customizing thresholds

Override defaults via env:

```bash
sudo CPU_CORES_REC=24 RAM_GB_REC=192 NVME_IOPS_REC=800000 ./monad-doctor
```

Available variables: `CPU_CORES_MIN/REC`, `RAM_GB_MIN/REC`, `DISK_GB_MIN`, `NVME_IOPS_MIN/REC`, `ULIMIT_MIN`, `KERNEL_MIN`, `NET_PORTS_REQUIRED`.

## What it does NOT do

- **Does not modify the system.** Read-only checks. Use `monad-validator-setup` (sister tool in this repo) for installation/tuning.
- **Does not test mainnet vs testnet specifics** — same hardware/OS requirements apply.
- **Does not validate Monad validator keys** — separate concern.

## License

MIT — see [LICENSE](../LICENSE).
