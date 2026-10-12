# TrancheGateCore

<p align="center">
  <strong>A grant paid in tranches that open only when validators agree the work was done.</strong>
</p>

<p align="center">
  <a href="https://github.com/valentinzubok/TrancheGateCore/actions/workflows/ci.yml"><img src="https://github.com/valentinzubok/TrancheGateCore/actions/workflows/ci.yml/badge.svg" alt="CI" /></a>
  <img src="https://img.shields.io/badge/GenLayer-Studio%20Dev%2061997-4ade80?style=flat-square" alt="Studio Dev" />
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-blue?style=flat-square" alt="MIT" /></a>
</p>

This repository holds the intelligent contract, its tests and its deploy record, and nothing
else. The console that uses it is a separate project: [TrancheGate](https://github.com/valentinzubok/TrancheGate).

---

## The trust problem

A grant paid up front is a gift: the funder has nothing left to say if the work stops. A grant
paid on the funder's approval is a leash: the grantee ships and then waits for someone to feel
like signing. The usual middle is a committee, which is the same problem with a calendar.

**TrancheGate makes "did this period's work get done?" a consensus question about the grantee's
own progress page, and makes the answer move the money.** The funder escrows the whole grant in
native GEN on day one and fixes the rules. After that neither side can change them.

```
open_grant(grant_id, grantee, title, progress_url, criterion,
           tranches, review_interval, misses_allowed)            PAYABLE
      the attached value is the whole grant, split into equal tranches. The ONE page that
      will carry the progress, the criterion for reading it and the schedule are fixed here.
      with value attached it never reverts: a grant it cannot open is refused in the return
      value and the amount is booked as a credit for the sender

review(grant_id)                 anyone may call it; it takes a grant id and nothing else
      refused until a full review_interval has elapsed since the last accepted review
      validators fetch THE COMMITTED PAGE, freeze it under eq_principle.strict_eq and agree
      on one boolean: does it show finished work that meets the criterion for this tranche?
        met          -> one tranche is paid to the grantee, misses reset
        not met      -> a miss
        unchanged    -> the page is the one that already earned a tranche: a miss,
                        and no model is asked
        unreachable  -> the first is only noted; a second in a row is a miss
      misses_allowed misses in a row -> the grant is HALTED

reclaim(grant_id)                the funder, only on a halted grant: the remainder returns
renounce(grant_id)               the grantee may hand the remainder back at any time
withdraw_credit()                take back what open_grant could not escrow for you

get_grant / get_schedule / get_balance / get_credit / get_stats / get_events    free to read
```

### Why it fails the way it does

A tranche is money and a halt is the end of someone's funding, so a pipeline problem must never
look like an answer in either direction: a malformed model answer, a model error or a consensus
failure **reverts the transaction**. Nothing is paid, no miss is counted, and the review can be
run again.

| Risk | What stops it |
|---|---|
| The funder stops paying for work that is being delivered | The funder has no cancel. `reclaim` works only on a halted grant, and only consecutive misses agreed by validators halt one. There is no owner or admin. |
| The grantee is paid for a page that says nothing new | The SHA-256 of the page that earned the last tranche is stored. The same page again is a miss, decided without asking a model. Whitespace edits do not change the hash. |
| Plans presented as progress | The prompt counts only work the page states as done for this tranche. "In progress", "coming soon" and a bare claim that the criterion is met are not evidence, and doubt resolves to false. |
| Draining the grant by calling `review` in a loop | `next_review_at` is stored on chain and moves to *this transaction's time + review_interval* after every accepted review. It is elapsed time, not a calendar window, so two calls either side of a clock boundary are one review. |
| The criterion or the page being swapped later | Both are fixed in `open_grant` and have no setter. `review` takes a grant id and nothing else. |
| An outage halting a healthy grant, or paying a dead one | An unreachable page never pays. One outage is only noted; consecutive outages count as misses, so a page that is gone for good still ends the grant. Every review uses up its interval. |
| `"true"`, `1` or `"yes"` counted as a verdict | `literal_bool()` accepts only JSON `true`/`false`. Anything else reverts and the grant is untouched. |
| The page steering the model | Title, criterion and page excerpts are fenced as untrusted data, inner fences are neutralized, and instruction-like phrasing is flagged to the model and stored on the grant. |
| A grant stranded by a failed call | Value attached to a reverting call is not returned on this network. `open_grant` therefore never raises once value is attached: ten different refusals are tested, and each credits the full amount to the sender. |
| A payout that "succeeds" and pays nobody | Wallets are paid with an external transfer through an EVM interface, after the books are updated. An internal message to a wallet succeeds and moves nothing. |
| Money leaving any other way | One payable method and one `emit_transfer` in the source. After every step of a 300-step random run the contract balance equals unpaid escrow plus credits. |
| Consensus quietly degrading | No fallback: if `prompt_comparative` cannot run, the transaction reverts. `principle` is passed positionally (it is positional-only in GenVM v0.3). |

## Live

| | |
|---|---|
| Network | GenLayer Studio Dev / Studio Next — chain `61997` |
| Contract | [`0x67aede124648dEb4749f0F96173F44df6F6d09D1`](https://explorer-studio-dev.genlayer.com/address/0x67aede124648dEb4749f0F96173F44df6F6d09D1) |
| Deploy record | [`STUDIO_DEV_DEPLOY.md`](STUDIO_DEV_DEPLOY.md) — every transaction of the demo, including the refused ones |

`scripts/verify_deployment.py` runs in CI and fails the build if the deployed bytes stop matching
[`contracts/TrancheGate.py`](contracts/TrancheGate.py).

The progress pages the on-chain demo read are kept in [`fixtures/`](fixtures/): two reports that
state released work, and one that states only plans.

## Tests

```bash
pip install -r requirements-dev.txt
python3 -m pytest -q      # 72 tests
```

[`tests/test_tranchegate.py`](tests/test_tranchegate.py), by what it proves:

- **Nobody can change the rule.** No owner, admin, setter, cancel or pause in the contract; the
  funder cannot reclaim an active grant; only the funder reclaims a halted one; only the grantee
  renounces.
- **It cannot be hurried.** No review before the first interval for four different accounts; a
  second review inside the interval reverts without fetching; the interval is elapsed time across
  a midnight boundary; a caller hammering every ten minutes cannot drain a six-tranche grant
  faster than six intervals.
- **The same page is not paid twice.** An unchanged page is a miss with no prompt sent;
  whitespace edits do not make it new; a met review resets the misses.
- **An outage is not a verdict.** First unreachable only noted, second is a miss, it uses its
  interval, and five outages pay nobody.
- **Nothing malformed becomes a verdict.** Thirteen malformed answers, a model error and a
  consensus failure each revert and leave the grant byte-identical; a failed review does not use
  up the interval.
- **It cannot be steered.** Fencing, planted fences, flagged injection phrasing, a long page
  digested within budget with its tail still read.
- **Money.** Ten refused openings never raise and credit everything; credits are per address and
  single-use; grants do not share money; the books are updated before the transfer; the balance
  equals escrow plus credits after every step of a random run.

## License

[MIT](LICENSE) © 2026 Valentyn Zubok
