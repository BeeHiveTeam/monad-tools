# Manual fixes

`monad-doctor` flags many issues that `monad-validator-setup` cannot fix
automatically — they require destructive operations, hardware changes, BIOS
access, or a reinstall. This page is the reference for those.

If a fix can be automated, it lives in `monad-validator-setup` (see the `Auto:`
line in the relevant doctor hint). This page is only for the rest.

> **Always back up validator keys before any destructive operation.**

---

## SMT / HyperThreading still active (`cpu.smt` FAIL)

**Doctor:** `cpu.smt FAIL — SMT/HyperThreading active`

### Preferred: BIOS

Most servers expose SMT (AMD) / HyperThreading (Intel) under
*Advanced → CPU Configuration*. Disable, save, reboot.

### Workaround: `nosmt` kernel parameter (Hetzner dedicated, OVH, etc.)

When BIOS is unreachable (typical for rented bare-metal):

```bash
sed -i 's/GRUB_CMDLINE_LINUX="\([^"]*\)"/GRUB_CMDLINE_LINUX="\1 nosmt"/' /etc/default/grub
update-grub
reboot
```

Note: some Hetzner installimage configs ignore `GRUB_CMDLINE_LINUX_DEFAULT` —
put `nosmt` in `GRUB_CMDLINE_LINUX` for reliability.

Verify after reboot:

```bash
cat /proc/cmdline | grep -o nosmt           # → nosmt
cat /sys/devices/system/cpu/smt/active      # → 0
nproc                                       # logical = physical
```

**Why we don't automate this:** modifying GRUB and rebooting is destructive of
the running validator session; this MUST be done at provisioning time, not on a
live node.

---

## OS on single disk (`disk.role.<dev>` WARN, SPOF)

**Doctor:** `disk.role.nvme0n1 WARN — OS on single disk (SPOF)`

This means `installimage` was run with `SWRAID 0`, or only one NVMe is present.
There is no in-place fix — software RAID1 must be created at install time.

### Fix on Hetzner

1. **Back up validator keys** (`/home/monad/.../keystore/`) and `node.toml`.
2. Hetzner Robot → **Reset** → **Activate rescue system**.
3. SSH into rescue → run `installimage`.
4. In the config editor:
   ```
   SWRAID 1
   SWRAIDLEVEL 1
   ```
5. Save (F10), wait for install, reboot.
6. Restore keys, re-run `monad-validator-setup`.

### Trade-off

| Choice | Usable | Resilience |
|---|---|---|
| `SWRAID 0` (single) | full disk | one drive failure → node down |
| `SWRAID 1 + RAID1` | half disk | one drive failure → node keeps running |
| `SWRAID 1 + RAID0` | full disk | one drive failure → node down (no redundancy) |

For mainnet validators, **RAID1 is the only safe choice**.

### When you can accept SPOF

- Testnet or short-lived experiments.
- A hot-standby validator exists with the same keys ready to take over
  (requires careful double-sign protection — out of scope here).

---

## TriedB device wrong LBA format (`disk.lba.<dev>` FAIL)

**Doctor:** `disk.lba.nvme1n1 FAIL — 4096-byte LBA — docs require 512`

NVMe drives can be formatted with 512B or 4096B sectors. Docs mandate 512B on
the triedb device. Switching is destructive — all data on the device is wiped.

```bash
# 1. Back up validator keys FIRST. Confirm /dev/triedb does NOT contain them.
# 2. Stop anything using the disk.
sudo systemctl stop monad-bft
sudo umount /dev/nvme1n1* 2>/dev/null || true

# 3. List available LBA formats — pick the index with LBADS=512 (usually lbaf=0).
sudo nvme id-ns /dev/nvme1n1 -H | grep -i "LBA Format"

# 4. Reformat. DESTROYS ALL DATA.
sudo nvme format /dev/nvme1n1 --lbaf=0 --force

# 5. Verify
lsblk -dn -o NAME,LOG-SEC /dev/nvme1n1   # → 512
```

After reformat, recreate the triedb partition + udev rule (or re-run
`monad-validator-setup` — `step_triedb_symlink` handles the udev rule).

**Why we don't automate this:** `nvme format` destroys everything on the drive;
running it from a setup script without explicit operator confirmation would be
catastrophic for an already-running node.

---

## TriedB on RAID (`disk.triedb-raid` WARN)

**Doctor:** `disk.triedb-raid WARN — triedb device is part of RAID`

TriedB is write-heavy. RAID1 doubles writes (each write hits both mirrors); RAID5/6
adds parity write amplification. The docs require a *dedicated* NVMe for triedb.

### Fix

1. Provision (or repurpose) a dedicated NVMe that is **not** in any RAID array.
2. Find its PARTUUID after partitioning:
   ```bash
   sudo blkid -s PARTUUID -o value /dev/<new-part>
   ```
3. Update `/etc/udev/rules.d/99-triedb.rules` with the new PARTUUID.
4. `udevadm control --reload && udevadm trigger`
5. Migrate triedb data (or let monad rebuild from genesis if acceptable).

---

## No unused NVMe for triedb (`disk.triedb-candidate` WARN)

Every disk is already in a RAID array, mounted, or used for swap. The
`validator-setup` will refuse to create `/dev/triedb` without a free device.

### Options

| Option | When to choose |
|---|---|
| Add a dedicated NVMe | cleanest; requires hardware access |
| Reinstall with OS on ONE drive | reasonable on 2-NVMe Hetzner: drop SWRAID, install OS on `/dev/nvme0n1`, leave `/dev/nvme1n1` for triedb |
| Repartition existing disk | only if OS partition has space to give up; tricky and risky |

---

## NTP significant drift (`ntp` FAIL with large offset)

**Doctor:** `ntp FAIL — chrony, |offset|=XYZ µs — significant drift`

Chrony is installed but cannot reach reliable sources.

### Diagnose

```bash
chronyc sources -v
chronyc tracking
journalctl -u chrony -n 100 --no-pager
```

Look for:
- All sources marked `^?` (unreachable) — outbound 123/udp blocked or DNS issue.
- Stratum 0 or 16 on the selected source — pool returned no good sources.
- Skew > 100 ppm — clock hardware issue, possibly virtualized despite docs.

### Fixes

- **Blocked outbound 123/udp:** open in firewall / ask provider; or use HTTPS
  source like `chrony.cloudflare.com` if your network allows :443.
- **Stale sources:** edit `/etc/chrony/chrony.conf`, replace `pool` lines with
  geographically close ones:
  ```
  pool 2.ubuntu.pool.ntp.org iburst maxsources 4
  server time.cloudflare.com iburst nts
  ```
  Then `sudo systemctl restart chrony`.
- **Bad clock hardware:** common on KVM/cloud — but the validator must be bare
  metal anyway, so this points at an underlying virtualization issue.

---

## CVE-2026-31431 — `algif_aead` LPE (`cve.2026-31431` WARN)

Blacklist the module so it cannot be loaded:

```bash
echo 'install algif_aead /bin/false' | sudo tee /etc/modprobe.d/disable-algif.conf
sudo rmmod algif_aead 2>/dev/null || true   # if currently loaded
```

No reboot required. Doctor will switch to PASS on next run.

---

## SSH lockout safety

Several doctor hints (`ssh.passwordauth`, `ssh.rootlogin`) recommend toggling
sshd settings. Doing this on a remote validator can lock you out permanently.

**Before every sshd edit:**

1. Confirm your public key is in `~/.ssh/authorized_keys`:
   ```bash
   cat ~/.ssh/authorized_keys
   ```
2. In a SECOND terminal already logged in via key, run the fix.
3. From a THIRD fresh shell, test the new login BEFORE closing the existing
   session.
4. Have provider console access (Hetzner vKVM / OVH IPMI) ready as fallback.

Never edit sshd_config from a session that lacks key-based auth backup.

---

## Reference: what `validator-setup` does automate

If you see this in doctor output you do NOT need to come here — re-run
`monad-validator-setup`:

| Check | Auto-fix in setup |
|---|---|
| `ulimit.limits-conf` | `step_ulimits` (writes `/etc/security/limits.d/99-monad-validator.conf`) |
| `ulimit.system` (fs.file-max) | **Manual** — BeeHive recommendation, not in docs.monad.xyz. `echo "fs.file-max = 2097152" \| sudo tee /etc/sysctl.d/99-monad-tuning.conf && sudo sysctl --system` |
| `sysctl.netbuf` (rmem/wmem_max) | **`monad` apt package** — drops `/etc/sysctl.d/90-monad-network-buffer.conf` itself. The script no longer writes a sysctl drop-in (overriding pkg's value caused monad-bft panics). |
| `sysctl.swappiness` | **Manual** — BeeHive recommendation, not in docs.monad.xyz. `echo "vm.swappiness = 1" \| sudo tee /etc/sysctl.d/99-monad-tuning.conf && sudo sysctl --system` |
| `ntp` (timesyncd → chrony) | `step_chrony` |
| `auto-upgrades` (Automatic-Reboot=false) | `step_cve_mitigation` |
| `cve.2026-31431` (algif_aead) | `step_cve_mitigation` (blacklists module + unloads if present) |
| `net.firewall` (UFW install + rules) | `step_firewall` |
| `iptables.udp-filter` | `step_iptables_ddos` |
| `deps.nvme-cli` | `step_dependencies` |
| `monad.cruft-timer` | `step_cruft_timer` |
| `monad.triedb-udev` | `step_triedb_symlink` |
| `disk.sched.*` (mq-deadline) | `step_io_scheduler` (udev-persisted) |
| `vdp.otel-collector` (enable) | `step_otelcol_enable` |
| `vdp.mf-push`, `vdp.secp-key-label`, `vdp.otel-version` | `step_vdp_otel_push` (auto-enabled on testnet, soft-fails on verify) |
