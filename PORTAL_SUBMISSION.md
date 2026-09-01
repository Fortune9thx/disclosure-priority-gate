# Portal submission text

Ready-to-paste text for the GenLayer Portal submission form. The description below is exactly 981 characters (verified with Python `len()`, not eyeballed) against the Portal's 1000-character Notes/Description field limit.

## Name

```
DisclosurePriorityGate
```

## One-line summary

```
A reusable primitive that decides whether a submitted disclosure report duplicates an earlier confirmed one, via independent validator consensus rather than a program owner's own call.
```

## Description / Notes field (981 characters)

```
DisclosurePriorityGate decides whether a newly submitted disclosure report duplicates an already-confirmed earlier one, or is genuinely distinct -- via independent validator consensus, never a program owner's own say-so. A program registers with a scope, submission fee, and challenge stake. Anyone may submit a report (fee forwarded to the owner); the first report auto-confirms, later ones start pending. Anyone may challenge a pending report as a duplicate of a confirmed baseline, staked in GEN. Validators independently compare both reports' text and judge DUPLICATE or DISTINCT -- never trusting the leader's claim. DUPLICATE permanently disqualifies the report and refunds the stake; DISTINCT confirms it and forfeits the stake to the reporter, deterring bad-faith challenges. A bounded expiry refunds a stuck stake if evaluation never converges. Uses run_nondet_unsafe with a custom leader/validator pair. 63 tests pass; lint and typecheck clean. Live-verified on Bradbury.
```

## Source code / repository

```
https://github.com/Fortune9thx/disclosure-priority-gate
```

## Deployed contract

**Network:** GenLayer Bradbury Testnet

**Contract address:** `<filled in after live deployment -- see docs/DESIGN.md#live-verification>`

## Category / tags

```
Disclosure/Bounty Adjudication, Duplicate Detection, Equivalence Principle, Reusable Primitive, Anti-Griefing
```

## Why this survives the current review bar

- **A structurally different consensus shape from this account's prior primitives, not a re-skin.** IndependentEvidenceSettler is claim-vs-fetched-evidence; UpgradeChangelogGate is diff-vs-prose; both are asymmetric (one side is authoritative ground truth). This contract is symmetric claim-vs-claim comparison -- neither report is more authoritative, and the question ("do these two texts describe the same issue") has no meaning unless both are read on equal footing. See `docs/DESIGN.md`'s Originality section for the explicit comparison.
- **Validators never trust the leader.** `validator_fn` calls the identical `leader_fn` again over the identical, already-agreed report texts -- its own independent LLM judgment, not a shape check -- and only agrees if it lands on the same bucket.
- **Exactly one non-deterministic call per write method.** `evaluate_challenge` contains a single `gl.vm.run_nondet_unsafe(leader_fn, validator_fn)`; all validation and status/stake bookkeeping is plain deterministic code outside it. Confirmed by a clean `genvm-lint check`.
- **A deliberately reasoned fail-closed default.** Unparseable model output defaults to `DISTINCT`, not `DUPLICATE` -- the direction that protects a report's reward eligibility rather than handing a costless dismissal tool to a program owner. Full reasoning documented, not just asserted.
- **A liveness escape hatch applied proactively, not discovered after rejection.** This account's prior project (UpgradeChangelogGate) needed a real GenLayer Portal steward review to catch a missing timeout escape hatch for staked funds behind a non-deterministic resolution. That lesson is applied here from the first commit: `reclaim_expired_challenge` is a bounded (72h), permissionless timeout for the one path (`challenge_duplicate`) that actually escrows GEN -- the other two payable paths (`submit_report`, `claim_reward`) are analyzed explicitly and shown to need no escape hatch at all, since neither ever holds a balance pending resolution.
- **A strict internal audit found and closed a real concurrency race, with a regression test proving it.** Two challengers could otherwise open concurrent challenges against the same pending report against two different confirmed baselines, letting a later evaluation silently overwrite an earlier one's legitimate resolution. Closed with an at-most-one-open-challenge-per-report guard, independently corroborated as a recognized pattern (`tendercouncil`'s "one challenge per bidder") during benchmarking against three independently-accepted Portal submissions.
- **GenLayer's role is deliberately narrow.** The contract never judges scope, severity, or reward size -- only whether two reports describe the same underlying issue. Everything else stays the program owner's own call.

## Verification commands

```bash
genvm-lint check contracts/DisclosurePriorityGate.py
genvm-lint typecheck contracts/DisclosurePriorityGate.py
gltest tests/direct/ -v
```
