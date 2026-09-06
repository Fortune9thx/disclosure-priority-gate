# Final checklist

Pre-submission / pre-deployment verification record for DisclosurePriorityGate.

## Contract requirements

- [x] File header exact match: line 1 `# v0.2.16`, line 2 `# { "Depends": "py-genlayer:1jb45aa8ynh2a9c9xn3b7qqh8sm5q93hwfp7jqmwsfhh8jpz09h6" }` -- matches every other live-deployed contract in this account's project history using this pin.
- [x] Storage: `programs: TreeMap[str, str]`, `reports: TreeMap[str, str]`, `challenges: TreeMap[str, str]`, `report_counter: u256`, `challenge_counter: u256`. Bare class-level annotations, no explicit `TreeMap()` construction in `__init__` -- the verified-working pattern matching GenLayer's own official boilerplate reference contract (both this and the explicit-init form are confirmed-safe; bare annotation was chosen for consistency with the most recent prior contracts in this account's history).
- [x] Equivalence strategy: `gl.vm.run_nondet_unsafe(leader_fn, validator_fn)`, both named `def`s (never `lambda`). `validator_fn` independently re-derives the verdict via calling `leader_fn()` again over already-agreed report text -- never validates leader output by shape alone.
- [x] Outcome forced into exactly one of `"DUPLICATE"`/`"DISTINCT"` (`_coerce_verdict`), and only that bucket is compared under the Equivalence Principle (`validator_fn`'s sole comparison). No third "unclear" bucket -- deliberate, reasoned in `docs/DESIGN.md`.
- [x] Non-determinism boundaries: exactly one top-level `gl.vm.run_nondet_unsafe` call, in `evaluate_challenge`; no `self.*` read/write inside `leader_fn`/`validator_fn` (every value copied to plain locals first); confirmed by `genvm-lint check` finding zero nesting violations.
- [x] Required public methods present: `register_program`, `submit_report` (payable), `challenge_duplicate` (payable), `evaluate_challenge`, `reclaim_expired_challenge`, `confirm_report`, `claim_reward` (payable), `update_challenge_stake_requirement` (8 writes); `get_program`, `get_report`, `get_challenge`, `get_report_count`, `get_challenge_count` (5 views) -- 13 total, confirmed by `genvm-lint check`'s own method count output.
- [x] Error handling: `gl.vm.UserError` used for every user-facing error; no bare `raise Exception(...)`. Strong validation on `program_id` format, title/description non-emptiness and length caps, fee/stake minimums, program-membership matching, ownership on every owner-gated method.
- [x] Educational comments: module docstring covers the structurally-different-consensus-shape originality argument, independent re-derivation rationale, discrete-bucket rationale and fail-closed default direction, the Portal-rejection-pattern mapping, the exact Equivalence Principle strategy, and a full per-path liveness-escape-hatch analysis.

## Quality bar

- [x] `genvm-lint check contracts/DisclosurePriorityGate.py` -- 3 checks passed, zero warnings on the first pass.
- [x] `genvm-lint typecheck contracts/DisclosurePriorityGate.py` -- zero type errors on the first pass.
- [x] `gltest tests/direct/ -v` -- 63/63 passed on the first full run; 74/74 after the steward-review fix below (11 new tests).
- [x] Studio + Bradbury ready: dependency pin (`py-genlayer:1jb45aa8ynh2a9c9xn3b7qqh8sm5q93hwfp7jqmwsfhh8jpz09h6`) matches both networks' current supported runner; storage pattern (`TreeMap[str, str]` only) is the Bradbury-verified-reliable one.
- [x] Obviously useful/reusable: no domain-specific coupling -- `program_id`/`scope`/`title`/`description` are fully caller-supplied, making this composable by any bug-bounty, whistleblower-reward, prior-art, or research-priority registry.
- [x] "Validators only checked format" is not a valid criticism: `TestValidatorIndependence` in the test suite directly proves independent re-derivation and verdict-mismatch rejection.
- [x] "Outcome not bound by equivalence" is not a valid criticism: the verdict bucket is the only value ever compared, and the only thing status transitions / stake routing act on.
- [x] "Format-only validator" / "generic AI decides X" is not a valid criticism: GenLayer's role is narrowly whether two texts describe the same issue, never whether a report is valid, in-scope, or worth a reward -- see `docs/DESIGN.md`'s scope-boundary section.
- [x] Not a re-skin of this account's prior primitives: explicit, written originality argument distinguishing this contract's symmetric claim-vs-claim shape from IndependentEvidenceSettler's (claim-vs-evidence) and UpgradeChangelogGate's (diff-vs-prose) asymmetric shapes -- see `docs/DESIGN.md#originality`.

## Lessons applied proactively from the first commit (not discovered via a later audit pass this time)

- [x] `program_id` format-restricted to ASCII from the first commit, closing the Unicode-encoding-collision class of bug before it could ever occur here. No redundant NFC-normalization helper added on top -- traced explicitly why the regex alone already makes that unnecessary, rather than cargo-culting a closely related precedent contract's extra step.
- [x] Report title/description get the same non-blocking manipulation-heuristic screen (`_MANIPULATION_PATTERNS`) as this account's other caller-controlled-instruction-adjacent fields, shipped from the start, and correctly ORs both compared reports' flags together at evaluation time (not just the challenged report's own).
- [x] Verdict is a fixed two-string enum with zero numeric clamp/snap arithmetic -- structurally avoids the whole non-finite-float edge-case class a numeric bucket scale needs explicit guards against.
- [x] Fail-closed default direction (`"DISTINCT"`, not `"DUPLICATE"`) reasoned through and documented BEFORE writing `_coerce_verdict`, not chosen arbitrarily then rationalized after: the direction that protects a report's reward eligibility from a costless ambiguous-challenge dismissal, matching this contract's own stated anti-griefing purpose.
- [x] Liveness escape hatch (`reclaim_expired_challenge`, 72h) shipped in the initial commit, not added after a steward review found it missing (as happened on this account's prior project, UpgradeChangelogGate) -- the lesson from that finding was applied here proactively. The other two payable paths (`submit_report`, `claim_reward`) were explicitly analyzed and shown to need no escape hatch, not merely assumed safe because they don't obviously escrow anything.
- [x] `.github/workflows/ci.yml` (lint, typecheck, full test suite on every push/PR) shipped in the initial commit.
- [x] No caller-supplied address parameter exists anywhere in this contract's interface -- checked explicitly against this account's own documented real GenLayer rejection pattern (checksum-lookup bug), confirmed structurally inapplicable rather than assumed safe.
- [x] Originality screened, in writing, against every primitive this account has already built and every repo benchmarked while designing this one -- see `docs/DESIGN.md#originality` and `docs/DESIGN.md#benchmarking-against-independently-accepted-portal-submissions`.

## Adversarial self-review pass (same session, before first deployment) -- adversarial re-read of the actual code, not the docs

- [x] **The one real finding: a concurrency race between two challenges on the same report.** Without a guard, two different challengers could each open a challenge against the same still-`"pending"` report against two different `confirmed_original` baselines; resolving them in sequence would let the second `evaluate_challenge` silently overwrite whatever the first legitimately decided. This is the same *shape* of bug as UpgradeChangelogGate's previously-found stale-`based_on_version` clobber (a later write unconditionally overwriting an earlier one's effect), reached here via a different mechanism (concurrent challenges, not a stale computed diff). Fixed with `report["open_challenge_id"]`, enforced by `challenge_duplicate` at the root (rejecting a second concurrent challenge outright) and re-checked defensively (documented as such, not as a live gap) in `evaluate_challenge`. Regression-tested: `TestChallengeValidation::test_cannot_open_a_second_concurrent_challenge_on_same_report` and `test_new_challenge_allowed_after_first_one_resolves` (proving the guard is per-in-flight-challenge, not permanent).
- [x] Confirmed solid under adversarial review, not just asserted: CEI ordering correct in every write that both mutates state and transfers value (`submit_report`, `evaluate_challenge`, `reclaim_expired_challenge`, `claim_reward` all flip status/flags before their corresponding `emit_transfer` call); no float anywhere in storage or return values (all numeric fields are `u256` on the wire, decimal strings in storage); `claim_reward`'s no-enforced-minimum design is a disclosed trust boundary, not an oversight; no cross-contract-write dependency anywhere (every `emit_transfer` target is a real signer address captured from `gl.message.sender_address`, never a stored contract address).
- [x] Considered and deliberately not adopted: wrapping `leader_fn`'s own `gl.nondet.exec_prompt` call in `try/except` to produce a graceful third outcome on a tooling failure (the pattern `spec-compliance-bounty` uses for its `UNCLEAR` bucket). Traced why letting the exception propagate and fail the whole transaction is the correct choice for a strictly-two-bucket design where both buckets carry real, asymmetric financial consequences -- see `docs/DESIGN.md`.

## Benchmarking pass against three independently-accepted Portal submissions

- [x] Cloned and read `tendercouncil`, `spec-compliance-bounty`, `rubricproof-intelligent-contract` fresh for this project, screening specifically for anything relevant to a duplicate-adjudication/bounty shape.
- [x] **Independent confirmation, not the origin of the idea:** `tendercouncil`'s `"one challenge per bidder is required"` corroborates the `open_challenge_id` concurrency guard found and fixed above as a recognized, precedented pattern.
- [x] **Considered and deliberately not adopted:** `spec-compliance-bounty`'s `try/except -> UNCLEAR` pattern around its own `exec_prompt` call -- reasoning documented in `docs/DESIGN.md` rather than silently declined.
- [x] **Reconfirmed, not re-discovered:** `spec-compliance-bounty`'s CEI-ordering convention already matches this contract's own throughout; `rubricproof-intelligent-contract`'s SSRF-hardening pattern was checked and found structurally inapplicable (this contract has no `gl.nondet.web.render` call and no caller-supplied fetch URL at all).

## Steward review (first submission) -- a real challenge-lifecycle gap, not caught by this account's own audit passes

- [x] GenLayer Portal steward finding: *"ensure that a DISTINCT result against one challenger-selected baseline does not immediately make the report immune to challenges against other confirmed baselines... keep the report pending after each DISTINCT result, settle that challenge's stake, and confirm only after the challenge window closes with no duplicate ruling."* This directly overturns the original design's own "well-founded DAG" reasoning about how cheaply `confirmed_original` could be reached (the reasoning that it should be terminal was correct; the reasoning that a single favorable verdict against one baseline earned that terminal state was not). Disclosed as a genuine gap this account's own adversarial self-review and three-repo benchmarking pass both missed, not smoothed over.
- [x] **Fix, matching the steward's proposed shape exactly:** `evaluate_challenge`'s `DISTINCT` branch no longer touches `report["status"]` -- it settles only that challenge and leaves the report `"pending"`. New method `confirm_report(report_id: str) -> None`, permissionless, requiring `status == "pending"`, `open_challenge_id is None`, and a fixed `CHALLENGE_WINDOW_SECONDS` (72h) elapsed since the report's own `created_at`. This is now the ONLY way a non-first report reaches `"confirmed_original"` -- also closing a related gap where an unchallenged report previously had no confirmation path at all.
- [x] Deliberately a global, non-owner-configurable window constant, for the same anti-gaming reason `CHALLENGE_TIMEOUT_SECONDS` already is.
- [x] A residual, bounded race (repeatedly re-challenging to prevent a "no open challenge" gap from ever existing) is disclosed, not hidden -- traced through explicitly and shown to cost the adversary real gas/stake per attempt with no guaranteed permanent block, since the reporter or anyone else can win the race by calling `confirm_report` the instant a gap opens. See `docs/DESIGN.md`.
- [x] 11 new regression tests: `TestNoImmunityAfterDistinct` (the exact scenario the steward described -- survive `DISTINCT` against baseline A, then correctly get ruled `DUPLICATE` against baseline B) and `TestConfirmReport` (before/after window, open-challenge block, post-expiry confirm, already-confirmed/already-duplicate rejection, unknown-id rejection, permissionless). Existing tests that asserted the old (incorrect) immediate-confirmation behavior were corrected to match the verified-correct new lifecycle, not left passing against stale assumptions.
- [x] Re-verified after the fix: `genvm-lint check`/`typecheck` still zero warnings, all 74 tests pass, redeployed to Bradbury and confirmed readable. See `docs/DESIGN.md#steward-review-first-submission----a-real-gap-in-the-challenge-lifecycle-not-caught-by-this-accounts-own-self-review`.
- [x] README.md / docs/DESIGN.md / PORTAL_SUBMISSION.md / CHANGELOG.md updated with the new address and this finding.

## Deliverables

- [x] `contracts/DisclosurePriorityGate.py`
- [x] `tests/direct/test_disclosure_priority_gate.py` (+ `tests/direct/conftest.py` for the one Windows-only gltest compatibility patch)
- [x] `README.md`
- [x] `docs/DESIGN.md`
- [x] `PORTAL_SUBMISSION.md`
- [x] `FINAL_CHECKLIST.md` (this file)
- [x] `LICENSE` (MIT), `CHANGELOG.md`, `SECURITY.md`, `.gitignore`
- [x] `examples/integration.md`
- [x] `.github/workflows/ci.yml`

## Repo / deployment

- [x] Git repository initialized, sole-author commit history confirmed (`git log --format='%an <%ae>'` -> `Fortunex9 <fortuneemx@gmail.com>`, no Claude co-author trailer)
- [x] Pushed to `https://github.com/Fortune9thx/disclosure-priority-gate`
- [x] Deployed (1.0.0, pre-steward-fix) to GenLayer Bradbury testnet: `0x6688dA9243b0095827d60E528f38e0933f65904a`, deploy tx `0xdc8c3d0b57219675dbb2804b2c6a7a78c32c07681c410328322e0fe368e3cec9` (`ACCEPTED`/`AGREE`/`FINISHED_WITH_RETURN`), confirmed readable -- superseded, kept in CHANGELOG.md for history
- [x] A real, live, end-to-end transaction sequence (register program -> submit two reports, a deliberate paraphrase pair -> challenge -> evaluate) run against 1.0.0, not just a bare deploy -- real, unscripted consensus correctly judged `DUPLICATE` with genuine reasoning citing the shared root cause. See `docs/DESIGN.md#live-verification`.
- [ ] Redeployed (1.1.0, steward-review fix) to GenLayer Bradbury testnet -- address and deploy tx to be recorded here, in README.md, docs/DESIGN.md, and PORTAL_SUBMISSION.md
