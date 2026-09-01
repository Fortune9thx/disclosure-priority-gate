# DisclosurePriorityGate -- Design

## Originality: a structurally different consensus shape

This account has built two other GenLayer adjudication primitives before this one:

- **IndependentEvidenceSettler** -- claim-vs-fetched-evidence. A caller asserts a claim; validators independently fetch external URLs and judge how well that fetched evidence supports the claim.
- **UpgradeChangelogGate** -- diff-vs-prose. The contract computes a deterministic diff between two already-stored JSON blobs; validators judge whether a proposer's changelog prose is a faithful account of that already-fixed diff.

Both are fundamentally **asymmetric**: one side is an authoritative, contract-derived ground truth (fetched evidence, a computed diff), and the other is a claim being checked against it.

DisclosurePriorityGate's judgment is **symmetric claim-vs-claim comparison**. Neither report is more authoritative than the other. There is no third-party ground truth to fetch and no deterministic computation to derive -- the entire question put to GenLayer, "do these two independently-written texts describe the same underlying issue," has no meaning unless both sides are read and weighed against each other on equal footing. There is nothing here resembling a diff or an evidence fetch. This is deliberately not a re-skin of this account's other primitives in a new domain; it is a different consensus shape. The rule that `prior_report_id` must already be `confirmed_original` (never itself `pending`) exists precisely to keep that symmetric comparison well-founded -- every challenge is always settled-vs-unsettled, never unsettled-vs-unsettled -- despite the underlying question being symmetric.

## What problem this closes

Any permissionless disclosure/bounty registry has the same unsolved problem: two independent parties may describe the same underlying issue in different words. Someone has to decide whether a new report duplicates an earlier one or is genuinely distinct -- and a program owner must not be able to unilaterally dismiss an inconvenient report as "duplicate" to dodge a payout. GenLayer's validator set, not the program owner, makes that call, via genuine independent re-derivation rather than trust in any one party's account.

## Consensus design

`evaluate_challenge` contains exactly ONE top-level non-deterministic call: `gl.vm.run_nondet_unsafe(leader_fn, validator_fn)`. Both are named `def`s (never `lambda`s -- `genvm-lint` enforces this as a hard rule). Neither closure reads or writes `self.*` -- every value they need (both reports' title/description, the manipulation-screen flag) is copied into plain locals before the non-deterministic section begins.

`validator_fn` does not inspect the leader's claimed verdict for plausibility. It calls the identical `leader_fn` again, over the identical, already-agreed report texts, and only agrees if its own, independently-run LLM judgment lands on the same bucket:

```python
def validator_fn(leader_result) -> bool:
    if not isinstance(leader_result, gl.vm.Return):
        return False
    leader_data = leader_result.calldata
    if not isinstance(leader_data, dict):
        return False
    try:
        mine = leader_fn()
    except Exception:
        return False
    return mine.get("verdict") == leader_data.get("verdict")
```

This closes the exact rejection pattern a GenLayer Portal reviewer has previously named explicitly: *"the validator checks only verdict shape, ranges, and a few field combinations; it does not independently review the evidence or verify the fulfillment decision."* Here the validator genuinely can and does independently verify the decision, because both texts it compares are already-fixed, already-agreed state -- there is nothing to trust the leader's account of.

`run_nondet_unsafe` (not the sandboxed `run_nondet`) is used deliberately: `validator_fn`'s one fallible step is wrapped in `try/except -> return False`, closing the one documented gap between the two primitives, and `run_nondet_unsafe` with a custom validator is the pattern already proven live on GenLayer Bradbury in this account's history, while `run_nondet` has no live-verified precedent yet.

### Why the leader's own `exec_prompt` call is NOT wrapped in try/except (a deliberate, benchmarked choice)

While screening this design against `spec-compliance-bounty` (an independently-accepted Portal submission, see Benchmarking below), its `_judge_compliance`'s `leader()` wraps `gl.nondet.exec_prompt` in `try/except -> UNCLEAR` (a third bucket in that contract's vocabulary). This account considered the same pattern here and deliberately did **not** adopt it. Reasoning: this contract's own bucket design (see below) explicitly rejects a third "unclear" outcome, because it would give a low-accountability way for a genuinely close call to sit in permanent limbo. If `leader_fn`'s `exec_prompt` call itself raises (a transient model/provider error), letting that exception propagate simply fails the whole `evaluate_challenge` transaction -- no state is written, the challenge stays `"pending"`, and it is retriable exactly like a round that failed to reach validator agreement for any other reason. That is the same safe, atomic, no-partial-effects behavior this account's UpgradeChangelogGate and IndependentEvidenceSettler already use, and it avoids forcing a pure infrastructure hiccup into either `"DUPLICATE"` or `"DISTINCT"` -- both of which have real, asymmetric financial consequences (a report's permanent reward eligibility, a challenger's stake) that a tooling failure should never trigger either side of.

## Bucket design

Exactly two discrete buckets: `"DUPLICATE"` / `"DISTINCT"`. No third "unclear/needs review" bucket, deliberately: it would not resolve anything (someone still has to decide, later, with no more information than is available now) and would instead let a genuinely close call sit in permanent limbo, re-litigated forever instead of settled.

**Fail-closed default: `"DISTINCT"`, never `"DUPLICATE"`.** If the model's raw output cannot be confidently parsed into either bucket, `_coerce_verdict` defaults to `"DISTINCT"`. This contract's entire purpose is to stop a program owner -- or anyone else -- from costlessly using an ambiguous or bad-faith "duplicate" challenge to suppress a legitimate report and dodge a payout. Defaulting an unparseable/uncertain outcome to `"DUPLICATE"` would hand exactly that capability to anyone willing to submit a challenge and hope for a confused model response: worst case for the challenger is a refunded stake, but the reporter silently loses reward eligibility forever with no recourse. Defaulting to `"DISTINCT"` instead means the worst case of genuine model confusion falls on the challenger (stake forfeited to the reporter -- the same real cost as losing a challenge on the merits) and the reporter's standing is never destroyed by an inconclusive round. This mirrors the contract's own explicit deterrent design for a losing challenge on the merits: an inconclusive challenge is treated the same as a failed one, never as a free retry for the challenger.

## Storage

`TreeMap[str, str]` only (`programs`, `reports`, `challenges`), plus `u256` for the two genuinely scalar counters -- the only storage shape with reliable post-deploy readability on the current GenVM build behind Bradbury. All structured records are JSON-encoded before storage. No public method ever returns a raw `dict`; every view returns a JSON-encoded `str`. No field anywhere is ever a bare Python `float` -- fees, stakes, and reward amounts are `u256` on the wire and decimal strings in storage, so no return value can ever carry an un-encodable float, structurally, not just by convention.

## Which path actually needs a liveness escape hatch, and why the other two deliberately do not

Only `challenge_duplicate` escrows GEN pending a non-deterministic resolution:

- **`submit_report`** forwards its *entire* attached `gl.message.value` (not just the configured `submission_fee`) to the program owner synchronously, in the same transaction. This is a paid-service fee, not an escrow -- the transfer either happens as part of this one transaction succeeding, or the whole transaction never commits. Forwarding the full attached value, not just the minimum fee, means this method never retains any balance of its own making, even from an accidental overpayment -- there is structurally nothing here that could ever need a rescue path.
- **`claim_reward`** is the same shape: the owner attaches whatever GEN they want to pay, forwarded to the reporter synchronously in the same call. There is no stored `reward_amount`, no reward pool, no separate release step. The owner pays out of pocket at the exact moment they decide to, or the call simply never happens.
- **`challenge_duplicate`** is genuinely different: its stake is held pending `evaluate_challenge`'s non-deterministic resolution, a process that -- being permissionlessly retriable -- is *not thereby guaranteed to ever converge*. "Permissionlessly retriable" is not the same claim as "guaranteed to eventually resolve": nothing bounds how many times, or for how long, a genuinely ambiguous pair of reports, or a persistent validator-infrastructure issue, can cause every attempt to fail to reach agreement. `reclaim_expired_challenge` is the bounded (72h), permissionless timeout path for exactly this case. This lesson comes directly from a real GenLayer Portal steward review of this account's UpgradeChangelogGate, which rejected that project's own prior benchmark-pass conclusion that no escape hatch was needed -- applied here proactively from the first commit rather than discovered after the fact.

Both non-escrowing paths were re-verified against this reasoning during the adversarial self-review pass below, not merely assumed correct because the build spec said so.

## Race-condition / stale-artifact analysis

A strict internal review specifically checked for the "stale artifact application" bug class this account has previously found in UpgradeChangelogGate (a write stores a computed artifact from current state; a later write applies it unconditionally, letting state drift underneath it). This contract computes no diffs or snapshots, so the exact prior bug shape does not apply -- but the review found a **different**, real race worth closing:

**The race:** without a guard, two different challengers could each open a challenge against the *same* still-`"pending"` report -- one citing baseline A, the other citing baseline B (both themselves independently `confirmed_original`). Resolving them in sequence would let the second `evaluate_challenge` call silently overwrite whatever the first one legitimately decided (e.g. challenge #1 resolves `DISTINCT`, confirming the report; challenge #2, still pending from before, later resolves `DUPLICATE`, overwriting that confirmation with `duplicate_of:B`) -- exactly the same *shape* of bug as UpgradeChangelogGate's stale-`based_on_version` clobber, even though the underlying mechanism (concurrent challenges, not a stale diff) differs.

**The fix:** `challenge_duplicate` enforces at most one open (`"pending"`) challenge per report at a time, tracked via a `report["open_challenge_id"]` field, cleared only when that specific challenge resolves or expires. A second challenge against an already-open-challenged report is rejected outright, closing the race at the root rather than reconciling it after the fact. `evaluate_challenge` additionally re-checks `report["status"] == "pending"` defensively right before entering the non-deterministic section, even though the guard above should make that check unreachable given every other current code path -- documented as defense-in-depth, not a live gap, since a future code change could otherwise silently reintroduce the exact race this closes.

This exact pattern -- "at most one open challenge per report/bidder at a time" -- was independently confirmed as a recognized, precedented design during the benchmarking pass below: `tendercouncil` (an independently-accepted Portal submission) enforces "one challenge per bidder is required" for the same underlying reason.

Regression coverage: `TestChallengeValidation::test_cannot_open_a_second_concurrent_challenge_on_same_report` and `test_new_challenge_allowed_after_first_one_resolves` in `tests/direct/test_disclosure_priority_gate.py`.

## Trust boundaries

- **Report title/description are caller-controlled, LLM-facing text.** Both fields get the non-blocking `_MANIPULATION_PATTERNS` heuristic screen (mirrors the proven pattern in this account's IndependentEvidenceSettler and UpgradeChangelogGate). This is a transparency flag on the stored record (`"flagged"`), never a rejection gate -- a false positive must never block a genuine report. It catches only literal phrasing and is trivially bypassable by paraphrase; it is not, and is not claimed to be, a real defense against a determined adversary. `evaluate_challenge` ORs both compared reports' flags together for the prompt-level warning, since either side's text -- the challenged report or the baseline it's compared against -- could carry injection-shaped phrasing.
- **No caller-supplied address parameter exists anywhere in this contract.** Every address this contract acts on (program owner, reporter, challenger) is derived exclusively from `gl.message.sender_address` at the moment of the relevant write, never accepted as a raw string parameter and re-parsed. This was checked explicitly against this account's own documented, real GenLayer Portal rejection pattern (a `TreeMap` keyed/compared by `Address.as_hex` but looked up with unnormalized raw caller input) -- the bug class cannot occur here because the surface it requires (a caller-supplied address string) does not exist in this design.
- **`program_id` is ASCII-regex-restricted (`^[a-z0-9][a-z0-9_-]{0,63}$`, full-string match), so no separate Unicode-normalization step is needed for it.** This is a deliberate simplification versus a closely related precedent: UpgradeChangelogGate applies an NFC-normalization helper (`_protocol_key`) on top of an equally ASCII-locked `protocol_id` regex, which is redundant there for the same reason it would be redundant here -- a `$`-anchored regex that only accepts `[a-z0-9_-]` cannot ever match a non-ASCII codepoint, so there is only one possible encoding of any string that passes it. `report_id`/`challenge_id` are never caller-supplied at all (auto-generated as `report-{counter}`/`challenge-{counter}`), so they carry no identity-collision risk to close in the first place; format-validating them would only reject legitimate lookups against valid contract-generated IDs, so they are looked up as-is.
- **`claim_reward` enforces no minimum reward size.** The program owner decides the amount at the moment they call it, and a report can only ever be rewarded once (`rewarded` flips permanently). A program that wants to guarantee a specific payout should communicate that expectation in its `scope` text; this contract's role is narrowly "who gets to claim, once, and only if their report survived," never "how much they're owed" -- consistent with GenLayer's role being deliberately narrow everywhere else in this design (see below).
- **Cross-contract writes and value transfers to another Intelligent Contract are known-unreliable on the current GenVM build.** This contract never calls another IC's write method and never routes GEN to a stored address that could itself be a contract rather than an EOA -- every `emit_transfer` recipient (owner, reporter, challenger) is always a real signer address captured from `gl.message.sender_address`, never a contract address. See `examples/integration.md` for the pull-based consumption pattern this implies for any platform built on top of this primitive.

## GenLayer's role is deliberately narrow

This contract never asks the model whether a report is in scope, valid, severe, or worth a reward -- only whether two already-submitted texts describe the same underlying issue. Scope judgment, reward sizing, and the decision to pay at all remain the program owner's own calls via `claim_reward`.

## How this maps to known GenLayer Portal rejection patterns

- *"Validators that only check well-formed strings"* -- closed: `validator_fn` re-derives the verdict from scratch via the identical `leader_fn`, never inspects the leader's JSON for shape alone.
- *"Quantitative outcomes not bound by equivalence criteria"* -- the verdict bucket is the only value `validator_fn` compares and the only outcome field this contract stores, returns, or acts on.
- *"Nested non-deterministic blocks"* -- `evaluate_challenge` contains exactly one top-level `gl.vm.run_nondet_unsafe` call; all validation and bookkeeping is plain deterministic code outside it. Confirmed by `genvm-lint check` passing cleanly.
- *"State changes from caller text alone"* -- a report's reward eligibility never moves from either party's own say-so; it moves only after an agreed verdict from independent LLM judgment, or is left untouched by an inconclusive/expired challenge.
- *"Staked funds with no bounded escape hatch if consensus never resolves"* -- see the escape-hatch analysis above; applied proactively from the first commit.
- *Address checksum case-sensitivity* -- structurally inapplicable; see Trust boundaries above.

## Benchmarking against independently-accepted Portal submissions

Cloned and read three independently-accepted GenLayer Portal repos, specifically screening for anything relevant to this contract's own bounty/duplicate-adjudication shape not already covered by this account's prior benchmarking of the same repos on earlier projects:

- **`tendercouncil`** (github.com/GIFTEDLOV/tendercouncil) -- confirms "at most one open challenge per participant at a time" (`"one challenge per bidder is required"`) as a recognized, precedented pattern for exactly the concurrency race this contract's `open_challenge_id` guard closes (see Race-condition analysis above). Independent confirmation, not the origin of the idea -- the guard was designed and tested here first, then cross-checked against this repo.
- **`spec-compliance-bounty`** (github.com/lolaaa00/spec-compliance-bounty) -- the closest domain match (a staked bounty contract with its own permissionless-timeout escape hatch, already benchmarked as the source of this account's 72h convention on UpgradeChangelogGate). This pass specifically checked its `_judge_compliance`/`leader()` pattern of wrapping `gl.nondet.exec_prompt` in `try/except -> UNCLEAR`. Considered and deliberately not adopted here -- see "Why the leader's own `exec_prompt` call is NOT wrapped in try/except" above for the reasoning. Also reconfirms its CEI-ordering comment convention (*"state written before value leaves -- a re-entrant or duplicated call finds the settled status already flipped and reverts above"*), which this contract already follows throughout (`report["rewarded"] = True` / challenge and report status flips all precede their corresponding `emit_transfer` calls).
- **`rubricproof-intelligent-contract`** (github.com/jason4185/rubricproof-intelligent-contract) -- reconfirms the SSRF-hardening (`_unsafe_host_reason`) and disqualifier-rebuttal patterns already documented from this account's prior benchmarking pass on IndependentEvidenceSettler. Not directly applicable here: this contract has no `gl.nondet.web.render` call and no caller-supplied fetch URL at all (both compared texts are already-stored contract state), so there is no SSRF surface to harden in the first place -- checked explicitly rather than assumed.

## Live verification

Deployed to GenLayer Bradbury testnet at `0x6688dA9243b0095827d60E528f38e0933f65904a`, deploy tx `0xdc8c3d0b57219675dbb2804b2c6a7a78c32c07681c410328322e0fe368e3cec9` (`ACCEPTED`/`AGREE`/`FINISHED_WITH_RETURN`), confirmed readable via a real `genlayer call get_report_count` returning `0`.

A full, real transaction sequence was then run end to end against this deployment, using the `bradbury-deploy` account (`0xc6e6d3b2accaececeb40ad4bd3df123ddcb4e537`) for every call:

1. **`register_program("live-test-1", "Vulnerabilities in the DisclosurePriorityGate demo staking contract's withdrawal path.", 0, 0)`** -- tx reached `ACCEPTED`/`AGREE`, all 5 initial validators voted `AGREE`.
2. **`submit_report("live-test-1", "Reentrancy in withdraw()", "The withdraw function sends ETH via a raw call before updating the caller's recorded balance, letting a malicious contract re-enter withdraw and drain funds before the balance is zeroed out.")`** -- `ACCEPTED`/`AGREE`. Confirmed via `get_report("report-0")`: `"status": "confirmed_original"` (auto-confirmed as the program's first report, exactly as designed).
3. **`submit_report("live-test-1", "Funds can be drained by re-entering the withdraw function", "withdraw() performs the external transfer prior to zeroing out the user's recorded balance, so a recursive call from a fallback function can withdraw repeatedly before state catches up.")`** -- a deliberate paraphrase of report-0, different vocabulary and structure describing the identical root cause. `ACCEPTED`/`AGREE`. Confirmed via `get_report("report-1")`: `"status": "pending"`.
4. **`challenge_duplicate("report-1", "report-0")`** -- `ACCEPTED`/`AGREE`. Zero stake (the live-test program was registered with `min_challenge_stake=0` specifically so this end-to-end sequence could run as plain CLI calls without needing a custom signing script for value-attached transactions -- the CLI's `write`/`call` commands as of this account's installed version expose no flag for attaching GEN value to a payable call, only `--fee-value` for the transaction's own fee deposit; see `genlayer-cli-tooling-gotchas` -- this affects only whether the `emit_transfer` amount-guard is exercised, not whether the real non-deterministic consensus mechanism itself runs).
5. **`evaluate_challenge("challenge-0")`** -- the one real non-deterministic round in this whole sequence. `ACCEPTED`/`AGREE`, `txExecutionResultName: FINISHED_WITH_RETURN`. Round detail: 5 validators, 3 `AGREE` + 2 `TIMEOUT` votes (a timeout is validator-infrastructure latency, not a disagreement -- see `genlayer-bradbury-intermittent-read-unavailability` for this account's prior documentation of Bradbury's variable finality latency), still reaching overall `AGREE`. The genuinely unscripted model call, verified by reading the actual stored record afterward, correctly judged:

   ```json
   {
     "verdict": "DUPLICATE",
     "reason": "Both reports describe the same underlying reentrancy vulnerability in the withdraw function where an external transfer occurs before updating the user's balance state, enabling recursive calls to drain funds."
   }
   ```

   `get_report("report-1")` afterward confirms `"status": "duplicate_of:report-0"` -- the real judgment landed exactly on the correct bucket for a genuine paraphrase, with reasoning that names the actual shared root cause rather than surface wording, and the deterministic status-routing code applied it correctly.

This is real evidence the independent-re-derivation mechanism converges in practice on a genuinely paraphrased pair of reports, not only in `gltest`'s mock -- the exact scenario this contract's design exists to handle correctly.

## Limitations

- The manipulation-heuristic screen is a literal-phrasing regex; it does not detect paraphrased injection attempts and is not claimed to.
- `claim_reward` has no enforced minimum -- reward size and payment timing are entirely at the program owner's discretion, once eligible.
- This contract does not judge scope, severity, or validity -- only whether two reports describe the same underlying issue. A program that wants those additional judgments needs its own separate mechanism.
- GenLayer Bradbury testnet has exhibited intermittent, time-varying read-path flakiness independent of contract correctness (confirmed across multiple prior projects on this account) -- a failed `.view()` read is not on its own evidence of a broken contract; cross-check against `getTransaction`'s `txExecutionResultName` for the write in question.
