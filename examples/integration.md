# Integration example

DisclosurePriorityGate is a reusable primitive with no domain-specific fields -- `program_id`, `scope`, `title`, and `description` are entirely caller-supplied, so any bug bounty, whistleblower-reward, prior-art registry, or research-priority program can adopt it as its duplicate-adjudication layer without modification.

## Important: writes must be called directly, not cross-contract

`submit_report`, `challenge_duplicate`, `evaluate_challenge`, `reclaim_expired_challenge`, and `claim_reward` are all `@gl.public.write` methods. GenLayer Bradbury's internal cross-contract *write* calls (`.emit(...)`) are known to be asynchronous and, for value-carrying calls with `on='finalized'`, unreliable -- and a plain value transfer to another Intelligent Contract (rather than a real EOA) can silently fail to deliver with no rescue path. This means:

- A bounty platform's frontend or keeper should call every write here as a real, directly-signed transaction -- from a program owner's wallet, a reporter's wallet, or a keeper script -- never triggered inline from inside another Intelligent Contract's write execution.
- A platform that wants to *react to* a resolved challenge or a paid reward (e.g. update its own UI state, index reports for search) should be a **pull-based** consumer: it reads this contract's state via `.view()` when someone explicitly asks, or by watching the `ChallengeResolved`/`RewardClaimed` events, rather than expecting an inline cross-contract callback.

## Example: a bug bounty program's flow via genlayer-js

```javascript
import { createClient } from "genlayer-js";
import { testnetBradbury } from "genlayer-js/chains";

const client = createClient({ chain: testnetBradbury, account, provider });

// One-time: the program owner registers the bounty.
await client.writeContract({
  address: GATE_ADDRESS,
  functionName: "register_program",
  args: [
    "acme-bounty",
    "Vulnerabilities in the Acme staking contract and its withdrawal path.",
    10_000000000000000n,  // submission_fee, in wei
    50_000000000000000n,  // min_challenge_stake, in wei
  ],
});

// A researcher submits the first report -- auto-confirms, nothing to
// compare it against yet.
const firstReportId = await client.writeContract({
  address: GATE_ADDRESS,
  functionName: "submit_report",
  args: ["acme-bounty", "Reentrancy in withdraw()", "Full technical writeup..."],
  value: 10_000000000000000n,
});

// A second researcher submits a report describing the same bug in
// different words -- starts "pending".
const secondReportId = await client.writeContract({
  address: GATE_ADDRESS,
  functionName: "submit_report",
  args: ["acme-bounty", "Funds can be drained by re-entering withdraw", "Different writeup..."],
  value: 10_000000000000000n,
});

// Anyone challenges the second report as a duplicate of the first.
const challengeId = await client.writeContract({
  address: GATE_ADDRESS,
  functionName: "challenge_duplicate",
  args: [secondReportId, firstReportId],
  value: 50_000000000000000n,
});

// Anyone triggers evaluation (the one consensus round).
const verdict = await client.writeContract({
  address: GATE_ADDRESS,
  functionName: "evaluate_challenge",
  args: [challengeId],
});
// verdict === "DUPLICATE" | "DISTINCT"

// Only the program owner, and only for a report that is
// confirmed_original and not yet rewarded:
await client.writeContract({
  address: GATE_ADDRESS,
  functionName: "claim_reward",
  args: [firstReportId],
  value: 2_000000000000000000n,  // whatever GEN the owner wants to pay
});
```

## Example: a bounty platform pulling state to display report status

```python
# Inside a separate platform contract's own write method, in its
# deterministic section (never inside a non-deterministic block --
# cross-contract calls are forbidden inside run_nondet_unsafe closures):
import json
import genlayer.gl as gl
from genlayer import Address

GATE_ADDRESS = Address("<deployed DisclosurePriorityGate address>")

def sync_report_status(self, report_id: str):
    raw = gl.get_contract_at(GATE_ADDRESS).view().get_report(report_id=report_id)
    report = json.loads(raw)
    self.cached_status[report_id] = report["status"]
    self.cached_rewarded[report_id] = report["rewarded"]
```

This pull pattern -- DisclosurePriorityGate records the verified, agreed outcome; a consumer contract reads it via `.view()` on demand -- is the verified-reliable shape for composing this primitive into a larger system on the current GenLayer Bradbury build.

## If evaluation never resolves

`evaluate_challenge` can, in principle, fail to reach validator agreement (a genuinely ambiguous pair of reports, or a transient infrastructure issue) -- when that happens, no state is written and the challenge stays `"pending"`, simply retriable by calling `evaluate_challenge` again. If it's been `"pending"` for at least 72 hours with no agreed verdict, anyone -- including the original challenger -- may call `reclaim_expired_challenge(challengeId)` to refund the staked GEN and mark the challenge `"expired"`, leaving the challenged report exactly as challengeable as it was before. A consuming platform's keeper/automation should treat a challenge that's been pending unusually long as a signal to retry `evaluate_challenge` first, and only fall back to `reclaim_expired_challenge` once the 72h window has genuinely elapsed.
