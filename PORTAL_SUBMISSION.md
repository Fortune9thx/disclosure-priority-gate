# Portal submission text

Ready-to-paste text for the GenLayer Portal submission form. The description below is exactly 985 characters (verified with Python `len()`, not eyeballed) against the Portal's 1000-character Notes/Description field limit.

## Name

```
DisclosurePriorityGate
```

## One-line summary

```
A reusable primitive that decides whether a submitted disclosure report duplicates an earlier confirmed one, via independent validator consensus rather than a program owner's own call.
```

## Description / Notes field (985 characters)

```
DisclosurePriorityGate decides whether a submitted report duplicates an already-confirmed one, or is distinct -- via independent validator consensus, never a program owner's say-so. A program registers with a scope, fee, and challenge stake. Anyone may submit a report (fee forwarded to the owner); the first auto-confirms, later ones start pending. Anyone may challenge a pending report against a confirmed baseline, staked in GEN. Validators independently compare both reports and judge DUPLICATE or DISTINCT -- never trusting the leader. DUPLICATE permanently disqualifies the report; DISTINCT settles only that challenge (stake to the reporter) and leaves it pending, so surviving one baseline never confers immunity from another. Once its window closes with nothing open and no duplicate ruling, anyone may confirm it. Bounded expiries refund stuck stakes. Uses run_nondet_unsafe with a custom leader/validator pair. 74 tests pass; lint/typecheck clean. Live-verified on Bradbury.
```

## Source code / repository

```
https://github.com/Fortune9thx/disclosure-priority-gate
```

## Deployed contract

**Network:** GenLayer Bradbury Testnet

**Contract address (1.1.0, current):** [`0x6E4c3445bE1b2ae0EA4EC73FfDAB93Fc7a6dA2f4`](https://explorer-bradbury.genlayer.com/address/0x6E4c3445bE1b2ae0EA4EC73FfDAB93Fc7a6dA2f4)

Deploy tx `0xff062c79c252171c0a9ea4a873e1c37901e3d99f7964cda7dc5dfdd747a3cc45` -- `ACCEPTED`/`FINISHED_WITH_RETURN`, confirmed readable. Supersedes 1.0.0 (`0x6688dA9243b0095827d60E528f38e0933f65904a`, deploy tx `0xdc8c3d0b57219675dbb2804b2c6a7a78c32c07681c410328322e0fe368e3cec9`, `ACCEPTED`/`AGREE`/`FINISHED_WITH_RETURN`, confirmed readable), redeployed after a GenLayer Portal steward's review of this contract's first submission found a real challenge-lifecycle gap -- see "Steward finding" below.

A real, live regression of the exact steward-described scenario was run against 1.1.0: `register_program` → `submit_report` (x2, two genuinely distinct issues) → `challenge_duplicate` → `evaluate_challenge`, all real signed transactions. Real, unscripted consensus judged `DISTINCT` with genuine reasoning ("the root causes and attack vectors... are fundamentally different"), and reading the report back afterward confirmed `"status": "pending"`, not `"confirmed_original"` -- live proof the fix holds. `confirm_report` was also confirmed live to correctly reject before the challenge window elapses. The 1.0.0 `DUPLICATE`-path live record (a deliberately paraphrased report correctly judged `DUPLICATE`) still stands as proof of the underlying consensus mechanism, unchanged by the 1.1.0 fix. Full record: `docs/DESIGN.md`'s "Live verification" section.

## Category / tags

```
Disclosure/Bounty Adjudication, Duplicate Detection, Equivalence Principle, Reusable Primitive, Anti-Griefing
```

## Why this survives the current review bar

- **A structurally different consensus shape from this account's prior primitives, not a re-skin.** IndependentEvidenceSettler is claim-vs-fetched-evidence; UpgradeChangelogGate is diff-vs-prose; both are asymmetric (one side is authoritative ground truth). This contract is symmetric claim-vs-claim comparison -- neither report is more authoritative, and the question ("do these two texts describe the same issue") has no meaning unless both are read on equal footing. See `docs/DESIGN.md`'s Originality section for the explicit comparison.
- **Validators never trust the leader.** `validator_fn` calls the identical `leader_fn` again over the identical, already-agreed report texts -- its own independent LLM judgment, not a shape check -- and only agrees if it lands on the same bucket.
- **Exactly one non-deterministic call per write method.** `evaluate_challenge` contains a single `gl.vm.run_nondet_unsafe(leader_fn, validator_fn)`; all validation and status/stake bookkeeping is plain deterministic code outside it. Confirmed by a clean `genvm-lint check`.
- **A deliberately reasoned fail-closed default.** Unparseable model output defaults to `DISTINCT`, not `DUPLICATE` -- the direction that protects a report's reward eligibility rather than handing a costless dismissal tool to a program owner. Full reasoning documented, not just asserted.
- **Responded to a real GenLayer Portal steward finding, not just this account's own self-review.** A steward found that a `DISTINCT` verdict against one challenger-selected baseline was incorrectly granting a report permanent immunity from challenges against other confirmed baselines -- and that an unchallenged report had no path to confirmation at all. Fixed exactly to the steward's proposed shape: `DISTINCT` now only settles that one challenge and leaves the report `"pending"`; a new permissionless `confirm_report` method is the only way a non-first report reaches `"confirmed_original"`, gated on a fixed challenge window closing with no open challenge and no `DUPLICATE` ever recorded. Regression-tested by directly reproducing the exact scenario the steward described (survive `DISTINCT` against baseline A, then correctly get ruled `DUPLICATE` against baseline B) and proving it now resolves correctly. See `docs/DESIGN.md`'s "Steward review" section.
- **A liveness escape hatch applied proactively, not discovered after rejection.** This account's prior project (UpgradeChangelogGate) needed a real GenLayer Portal steward review to catch a missing timeout escape hatch for staked funds behind a non-deterministic resolution. That lesson is applied here from the first commit: `reclaim_expired_challenge` is a bounded (72h), permissionless timeout for the one path (`challenge_duplicate`) that actually escrows GEN -- the other two payable paths (`submit_report`, `claim_reward`) are analyzed explicitly and shown to need no escape hatch at all, since neither ever holds a balance pending resolution.
- **A strict internal audit found and closed a real concurrency race, with a regression test proving it.** Two challengers could otherwise open concurrent challenges against the same pending report against two different confirmed baselines, letting a later evaluation silently overwrite an earlier one's legitimate resolution. Closed with an at-most-one-open-challenge-per-report guard, independently corroborated as a recognized pattern (`tendercouncil`'s "one challenge per bidder") during benchmarking against three independently-accepted Portal submissions.
- **GenLayer's role is deliberately narrow.** The contract never judges scope, severity, or reward size -- only whether two reports describe the same underlying issue. Everything else stays the program owner's own call.

## Verification commands

```bash
genvm-lint check contracts/DisclosurePriorityGate.py
genvm-lint typecheck contracts/DisclosurePriorityGate.py
gltest tests/direct/ -v
```
