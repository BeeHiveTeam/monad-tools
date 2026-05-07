# monad-doctor

Pre-flight readiness check for Monad validator nodes. Single bash script, zero dependencies (except `fio` for the optional IOPS test, `ethtool` for bandwidth, `nvme-cli` for the LBA fix command), runs on any modern Ubuntu server.

**48 checks across 5 sections** — tells you in 30 seconds whether your server is ready to be a Monad validator, and exactly what to fix if not.

```
$ sudo ./monad-doctor --quick

  ╔══════════════════════════════════════════════╗
  ║       monad-doctor — pre-flight check        ║
  ║       BeeHive monad-tools                    ║
  ╚══════════════════════════════════════════════╝
  Hostname: validator-1
  Date:     2026-05-07T16:19:18Z

━━ HARDWARE ━━
  ✓ host.bare-metal                running on bare metal
  ✓ cpu.cores                      16 physical cores (AMD EPYC 4585PX)
  ✓ cpu.smt                        SMT/HyperThreading disabled (per docs)
  ✓ cpu.clock                      5.75 GHz max (≥4.5 GHz required)
  ✓ cpu.avx                        AVX-512 supported (faster BLS)
  ✓ ram.total                      125 GB total (106 GB available)
  ✓ disk.nvme                      4 NVMe drive(s), 5365 GB total
  ✓ disk.role.nvme0n1              nvme0n1 (894G): OS RAID — member of md3
  i disk.role.nvme2n1              nvme2n1 (1.7T): unused — no partitions
  ✓ disk.role.nvme1n1              nvme1n1 (1.7T): triedb — /dev/triedb → nvme1n1p1
  ✓ disk.role.nvme3n1              nvme3n1 (894G): OS RAID — member of md3
  ✓ disk.lba.nvme1n1               triedb device nvme1n1: 512-byte LBA (per docs)
  ✓ disk.sched.nvme1n1             triedb device on mq-deadline ✓
  ✓ disk.os                        746 GB free on / (need ≥500 GB)
  ✓ disk.triedb-size               /dev/triedb on /dev/nvme1n1 (1788 GB)

━━ OS ━━
  ✓ os.distro                      Ubuntu 24.04.4 LTS
  ✓ kernel.version                 6.8.0-110-generic
  ✓ ulimit.service                 monad-bft LimitNOFILE=1048576
  ✓ ntp                            chrony, |offset|=4µs

━━ NETWORK ━━
  ✓ net.public-ip                  198.51.100.42
  ✓ net.speed.enp10s0f0np0         25000 Mb/s (≥300 required for validator)
  ✓ net.port.8000/tcp              port 8000/tcp owned by monad-node
  ✓ net.port.8000/udp              port 8000/udp owned by monad-node
  ✓ net.port.8001/udp              port 8001/udp owned by monad-node
  ✓ net.firewall                   ufw active, all required port/proto pairs allowed

━━ SECURITY ━━
  ✓ ssh.passwordauth               PasswordAuthentication=no
  ✓ rpc.exposure                   no RPC ports (8080/8081) publicly exposed
  ✓ iptables.udp-filter            UDP length filter on :8000 active (per docs)
  ✓ cve.2026-31431                 algif_aead blacklisted (CVE-2026-31431 mitigated)
  ✓ deps.nvme-cli                  nvme-cli installed
  ✓ fail2ban                       fail2ban active

━━ MONAD ━━
  ✓ monad.user                     monad system user exists (1001:1001)
  ✓ monad.triedb                   /dev/triedb → /dev/nvme1n1p1
  ✓ monad.version                  installed=0.14.3 (latest)
  ✓ monad.upstream                 matches GitHub latest stable (v0.14.3)
  ✓ monad.authudp                  Auth UDP active (wireauth sessions in logs)
  ✓ monad.authudp-cfg              node.toml has all Auth UDP keys in correct sections

━━ Summary ━━
  PASS: 39   WARN: 0   FAIL: 0   INFO: 3

✓ Verdict: READY
```

## Why use this

- **Before deploying a new validator**: catch hardware/OS issues before staking. Wrong NVMe LBA format, SMT enabled, or a virtualized host can take hours to debug — this script catches them in 30 seconds.
- **Audit existing validator**: find drift — kernel parameters that got reverted, services that turned themselves on, version upgrades you missed.
- **VDP eligibility audit**: bare-metal check, public RPC exposure check, kernel version check, SMT check — every doc-mandated requirement enforced.
- **CI / monitoring**: `--json` output (handles newlines/control chars correctly) integrates into existing alerting.

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
sudo ./monad-doctor --quick               # skip slow fio IOPS test (default for live nodes)
sudo ./monad-doctor --json                # machine-readable output
sudo ./monad-doctor --section hardware    # one section only (validates name)
sudo ./monad-doctor --no-color            # plain text (CI)
```

`--section` validates against the allow-list (`hardware`, `os`, `network`, `security`, `monad`) — typos exit with code 64, never silent false PASS.

## What it checks (48 checks)

| Section | Checks |
|---|---|
| **HARDWARE** (16) | `host.bare-metal` (systemd-detect-virt) · CPU physical cores (excludes SMT threads) · **`cpu.smt`** (must be disabled in BIOS per docs) · CPU base clock ≥4.5 GHz · AVX-512 / AVX2 · RAM ≥32 GB · swap not in use · NVMe count + total · **disk topology** (`os-raid` / `os-single` / `triedb` / `swap-only` / `unused` / `other`) · **`disk.lba.<dev>`** (triedb must be 512-byte LBA per docs) · IO scheduler `mq-deadline` on triedb · disk space on / (≥500 GB) · triedb device size (≥1.7 TB) · IOPS via fio (skipped if monad-bft active to avoid stomping production IO) |
| **OS** (8) | Distro **Ubuntu 24.04+** (apt repo only ships `noble`) · **kernel ≥6.8.0-60** + reject buggy `6.8.0-{56..59}` range · `LimitNOFILE` on monad-bft.service · `fs.file-max` · `net.core.rmem_max/wmem_max` · `vm.swappiness` · NTP (chrony preferred) |
| **NETWORK** (8) | Public IP · NAT detect · **bandwidth ≥300 Mbit/s** (validator) / 100 Mbit/s (full-node) via ethtool · port 8000/tcp · 8000/udp · 8001/udp (proto-aware!) · ufw rules cover all required port/proto pairs |
| **SECURITY** (10) | SSH `PasswordAuthentication` · `PermitRootLogin` (softened to INFO when PasswordAuth=no) · unattended-upgrades **with `Automatic-Reboot=false`** (not full disable — security patches still applied) · fail2ban · **`rpc.exposure`** (UFW 8080/8081 publicly open = VDP violation) · **`iptables.udp-filter`** (`-m length 0:1400` per docs) · **`cve.2026-31431`** (algif_aead blacklist per Foundation 4/30) · `deps.nvme-cli` (utility for fix commands) |
| **MONAD** (6) | system user `monad` · `/dev/triedb` symlink · package state (incl. apt hold) · current vs available version · **`monad.upstream`** (GitHub releases/latest, independent of apt cache) · Auth UDP active (wireauth log entries) · 4 Auth UDP keys in `node.toml` in **correct TOML sections** (`[peer_discovery]` and `[network]`) |

## Exit codes

| Code | Meaning |
|---|---|
| `0` | All checks passed (or only INFO) |
| `1` | Warnings only — node will run, but suboptimal |
| `2` | One or more FAIL — do not deploy validator |
| `64` | Bad argument (e.g. unknown `--section`) |

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

| Variable | Default | Source |
|---|---|---|
| `UBUNTU_VERSION_MIN` | `24.04` | docs (apt repo ships only `noble`) |
| `KERNEL_MIN` / `KERNEL_MIN_PATCH` | `6.8.0` / `60` | docs (≥6.8.0-60) |
| `KERNEL_BUGGY_PATCHES` | `"56 57 58 59"` | docs (known hang bug) |
| `CPU_CORES_MIN` | `16` | docs |
| `CPU_GHZ_MIN` | `4.5` | docs |
| `RAM_GB_MIN` / `RAM_GB_REC` | `32` / `64` | docs (≥32 GB) |
| `DISK_GB_OS_MIN` | `500` | docs |
| `DISK_GB_TRIEDB_MIN` | `1700` | docs (2 TB nominal ≈ 1788 GiB on real drives) |
| `NVME_IOPS_MIN/REC` | `200000` / `500000` | community heuristic — not in docs |
| `ULIMIT_MIN` | `1000000` | community heuristic |
| `NET_PORTS_REQUIRED` | `"8000/tcp 8000/udp 8001/udp"` | docs |
| `NET_SPEED_VALIDATOR_MBPS` / `NET_SPEED_FULLNODE_MBPS` | `300` / `100` | docs |
| `MONAD_AUTHUDP_VERSION_MIN` | `0.12.6` | docs |
| `MONAD_NODE_TOML` | `/home/monad/monad-bft/config/node.toml` | docs |

## What it does NOT do

- **Does not modify the system.** Read-only. Use `monad-validator-setup` (sister tool) to apply fixes.
- **Does not test mainnet vs testnet** — same hardware/OS requirements apply.
- **Does not validate validator keys** — separate concern, operator's responsibility.

## License

MIT — see [LICENSE](../LICENSE).
