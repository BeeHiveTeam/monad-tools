# monad-authudp-check

Verify your Monad node has Authenticated UDP active and won't be disconnected when the network drops legacy UDP support.

```bash
$ sudo ./monad-authudp-check

monad-authudp-check  —  validator-1 at 2026-05-07T16:13:46Z
  ✓ version            monad 0.14.3 — fully compliant (≥0.14.0, Auth UDP enforced)
  ✓ service            monad-bft.service active
  ✓ wireauth.logs      755 wireauth log entries in last 1000 lines
  ✓ wireauth.peers     248 unique peers seen in keepalives
  ✓ udp.8000           UDP port 8000 listening (consensus P2P)
  ✓ udp.8001           UDP port 8001 listening (Authenticated UDP)
  ✓ config.toml        all Auth UDP keys present in correct sections

✓ READY — Auth UDP active, node compliant
```

## Why

[Monad release 0.14.3 dropped support for non-authenticated UDP at the network level](https://docs.monad.xyz/node-ops/upgrade-instructions/authenticated-udp-checking). Nodes that haven't migrated are disconnected.

The official guidance is "running 0.14.x means you're set" — but it's worth verifying:

- Package version is at least `0.12.6` (capability) and ideally `≥0.14.0` (fully compliant)
- `monad-bft.service` is actually active
- `monad_wireauth` log entries are flowing — proof Auth UDP code path is running
- Both UDP **8000** (consensus P2P) AND **8001** (Auth UDP) are listening — they are different ports
- `node.toml` has the 4 required keys in the **correct TOML sections** (`[peer_discovery]` and `[network]`)
- Peer keepalives flowing — proof remote peers acknowledge our authenticated handshake

## Run

### One-liner

```bash
curl -fsSL https://raw.githubusercontent.com/BeeHiveTeam/monad-tools/main/authudp-check/monad-authudp-check | sudo bash
```

### Local

```bash
sudo ./monad-authudp-check               # text report
sudo ./monad-authudp-check --json        # machine-readable
sudo ./monad-authudp-check --quiet       # CI mode: print only on FAIL
sudo ./monad-authudp-check --post URL    # post result to a compliance tracker
```

### CI / cron

```cron
*/15 * * * * /opt/monad-tools/authudp-check/monad-authudp-check --quiet --json | logger -t authudp-check
```

## Two-tier version threshold

Per Foundation Discord (2026-05-06):
> Running 0.14.x or later? You're all set — Authenticated UDP is already enforced and active on your node. Running before 0.14.0? Suggestion to upgrade as soon as possible.

| Version | Status | Verdict |
|---|---|---|
| `≥0.14.0` | **PASS** | Fully compliant — 0.14.3 enforcement won't drop this node |
| `0.12.6 ≤ v < 0.14.0` | **WARN** | Auth UDP capable but pre-0.14.0 (Foundation says "suggestion to upgrade") |
| `<0.12.6` | **FAIL** | Upgrade required immediately |

Override the thresholds via env if needed:

```bash
sudo MONAD_AUTHUDP_VERSION_MIN=0.12.6 MONAD_AUTHUDP_VERSION_REC=0.14.0 ./monad-authudp-check
```

## Exit codes

| Code | Meaning |
|---|---|
| `0` | Auth UDP confirmed active (or monad not installed — script bows out gracefully) |
| `1` | Active but with warnings (e.g. upgrade available, low peer count) |
| `2` | FAIL — node will be disconnected by the network |

If `monad` is not installed at all, the script prints a single info line and exits `0` instead of producing a wall of misleading FAIL'и (previous behaviour).

## What it checks

| Check | What | Why |
|---|---|---|
| `version` | dpkg `monad` version against two-tier threshold | Auth UDP shipped in 0.12.6, enforced from 0.14.0 |
| `version.upgrade` | apt has newer | Foundation 48-hour upgrade SLA |
| `service` | monad-bft.service active | Without it nothing else matters |
| `wireauth.logs` | `monad_wireauth` entries in journal | Proof Auth UDP code path is running, not just compiled in |
| `wireauth.peers` | unique peer addresses in keepalives (IPv4 + IPv6) | Proof peers acknowledge our handshake |
| `udp.8000` | UDP port 8000 listening | Required for consensus P2P |
| `udp.8001` | UDP port 8001 listening | Required for Authenticated UDP (separate from 8000) |
| `config.toml` | All 4 Auth UDP keys present in correct TOML sections | `[peer_discovery]` → `self_auth_port`, `self_record_seq_num`, `self_name_record_sig`; `[network]` → `authenticated_bind_address_port`. Section-aware match — a key in the wrong section gives FAIL because monad won't pick it up. |

## --post compliance tracker (optional)

```bash
sudo ./monad-authudp-check --post https://monad-tech.com/api/authudp-report
```

Submits result JSON to a public tracker. The BeeHive [monad-tech.com](https://monad-tech.com) site aggregates these into a network-wide compliance dashboard:

> **Auth UDP compliance: 187/200 active validators ready** (as of 2026-05-07)
> Validators still on legacy UDP: 0xa1b2…, 0x3f4e…, …

The result JSON is non-sensitive — only contains hostname (override via `HOSTNAME=anonymous`), timestamp, version, and pass/fail status. No keys, no chain data.

## License

MIT — see [LICENSE](../LICENSE).
