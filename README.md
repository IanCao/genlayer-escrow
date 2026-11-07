# Deliverable Escrow

An Intelligent Contract for GenLayer that settles on **"good enough"** instead of
on a boolean.

Real escrows settle on a binary signal - a signature, a hash, a price - which is
why real work still settles off chain. The condition that actually matters is
"does this artifact satisfy what we agreed", and that is not binary. Here
acceptance is a **settlement value**: `ACCEPT / PARTIAL / REJECT` map to
`10000 / 5000 / 0` basis points, so partial work pays partially instead of all or
nothing.

```
client ──create_escrow(criteria, deadline)──▶ contract ──funds locked──▶ provider
                                                   │
provider ──submit(artifact_url)────────────────────┘
                                                   │
                              judge()  ── round one, one document
                                                   │
              ┌────────────────────────────────────┴───────────────────┐
      client settle()                                        client escalate(counter_evidence)
      funds move, done                                              │
                                                       resolve_dispute()  ── round two
                                                       both documents, adversarial
```

## Why two rounds

Round one is **asymmetric**: only the provider holds a document, so a lazier
prompt would let the leader's reading of it stand. The guard is therefore
independent re-derivation - every validator fetches the artifact itself, applies
the criteria itself, and the leader is accepted only if `verdict` matches and
`payout_bps` is within 10 points of the validator's own read.

Round two is **adversarial**: both sides now hold documents, so the risk flips
from laziness to deference. The round-two prompt must not treat round one as
precedent, and counter evidence that ignores the written criteria counts for
nothing - a document cannot win by being loud.

Both rounds end in `_coerce()`, which re-pins `payout_bps` to its own `verdict`.
Validator drift therefore cannot create money: the payout is a function of the
decision, not a value the model gets to choose independently of it.

## Deployment

| Item | Value |
|---|---|
| Network | Studionet (chain id 61999) |
| Address | `0x28Ee25d23d30f8E7C53eF54DBf0dF328813E3105` |
| Constructor args | `escalation_window=3600` |
| Explorer | https://explorer-studio.genlayer.com/address/0x28Ee25d23d30f8E7C53eF54DBf0dF328813E3105 |

## Verify it yourself

`scripts/verify_deployment.mjs` proves the running bytecode is this source and
that the constructor values actually took effect. Read-only: no private key, no
signing, no gas.

```bash
npm install genlayer-js
node scripts/verify_deployment.mjs
```

```
Studionet  chainId=61999  head=1794058442

========================================================================
DeliverableEscrow  0x28Ee25d23d30f8E7C53eF54DBf0dF328813E3105
  code    identical  sha256 bf1dec99b7b00bc6 vs bf1dec99b7b00bc6
  schema  identical  12 methods, ctor [escalation_window]
  exec    escrow_count() -> 0
  guard   escrow amount 0 wei is below the minimum of 10000000000000000 wei
  slot    escalation_window  3600                 slot Ny1Gw62p+J… byte 56
```

Five checks, because each one alone can be fooled:

- **code** - `getContractCode(address)` returns the source stored on chain,
  compared byte-for-byte with the repo file. Only line endings differ (Studio
  stores CRLF); lengths and SHA-256 match after normalising them.
- **schema** - the repo file is compiled through `gen_getContractSchemaForCode`
  and diffed against `gen_getContractSchema(address)`. A schema match alone would
  not prove the bodies match; a code match alone would not prove the constructor
  signature survived deployment.
- **exec** - a live view call returns, so the deployed contract actually runs
  rather than merely existing.
- **guard** - `create_escrow()` is payable, and its guard reverts below the
  floor with the amount in the message. A value-0 simulation is therefore a free
  read of live state.
- **storage** - the receipt carries `contract_state`, and `escalation_window`
  is located inside it. Storage integers are variable-length little-endian, so
  the check searches the byte stream for the minimal encoding instead of decoding
  32-byte words.

## Tests

```bash
pip install -r requirements.txt
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 python -m pytest tests -q
```

29 tests, about a third of a second, no network and no Docker. They cover the
state machine, permission edges, all three verdict paths, payout drift,
escalation timing, the pull ledger, and prompt interference (a document that
tries to dictate the verdict does not get to dictate the verdict).

The suite runs against `tools/genshim`, a minimal stand-in for GenVM, because the
official `genlayer-test` direct runner downloads the GenVM runtime from GitHub
releases on first use. What the suite deliberately does **not** claim to cover is
consensus-level behaviour, which needs a real multi-node run.

## Contract surface

| Method | Kind | Purpose |
|---|---|---|
| `create_escrow(provider, criteria, deadline_seconds)` | `write.payable` | Lock funds and pin the acceptance criteria |
| `submit_deliverable(escrow_id, url)` | `write` | Provider submits one artifact |
| `judge(escrow_id)` | `write` | Round-one adjudication by independent re-derivation |
| `escalate(escrow_id, counter_evidence)` | `write` | Client contests round one with counter evidence |
| `resolve_dispute(escrow_id)` | `write` | Round two, adversarial, both documents |
| `settle(escrow_id)` | `write` | Release funds per the adjudicated payout |
| `refund_expired(escrow_id)` | `write` | Provider reclaims after the deadline |
| `withdraw()` | `write` | Pull-ledger exit, so payouts cannot revert settlement |
| `get_escrow(id)` / `quote_payout(id)` | `view` | Read state and the currently payable amount |
| `get_credit(account)` / `escrow_count()` | `view` | Pull-ledger balance and total escrows |

Details and design rationale: [`docs/deliverable_escrow.md`](docs/deliverable_escrow.md).

## Related

The companion primitive, a bonded-assertion oracle with a challenge window, lives
at [IanCao/genlayer-primitives](https://github.com/IanCao/genlayer-primitives).
The two contracts share no runtime code; they are separate submissions and
separate deployments.
