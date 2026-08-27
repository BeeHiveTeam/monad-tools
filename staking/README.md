# monad-delegate

Delegate MON to a Monad validator through [staking-sdk-cli](https://github.com/monad-developers/staking-sdk-cli),
with the checks the CLI does not do for you.

## Why

The CLI signs and sends. It does not verify that the key you typed belongs to the address
you meant, it does not estimate gas (a fixed 2,000,000 at 500 gwei, up to ~1 MON per
attempt, charged whether the call succeeds or reverts), and it does not tell you what the
calldata actually targets. Delegating to the wrong validator is not recoverable.

## What it checks before anything is sent

1. The private key derives to the address you are delegating from.
2. The balance covers the amount **and** a gas reserve — delegating the whole balance
   leaves nothing to pay with, and the transaction fails after taking the fee.
3. The calldata decodes back to `delegate(<the id you asked for>)`.
4. `eth_call` simulates successfully against the staking precompile.

Dry run by default. `--confirm` is the only way to broadcast, and only after all four pass.

## It identifies your validator itself

Run it on the host running your validator and you only supply the amount:

```bash
AMOUNT=100 ./monad-delegate               # dry run
AMOUNT=100 ./monad-delegate --confirm     # send
```

The network comes from the node's own metric labels. The validator id and the authorised
address come from the staking precompile, matched against the secp key on disk. This is
deliberate: a tool that ships a validator id as a default is one forgotten variable away
from sending a stranger's stake somewhere they did not choose.

If you pass `VAL_ID` and it disagrees with what the node reports, the script stops rather
than picking one. Off the node, pass `VAL_ID` and `FROM` yourself — nothing is guessed.

Both networks are supported; `NETWORK=mainnet` switches the endpoint, and the precompile
address is the same on both.

## Key handling

The key is read with a hidden prompt, never taken as an argument, and unset on exit.

During the send it is exported as `FUNDED_ADDRESS_PRIVATE_KEY`, because that is the only
input the staking CLI accepts. While the CLI runs, the key is in that process's
environment and readable by root through `/proc/<pid>/environ`. Run this on a host you
control, and prefer a key holding only what you intend to delegate.

## Requirements

A checkout of `staking-sdk-cli` with its virtualenv (`.venv`) and `config.toml`, this
script placed alongside them. `curl` and `python3` for the node lookups.

## Caveat worth knowing

`add-validator` in the upstream CLI sets commission to 0% regardless of what you pass.
Check your commission on chain after registering, and correct it if it matters to you.
This script only delegates and does not touch commission.
