# Deliverable Escrow

**One line:** escrow whose release condition is a paragraph of prose, resolved by
a two-round adjudication where the second round is adversarial.

`contracts/deliverable_escrow.py` · GenVM `py-genlayer:1jb45aa8ynh...`

## The problem it solves

Every escrow in production settles on a binary signal: a signature, a hash, an
oracle price. That is why freelance work, grant tranches, bounties and content
licensing still settle off-chain - the real condition is "is this good enough",
and no EVM contract can read a deliverable.

This contract makes "good enough" a first-class settlement value. Acceptance is
not boolean: it is **basis points**. `ACCEPT` pays 100%, `PARTIAL` pays 50%,
`REJECT` pays 0%, and the escrow splits the balance accordingly instead of
forcing an all-or-nothing flip. Those three numbers are constants of the
contract, never outputs of the reviewer.

## State

```
Escrow {
  client, provider, amount, criteria, deadline, status
  deliverable, verdict, payout_bps, reasoning, judged_at     # round 1
  escalated, client_evidence, final_verdict, final_bps        # round 2
}
```

Two ledgers: `escrows: TreeMap[u256, Escrow]` and `credits: TreeMap[Address, u256]`.
All payouts go through the ledger, so a payout can never revert the judgement
that produced it.

## Lifecycle

```
create_escrow(provider, criteria, deadline_seconds)  [payable]
        |
     FUNDED --refund_expired() after deadline--> REFUNDED (client made whole)
        |
     submit_deliverable(url)   provider, one URL, overwrite = revision
        |
     DELIVERED --judge()--> JUDGED
        |                     |
        |                     +-- client silent past window --> settle()  --> SETTLED (round 1 stands)
        |                     |
        |                     +-- escalate(counter_evidence) --> DISPUTED --resolve_dispute()--> SETTLED (round 2)
```

## Why two rounds, and why they are not the same code path

**Round 1 is asymmetric.** Only the provider has a document, so the question is
one-sided: "does this artifact satisfy the criteria". The only available guard
against a hallucinated approval is independent re-derivation, so every validator
re-fetches the deliverable and re-judges it.

**Round 2 is adversarial.** Now both parties have documents on the record and the
question changes to "whose reading of the criteria is more probative". The prompt
is explicitly told not to defer to the first decision and to treat a counter
document that does not address the criteria as worth nothing. A rejection that
survives a counter-argument is a much stronger signal than one produced in a
one-sided read - which is exactly why round 2 is the one that moves disputed
money.

## How consensus is used

Both rounds call `gl.vm.run_nondet_unsafe(leader_fn, validator_fn)`:

- leader fetches the document(s), fills a fixed prompt template, requests
  `response_format="json"`, and returns a canonicalised JSON string
  (`json.dumps(..., sort_keys=True)`);
- validator independently fetches, independently judges, and accepts only if
  `verdict` matches **exactly** and `payout_bps` matches **exactly**.

The payout in that comparison is not a number the model produced. `_coerce()`
discards any `payout_bps` the reviewer volunteers and looks the amount up in a
single module-level table: `ACCEPT=10000`, `PARTIAL=5000`, `REJECT=0`.
`judge()` and `resolve_dispute()` write to storage from the same table, so no
path exists from a model's arithmetic to `_split()`. Validators agreeing on a
verdict are necessarily agreeing on the amount that verdict pays.

### Why there is no tolerance band

The first submission compared payouts with a +/-1000 bps tolerance and only
replaced a model-supplied amount when it drifted more than 2500 bps from the
canonical value. That left a real hole: a leader returning `ACCEPT/7500` and a
validator returning `ACCEPT/8500` both survived the clamp, both passed the
tolerance check, and the escrow then settled on the leader's 7500 bps - money
the validators had never agreed to. Steward review rejected the submission on
exactly that point.

Two changes close it, either of which alone would have been sufficient:

1. **Derive, do not read.** The verdict is the only input to the amount; the
   prompt no longer asks the reviewer for a payout number at all.
2. **Exact agreement.** `validate()` compares `payout_bps` with `==` instead of
   a tolerance band, as a second line of defence against future changes.

`test_accept_cannot_release_two_different_amounts` replays the 7500/8500 pair
and asserts that the escrow quotes 100%; the same pair is replayed for round two.

## Prompt-injection posture

Both the criteria and the documents are untrusted (the provider picks the
deliverable URL, the client picks the counter URL). Documents are length-capped
(6000 chars), wrapped in per-party delimiters, and the prompt states that content
inside them is data, never instruction. Empty or unreachable documents cannot
satisfy anything - they fall to `REJECT`, not to a default approval.

## Why it is reusable

The chain `(criteria, deliverable) -> verdict -> payout_bps` is the whole primitive.
Integrators get freelance milestones, grant tranches, bug bounties, content
licensing and SLA credits by writing different criteria - no contract change, no
frontend required. The escalation round is boilerplate-free dispute handling that
most escrow implementations either omit or push to a human.

## Tests

`tests/test_escrow_logic.py` (39 tests) covers funding guards (self-escrow, dust
amount), provider-only submission, deadline expiry and `refund_expired`, round-1
accept / partial / reject settlement splits, escalation-window enforcement,
provider-only and client-only permissions, round-2 override of a wrong round-1
rejection, validator agreement/dissent, and the money-binding rule. The
regression group asserts that a model-supplied `payout_bps` is ignored for every
verdict, that neither prompt asks for an amount, and that the exact 7500/8500
pair from the Steward rejection settles at 100% in both rounds.

## Deploy

Constructor: `DeliverableEscrow(escalation_window: u256)`

```
escalation_window = 3600   # 1h on testnet; a day is sane for real work
```

Studio deploy payload: `{"args":[3600]}`. Passing `0` means "use the minimum"
(60s), so a bad unit guess cannot revert the deploy.

`withdraw()` needs a live network (ghost contract transfer); direct-mode tests
assert on `credits`.
