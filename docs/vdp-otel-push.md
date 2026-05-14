# VDP OTel metrics push

In May 2026 Monad Foundation (MF) emailed all VDP validators requiring metrics push to `otel-external.monadinfra.com:443`. 15% of mainnet and 30% of testnet operators were not compliant when the email went out.

This document describes how to set it up, both for the **default Monad apt setup** (plain `otelcol`) and **custom setups** (e.g. you replaced `otelcol` with `otelcol-contrib` for logs-to-Loki).

## Quick: default setup

Monad's apt package installs `otelcol` (plain). For this case the MF script runs as-is:

```bash
curl -fsSL -o /tmp/otel-validator-setup.sh https://bucket.monadinfra.com/tmp/otel-validator-setup.sh
echo "92cca0add5456db13c792b6e1b484999724b348ebd41a40a73d7f42086f26a23  /tmp/otel-validator-setup.sh" \
  | sha256sum -c \
  && sudo bash /tmp/otel-validator-setup.sh
```

The SHA-256 above was current as of 2026-05-14 — verify against the latest MF email.

`monad-validator-setup --with-vdp-otel` automates this step (downloads, verifies sha256, runs).

## Verification

After setup, `monad-doctor` verifies VDP compliance via 4 checks in the `monad` section:

```
monad     vdp.otel-collector       otelcol.service active
monad     vdp.otel-version         otelcol v0.139.0 (>= 0.139.0 required)
monad     vdp.secp-key-label       metrics tagged with secp_key=02ee6af4…000e (VDP-compliant)
monad     vdp.mf-push              OTel collector has 2 active TLS connection(s) to :443
```

All four should PASS. If `vdp.secp-key-label` is FAIL, push is not configured — re-run the MF script.

## Manual setup for custom OTel collector

The MF setup script:
1. Writes config to `/etc/otelcol/config.yaml`
2. Restarts `otelcol.service`

If your active OTel collector is `otelcol-contrib.service` (reads `/etc/otelcol-contrib/config.yaml`), the script does not apply correctly. You must merge the MF additions into your existing config.

### What MF adds

```yaml
processors:
  resource/secp:                # NEW
    attributes:
      - key: secp_key
        value: "<YOUR_SECP_PUBKEY>"
        action: upsert

exporters:
  otlp/monadinfra:               # NEW
    endpoint: https://otel-external.monadinfra.com:443

service:
  pipelines:
    metrics/monad:               # modify your existing metrics pipeline:
      receivers: [otlp]
      processors: [memory_limiter, resource/secp, batch]    # add resource/secp
      exporters: [prometheus, otlp/monadinfra]              # add otlp/monadinfra
```

Recommended pattern: **split pipelines** so only `monad_*` metrics (not hostmetrics or your custom telemetry) go to MF. See [otelcol-contrib wiki](https://github.com/BeeHiveTeam/monad-tools/blob/main/docs/vdp-otel-push.md#split-pipelines).

### Extracting SECP pubkey

The MF script extracts it via:

```bash
sudo -u monad bash -c '
  source /home/monad/.env
  /usr/local/bin/monad-keystore recover \
    --password "$KEYSTORE_PASSWORD" \
    --keystore-path /home/monad/monad-bft/config/id-secp \
    --key-type secp
' | awk '/Secp public key/ {print $NF}'
```

You need `KEYSTORE_PASSWORD` in `/home/monad/.env` (set when generating keys).

### Split pipelines (for custom contrib setups)

Replace your single `metrics` pipeline with two:

```yaml
service:
  pipelines:
    metrics/monad:                   # only monad_* → MF + local Prometheus
      receivers: [otlp]
      processors: [memory_limiter, resource/secp, batch]
      exporters: [prometheus, otlp/monadinfra]
    metrics/host:                    # hostmetrics → local Prometheus only
      receivers: [hostmetrics]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
    # ... your logs pipelines stay as-is
```

This sends MF exactly what the official script would: `monad_*` with `secp_key`, nothing else.

### Verifying without running the MF script

```bash
# Right service active?
systemctl is-active otelcol-contrib  # or otelcol

# secp_key label present?
curl -s localhost:8889/metrics | grep -c "secp_key=" | xargs echo "secp_key labels:"
# (should equal the number of monad_* metrics)

# TLS push to MF active?
sudo ss -tnp | grep otelcol | grep ":443" | wc -l
# (≥1 = pushing)
```

Or just run `monad-doctor` — the 4 `vdp.*` checks confirm everything.

## Bumping MF sha256

If MF rotates the setup script, the `monad-validator-setup --with-vdp-otel` flag will fail at sha256 check. To unblock:

1. Fetch new sha256 from MF email/Discord
2. Either:
   - Run with override: `VDP_OTEL_SETUP_SHA256=<new> sudo ./monad-validator-setup --with-vdp-otel`
   - Or open a PR updating `VDP_OTEL_SETUP_SHA256` default in `validator-setup/monad-validator-setup`
