# TrancheGate — Studio Dev (chain 61997) deploy record

| | |
|---|---|
| **Network** | GenLayer Studio Dev / Studio Next — chain `61997` |
| **RPC** | `https://studio-dev.genlayer.com/api` |
| **Contract** | [`0x67aede124648dEb4749f0F96173F44df6F6d09D1`](https://explorer-studio-dev.genlayer.com/address/0x67aede124648dEb4749f0F96173F44df6F6d09D1) |
| **Deploy tx** | [`0x1e867e0d…7195d7`](https://explorer-studio-dev.genlayer.com/tx/0x1e867e0d78cdb5423dd98070073f49787d46f31ff97c514683cf81517f7195d7) from `0x42F132a0519Ee2702A97f1f1f3Fe0E8041A2d542`, which has no role in the contract |
| **Source** | [`contracts/TrancheGate.py`](contracts/TrancheGate.py) — runner `py-genlayer:5jycge4q8k23462jtb0b9fyey1s9qz928sz2nbrd9mg4sxqg2qng` |
| **Source sha256** | `7ece0a0df4f8f29049b5cf7466310d4e01a9911ed1115c17b629439fc3504d82` |

## Verify the source yourself

```bash
curl -s -X POST https://studio-dev.genlayer.com/api -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"gen_getContractCode","params":["0x67aede124648dEb4749f0F96173F44df6F6d09D1"]}' \
  | python3 -c "import sys,json,base64,hashlib; print(hashlib.sha256(base64.b64decode(json.load(sys.stdin)['result'])).hexdigest())"
shasum -a 256 contracts/TrancheGate.py
# both print 7ece0a0df4f8f29049b5cf7466310d4e01a9911ed1115c17b629439fc3504d82
```

`scripts/verify_deployment.py` does the same in CI on every push.

## The run

Three fresh accounts, each given 100 test GEN by the faucet:

| Role | Address |
|---|---|
| funder | `0x3575CAde58edE84b34CABa9e8e28630Cd9Ec0473` |
| grantee | `0x5fF4A229887818206c6DD4356cF22cC5A9Ac3Af4` |
| stranger (triggers every review) | `0x90dc7649975e8f16D2c340Bcb15693a3a44eFc97` |

The `tidepool` grant committed to **one URL**, a file on the `main` branch of the console
repository ([TrancheGate](https://github.com/valentinzubok/TrancheGate)): `https://raw.githubusercontent.com/valentinzubok/TrancheGate/main/web/public/fixtures/grant-a.html`.
Between reviews that file was changed by public commits, the way a grantee updates a report.
The three versions are kept side by side in the repository
([month 1](https://github.com/valentinzubok/TrancheGate/blob/321b4df4c3e1b0187e870b90ba3f414b2207ae88/web/public/fixtures/versions/a-month-1.html),
[month 2](https://github.com/valentinzubok/TrancheGate/blob/321b4df4c3e1b0187e870b90ba3f414b2207ae88/web/public/fixtures/versions/a-month-2.html),
[month 3, plans only](https://github.com/valentinzubok/TrancheGate/blob/321b4df4c3e1b0187e870b90ba3f414b2207ae88/web/public/fixtures/versions/a-month-3-plans.html)).

Criterion of `tidepool`: *"Each month the page must state indexer work that is finished and
released in that month. Plans and work in progress do not count."*

Every transaction is listed, including the six that were refused. Wallet balances are
`eth_getBalance`; they also move by a fraction of a GEN in fees.

| # | Call | Sender | Result | What happened | Tx |
|---|------|--------|--------|---------------|----|
| 1 | `open_grant(tidepool) value 3 GEN` | funder | SUCCESS | `tidepool`: 3 GEN in 3 tranches, reviews 420 s apart, 2 misses halt it. Page: `raw…/main/web/public/fixtures/grant-a.html` | [`0x7fac8621…6b1b41`](https://explorer-studio-dev.genlayer.com/tx/0x7fac8621798a0ddfd05e19d3a325b2050034629d8cf380fa144ee7f9e36b1b41) |
| 2 | `open_grant(lantern) value 2 GEN` | funder | SUCCESS | `lantern`: 2 GEN in 2 tranches, reviews 60 s apart. Page pinned at commit `143c456` | [`0x892562c0…210aec`](https://explorer-studio-dev.genlayer.com/tx/0x892562c03acaef95ca93d4fd74d9175e5fac540e37bbd74b64d59a5c1b210aec) |
| 3 | `open_grant(uneven) value 1 GEN` | **stranger** | SUCCESS | value 1 GEN + 1 wei for 2 tranches. **Refused in the return value** (does not divide evenly); the whole amount is booked as the stranger's credit. Contract balance = 5 escrow + 1.000…001 credit | [`0x69c66a7b…79ed46`](https://explorer-studio-dev.genlayer.com/tx/0x69c66a7bf718e24bbbae590f42f93ccb2a2091f18af27907cb214eaaf479ed46) |
| 4 | `review(tidepool)` | stranger | **ERROR** | **refused**: too early, the first interval has not elapsed. The page is not fetched | [`0x3674d23e…e815f6`](https://explorer-studio-dev.genlayer.com/tx/0x3674d23e93647f310e21bd488c2da501bb9b07db3becb21dbf650a8ca3e815f6) |
| 5 | `reclaim(tidepool)` | **funder** | **ERROR** | **refused**: only a halted grant can be reclaimed. The funder cannot cancel an active grant | [`0xd0515252…16572d`](https://explorer-studio-dev.genlayer.com/tx/0xd051525269fddd8f54d2ab25e76cf7c30967433cdd3ba1c3d55980b6e716572d) |
| 6 | `review(lantern)` | stranger | SUCCESS | FINALIZED. Validators: **met**. Tranche 1 of `lantern` paid: grantee wallet 100 → 101 | [`0xc67210ad…29d862`](https://explorer-studio-dev.genlayer.com/tx/0xc67210ad0143ddfd648e60826298c48862a8d4cb63bde7039366fc694129d862) |
| 7 | `renounce(lantern)` | **funder** | **ERROR** | **refused**: only the grantee may renounce | [`0x428bf40b…1fa45f`](https://explorer-studio-dev.genlayer.com/tx/0x428bf40bbfa787427d08adf3adb3cf5a6c5bb43e69a894135e6b0fc5031fa45f) |
| 8 | `renounce(lantern)` | grantee | SUCCESS | FINALIZED. The grantee hands the remaining 1 GEN back: funder wallet 94.9998 → 95.9997 | [`0x43eb118b…1d62ef`](https://explorer-studio-dev.genlayer.com/tx/0x43eb118bcda93cc329ea5d4d31f5dd1dd877b547beaf16e6058fe0cc511d62ef) |
| 9 | `withdraw_credit()` | stranger | SUCCESS | FINALIZED. The refused 1 GEN + 1 wei returns to the stranger; credits 0 | [`0x9ebec245…bcbd62`](https://explorer-studio-dev.genlayer.com/tx/0x9ebec2452571b82697011111acdc1e9eabd38bb184bd64ee2df69841e3bcbd62) |
| 10 | `review(tidepool)` | stranger | SUCCESS | FINALIZED. Page is the month 1 report (sha-256 `c17c32e3660d…`). Validators: **met**. Tranche 1 paid | [`0x2a7f5e2e…b61d71`](https://explorer-studio-dev.genlayer.com/tx/0x2a7f5e2e965af5049fe41d4aebe90a3ec5c28d746e217b9a6a75603b3cb61d71) |
| 11 | `review(tidepool)` | stranger | SUCCESS | FINALIZED. Page changed by commit [`acaefcb`](https://github.com/valentinzubok/TrancheGate/commit/acaefcb5f7326af02fad28931eeb81c987ddd508) to the month 2 report (`76b85d7d3e98…`). **met**. Tranche 2 paid | [`0x6693e612…9d0615`](https://explorer-studio-dev.genlayer.com/tx/0x6693e61286133f5f07ec9247c8d3ff60c95eb49c84b0e7a1e38fd14e4b9d0615) |
| 12 | `review(tidepool)` | stranger | SUCCESS | FINALIZED. Nobody touched the page. Same hash `76b85d7d3e98…` → **unchanged**: miss 1, no model asked, nothing paid | [`0x2a1ffa24…44b74c`](https://explorer-studio-dev.genlayer.com/tx/0x2a1ffa2493af28c2d96038925c6b483c5f1201439339aaf2a70d600ce344b74c) |
| 13 | `review(tidepool)` | stranger | SUCCESS | FINALIZED. Page changed by commit [`321b4df`](https://github.com/valentinzubok/TrancheGate/commit/321b4df4c3e1b0187e870b90ba3f414b2207ae88) to "in progress, coming soon" (`40fe2e49e25f…`). Validators: **not met**. Miss 2 → grant **halted** | [`0x9bfe2131…b86382`](https://explorer-studio-dev.genlayer.com/tx/0x9bfe2131d43dd72059639e5ea5407f2d918a3a46e924e645b1334cad42b86382) |
| 14 | `review(tidepool)` | stranger | **ERROR** | **refused**: the grant is not active | [`0x6e913793…c5bab5`](https://explorer-studio-dev.genlayer.com/tx/0x6e913793eff0926853e13f314c147c8e7d347e5f49cd6ba08305084dd0c5bab5) |
| 15 | `reclaim(tidepool)` | **stranger** | **ERROR** | **refused**: only the funder may reclaim | [`0x1c7a714a…cdeba7`](https://explorer-studio-dev.genlayer.com/tx/0x1c7a714aadd83c36a2c230714586e1443c6218ca3b317c9785e186184fcdeba7) |
| 16 | `reclaim(tidepool)` | funder | SUCCESS | FINALIZED. The unpaid tranche returns: funder wallet 95.9997 → 96.9996. Contract balance 0 | [`0x7b07f901…fa82d6`](https://explorer-studio-dev.genlayer.com/tx/0x7b07f901e8acc86d7a731306f1fbe180e5f76d38f23014d12862235109fa82d6) |

### What the four reviews of one page show

| Review | The page at that moment | Result | Money |
|---|---|---|---|
| 1 | month 1: "the block indexer was finished and released as v0.1.0" | met | tranche 1 → grantee |
| 2 | month 2: "the public query API was finished and released as v0.2.0" | met | tranche 2 → grantee |
| 3 | the same month 2 page, untouched | unchanged, decided in code | nothing; miss 1 |
| 4 | month 3: "IN PROGRESS. The dashboard is coming soon" | not met | nothing; miss 2, halted |
| — | | | tranche 3 → back to the funder by `reclaim` |

Review 3 is the reason the contract stores the hash of the last paid page: a report that already
earned a tranche cannot earn the next one. Review 4 is a new page that looks like a report and
states no finished work.

End state, read from chain:

```
get_stats     {"grants":2,"by_status":{"reclaimed":1,"renounced":1},"paid_out":"3000000000000000000","returned":"2000000000000000000"}
get_balance   {"balance":"0","credits":"0","escrowed":"0"}
funder        100 → 96.9996   (5 GEN escrowed, 2 GEN returned)
grantee       100 → 102.9999  (+3 GEN: two tranches of tidepool, one of lantern)
stranger      100 → 99.9990   (the refused 1 GEN came back; the rest is fees)
```

6.000000000000000001 GEN went in and the same amount came out.

## Notes on Studio Dev

- The header must be `# v0.3.0` followed by the pinned `Depends` line.
- Every transaction carries a fee deposit (`genlayer-js@2.0.0-rc.1`, `studioDevnet`, `estimateTransactionFees`).
- A payout reaches the wallet when the transaction is finalized, not when it is accepted.
- `raw.githubusercontent.com` caches a branch file for about five minutes, so the demo used a
  420-second review interval and waited a full interval after each commit. The contract's own
  minimum is 60 seconds.
