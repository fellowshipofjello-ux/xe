# Consensus

> Two-phase ORV finality — every block finalizes at ≥67% of delegated XE weight.

Every block reaches a finalized state via representative voting — not just blocks that fork. Each chain **position** (account + `Previous` hash) runs an election; a fork just means the election has more than one candidate.

## Two-Phase Voting [#two-phase-voting]

| Phase       | Vote kind       | Behaviour                                                                                                                                                                                                                   |
| ----------- | --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1. Converge | `Final = false` | **Mutable.** A representative votes for its preferred candidate (lowest hash among candidates it holds) and may re-vote as its preference shifts. Converge votes establish a leader; they never finalize anything.          |
| 2. Final    | `Final = true`  | **Irrevocable.** Once a candidate has a visible ≥67% converge supermajority, the representative writes a **write-once commit-lock** for the position and casts a final vote for the locked hash. Only final votes finalize. |

A block finalizes when representatives holding **≥67% of delegated XE weight** have final-voted it:

```
blockWeight * 100 >= totalWeight * 67
```

Uses `big.Int` arithmetic. Deterministic preference: lexicographically lowest hash.

The commit-lock is never deleted — not on resolution, rollback, or restart — so a representative is structurally unable to final-vote two hashes at one position. Adversarial timing can delay finalization, never double it. The 67% quorum lock path additionally waits out a **3-second candidate-stability window** after the last new candidate appears, so a transient supermajority on a higher-hash sibling cannot grab the lock.

## Delegation [#delegation]

* Voting weight is the delegated **XE** balance, in micro-XE. XUSD confers **no** weight: XUSD is mintable (operator-minted via `sys.minter` only), and mintable supply must not mint consensus weight.
* Weight snapshots frozen at conflict detection time
* Persistent via DelegationStore interface
* `sync.RWMutex` for concurrency

## Dependency Gating [#dependency-gating]

A representative withholds its own vote for a block whose dependencies are not yet finalized locally — its `Previous`, plus the cross-account `Source` for receives and lease lifecycle blocks. It still tallies incoming votes, so a lagging node never deadlocks an election.

## The Finality Wall [#the-finality-wall]

* Each account has a `final_height` watermark — the height of its highest finalized block. The ledger refuses to replace any block at or below it.
* **Spendable balance** counts only finalized inflows; an account with nothing finalized has no spendable balance.

## Fallback Resolution [#fallback-resolution]

If 67% is unreachable (non-voting reps inflate the total), then after 10 seconds only the **lowest-hash** final-voted candidate may finalize, and only if it holds a &#x2A;*strict majority (>50%)** of total delegated weight in final votes. Otherwise the node withholds rather than risk divergent finalization — since each representative is commit-locked to one hash per position, at most one candidate can ever hold a strict majority.

## Conflict Detection [#conflict-detection]

* A conflict = two blocks with the same `Previous` hash (equivocation)
* Max 10 block hashes per conflict
* Conflicting blocks placed in staging
* Conflict callback fires on detection **and** whenever a new sibling body is staged for an existing conflict; only gossip re-deliveries of an already-staged body are suppressed

## Voting [#voting]

* Per-conflict mutex prevents double-voting
* Vote buffering for votes arriving before local conflict detection (max 10 per conflict)
* ±5 minute timestamp window
* Vote wire format: 204 bytes (138 signing bytes + 2 length + 64 signature) — see [Binary Encoding](/docs/encoding)

## Quorum Manager [#quorum-manager]

* Stale conflict sweep every 15 seconds
* Block status: Pending (0), Finalized (1), Rejected (2)
* Block swap mechanism for staged winners

---

Canonical HTML: https://test.network/docs/consensus · This Markdown is served for `Accept: text/markdown`.
More for agents: [llms.txt](https://test.network/llms.txt) · [Full docs as Markdown](https://test.network/llms-full.txt) · [OpenAPI](https://test.network/openapi.json) · [Sitemap](https://test.network/sitemap.xml)
