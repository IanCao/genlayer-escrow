# Correction after Steward review

Submission: `DeliverableEscrow - escrow that settles on "good enough"`,
contributed 2026-11-07, rejected.

## Rejection reason

> This contract cannot be accepted because validator agreement does not bind the
> payout amount: ACCEPT outputs of 7,500 and 8,500 basis points can pass the
> tolerance check while releasing different amounts. Please derive the payout
> strictly from the verdict, or require exact validator agreement on the payout,
> before resubmitting.

The finding is correct. This record describes the root cause and the fix.

## Root cause

Three pieces of the v1 implementation composed into the defect:

1. The prompt asked the reviewer for `{"verdict", "payout_bps", "reasoning"}`,
   so a model supplied a number that would later move funds.
2. `_coerce()` only replaced that number when it was more than **2500 bps** from
   the canonical value for the verdict. `ACCEPT/7500` and `ACCEPT/8500` are
   2500 and 1500 bps from `10000`, so both survived as legitimate values.
3. `validate()` accepted the leader when the verdict matched and
   `abs(leader_bps - validator_bps) <= 1000`. The difference between 7500 and
   8500 is exactly 1000, so it passed.

`gl.vm.run_nondet_unsafe` returns the **leader's** result, so the escrow stored
7500 bps and paid 75%. Validators had agreed on `ACCEPT`; nobody had agreed on
75%. Both the acceptance round and the adversarial appeal round used the same
clamp and the same tolerance band, so both were affected.

The existing test `test_payout_is_bound_to_the_verdict` only covered a `REJECT`
with 4000 bps, which is far outside the clamp. It did not cover two *valid*
`ACCEPT` values drifting inside the permitted band, which is why the suite was
green.

## Fix

Both remedies suggested by the Steward were applied; either alone would have
closed the hole.

| # | Change | Effect |
|---|---|---|
| 1 | Added module-level `VERDICT_PAYOUT_BPS = {ACCEPT: 10000, PARTIAL: 5000, REJECT: 0}`. `_coerce()` now **discards** any model-supplied `payout_bps` and returns the table value. | The verdict is the only input to the amount. |
| 2 | `judge()` and `resolve_dispute()` write `VERDICT_PAYOUT_BPS[verdict]` to storage instead of reading the field out of the consensus JSON. | No path from model arithmetic to `_split()`. |
| 3 | Both `validate()` functions now require `payout_bps` to be **exactly equal**, replacing `BPS_TOLERANCE`. | Second line of defence; a future change that reintroduces a number cannot ratify a different one. |
| 4 | Both prompts no longer ask for `payout_bps`; they state that the contract sets the amount from the verdict and discards any number supplied. | Removes the ambiguity at its source. |

`BPS_TOLERANCE` and the `+/-2500` clamp were deleted.

## What did not change

The two-round structure, both prompt framings (asymmetric vs adversarial), the
untrusted-data delimiters, the 6000-character document cap, the state machine,
the `Escrow` storage schema, the pull ledger, `refund_expired`, the escalation
window, and the `_split()` arithmetic are unchanged. The only behavioural
difference is that an `ACCEPT` now always pays 100%, a `PARTIAL` always 50%, and
a `REJECT` always 0%, regardless of what a reviewer says about money.

## Tests

`PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 python -m pytest tests -q` -> **39 passed**
(previously 29).

New and changed coverage:

- `test_accept_cannot_release_two_different_amounts` replays the exact
  `ACCEPT/7500` leader and `ACCEPT/8500` validator pair and asserts the escrow
  stores `10000` and quotes the full amount.
- `test_appeal_payout_is_also_verdict_derived` replays the same pair in round
  two and asserts the provider is credited 100%.
- `test_payout_is_derived_strictly_from_verdict` (7 parameterised cases) asserts
  that both dict and raw-string model outputs canonicalise for every verdict,
  including amounts far outside the old clamp.
- `test_model_supplied_payout_is_ignored` replaces
  `test_small_payout_drift_is_tolerated`, which encoded the rejected behaviour.
- `test_prompt_never_asks_the_model_for_an_amount` asserts the round-one prompt
  no longer requests a payout number.

The suite runs against `tools/genshim`, an offline GenVM stand-in. It still does
not claim to cover multi-node consensus rotation; that requires a live network.

## Files changed

- `contracts/deliverable_escrow.py`
- `tests/test_escrow_logic.py`
- `README.md`, `docs/deliverable_escrow.md`
- `docs/CORRECTION.md` (this file)
