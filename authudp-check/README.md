# monad-authudp-check

Verify your Monad node has Authenticated UDP active and won't be disconnected when the network drops legacy UDP support.

```bash
$ sudo ./monad-authudp-check

monad-authudp-check  —  validator-1.example at 2026-05-06T17:09:10Z
  ✓ version            monad 0.14.2 (Auth UDP available)
  ! version.upgrade    newer version available: 0.14.3 (current 0.14.2)
  ✓ service            monad-bft.service active
  ✓ wireauth.logs      40 wireauth log entries in last 1000 lines
  ✓ wireauth.peers     8 unique peers seen in keepalives
  ✓ udp.port           UDP port 8000 listening (raptorcast/wireauth)

! OK — Auth UDP active but with warnings
```

## Why

[Monad release 0.14.3 will drop support for non-authenticated UDP at the network level](https://docs.monad.xyz/node-ops/upgrade-instructions/authenticated-udp-checking). Nodes that have not migrated will be disconnected.

The official guidance is "running 0.14.x means you're set" — but it's worth verifying:

- The package version (≥0.14.0)
- That `monad_wireauth` log entries are actively being produced (proof Auth UDP is running, not just installed)
- That UDP port 8000 is actually listening
- That peer keepalives are flowing (proof peers acknowledge our authenticated handshake)

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

Add to operator's monitoring — fail loud if Auth UDP regresses:

```cron
*/15 * * * * /opt/monad-tools/authudp-check/monad-authudp-check --quiet --json | logger -t authudp-check
```

## Exit codes

| Code | Meaning |
|---|---|
| `0` | Auth UDP confirmed active |
| `1` | Active but with warnings (e.g. upgrade available, low peer count) |
| `2` | FAIL — node will be disconnected at next release |

## --post compliance tracker (optional)

```bash
sudo ./monad-authudp-check --post https://monad-tech.com/api/authudp-report
```

Submits result JSON to a public tracker. The BeeHive monad-tech site aggregates these into a network-wide compliance dashboard:

> **Auth UDP compliance: 187/200 active validators ready** (as of 2026-05-06)
> Validators still on legacy UDP: 0xa1b2…, 0x3f4e…, …

The compliance API is open to any operator; if you're maintaining a similar tracker, follow the same JSON schema and submit a PR to point this script at multiple endpoints.

The result JSON is non-sensitive — only contains hostname (which you can override via `HOSTNAME=anonymous`), timestamp, version, and pass/fail status. No keys, no chain data.

## What it checks

| Check | What | Why |
|---|---|---|
| `version` | `dpkg -W monad` ≥ 0.14.0 | Auth UDP only enforced from 0.14.0+ |
| `version.upgrade` | apt has newer | Stay current with releases |
| `service` | monad-bft.service active | Without it nothing else matters |
| `wireauth.logs` | `monad_wireauth` entries in journal | Proof Auth UDP code path is running |
| `wireauth.peers` | unique peer addresses in keepalives | Proof peers acknowledge handshake |
| `udp.port` | UDP 8000 listening | Required for raptorcast + auth-udp |

## License

MIT — see [LICENSE](../LICENSE).
