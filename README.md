# ProposalLens

Structured risk assessment of DAO proposals on GenLayer, with a registry other contracts can read.

Someone registers a proposal page. GenLayer validators fetch it and extract a fixed set of categorical and numeric fields. The risk level (LOW / MEDIUM / HIGH) is then computed by fixed rules in the contract code from those fields. No model ever writes the risk level, and no model output is accepted without an exact quote from the fetched page. Consensus is reached on the extracted fields. Anyone can challenge an assessment with a bond and counter-evidence. The final result is stored in an on-chain registry.

## What the model does and what the code does

| Step | Done by |
|---|---|
| Check the URL is https, clean, and under an owner-approved prefix | code |
| Fetch the page | each validator |
| Locate facts on the page (change type, amount phrase, recipients phrase, timelock phrase, admin powers, audit) and copy exact quotes | model |
| Drop every quote that is not an exact substring of the page; demote unquoted YES/NO answers to UNSTATED | code |
| Parse numbers from the quotes (amount, currency class, recipient count, timelock length) | code |
| Bucket the numbers, score the fields, assign LOW / MEDIUM / HIGH | code |
| Agree on the extracted fields that drive the rules, the score and the risk level | validators (exact match) |
| Re-check that the leader's quotes occur on the validator's own copy of the page and that the leader's fields follow from its quotes | each validator, in code |
| Challenge, bonds, payouts, windows, accounting | code |

## Risk rules

Fields are derived from the quotes. Points are added up and the total decides the level.

| Rule | Points |
|---|---|
| Change type: TREASURY_SPEND, GRANT, PARAMETER_CHANGE, OTHER | 1 |
| Change type: GOVERNANCE_RULE | 2 |
| Change type: CONTRACT_UPGRADE | 3 |
| USD amount below 10,000 / no amount on a non-spend proposal | 0 |
| USD amount 10,000 to 99,999 | 1 |
| USD amount 100,000 to 999,999 | 2 |
| USD amount 1,000,000 or more | 3 |
| Amount stated in a non-USD asset or without a currency (unpriced) | 1 |
| Spend or grant with no usable amount | 2 |
| Timelock of 24 hours or more | 0 |
| Timelock shorter than 24 hours, or stated without a readable length | 1 |
| Timelock not stated | 1 |
| Explicitly no timelock | 2 |
| One recipient and an amount of 100,000 USD or more | 1 |
| Spend or grant with no readable recipient count | 1 |
| Grants or changes admin, owner, minter, pauser or upgrader roles | 2 |
| Contract upgrade without a quoted audit | 1 |

Score 0-2 is LOW, 3-4 is MEDIUM, 5 or more is HIGH. A page needs a known change type and at least two grounded signals (amount, recipients, timelock, admin powers, audit); otherwise the assessment is UNDETERMINED. `get_rules()` returns the same table from the contract. Because the points add up, an explicit "no timelock" combined with a contract upgrade, a 1,000,000+ USD amount or admin powers is always HIGH.

## Lifecycle

```
register_proposal -> REGISTERED
REGISTERED --assess--> RATED (challenge window opens)
REGISTERED --assess, undetermined, 3rd time--> UNRATED
REGISTERED --stale, expire_proposal--> UNRATED
RATED --challenge_assessment (2 GEN bond)--> CHALLENGED
RATED --finalize after the window--> FINAL
CHALLENGED --resolve_challenge, rating changes--> FINAL (challenger gets bond + assurance)
CHALLENGED --resolve_challenge, rating holds or counter evidence does not cite the proposal--> FINAL (registrant gets bond + assurance)
CHALLENGED --resolve_challenge, undetermined--> RATED, bond refunded, finalizable now
CHALLENGED --stale, cancel_challenge--> RATED, bond refunded, finalizable now
```

The registrant may attach an optional assurance of 0 to 100 whole GEN when registering. It is returned at the end of the proposal's life unless a challenge changes the risk level, in which case the challenger receives it together with their bond. A challenge is only about the evidence: the counter-evidence page must cite the proposal URL and sit under an approved prefix. The original page is not extracted again: fields stated on the counter page replace the same fields in the stored, consensus-accepted rating, the merged fields are re-scored by the same rules, and the challenge succeeds only if the risk level changes.

## Contract

`contracts/proposallens.py` (single file, deploy as is). Constructor: `challenge_window_seconds`, `stale_after_seconds` (each 60 to 604800). Suggested values: `900, 1800` for testing and `172800, 259200` for production.

| Method | Who | Purpose |
|---|---|---|
| `add_prefix(prefix, label)` / `remove_prefix(prefix)` | owner | manage the https URL prefixes proposals may be registered under |
| `register_proposal(url)` payable, 0-100 whole GEN | anyone | register a proposal page; invalid input is refunded to the caller's balance, not reverted |
| `assess(proposal_id)` | anyone | consensus extraction and rating |
| `challenge_assessment(proposal_id, counter_evidence_url)` payable, exactly 2 GEN | anyone but the registrant | dispute the rating |
| `resolve_challenge(proposal_id)` | anyone | consensus extraction of the counter evidence, merged with the stored rating |
| `cancel_challenge(proposal_id)` | anyone | refund a challenge that stayed unresolved past the stale limit |
| `finalize(proposal_id)` | anyone | close an unchallenged rating after the window |
| `expire_proposal(proposal_id)` | anyone | release a proposal stuck before its first rating |
| `withdraw()` | anyone | pull payment of the caller's balance |

Views: `get_params`, `get_rules`, `get_stats`, `list_prefixes`, `list_proposals(offset, limit)`, `get_proposal`, `find_proposal(url)`, `get_risk(proposal_id)`, `get_risk_by_url(url)`, `meets_max_risk(proposal_id, max_risk)`, `get_balance`, `get_accounting`.

## Using the registry from another contract

Only FINAL ratings are exposed through `get_risk`, `get_risk_by_url` and `meets_max_risk`. An empty string or `False` means "no final rating".

```python
registry = gl.get_contract_at(Address("0xPROPOSALLENS_ADDRESS"))
if not registry.view().meets_max_risk("p1", "MEDIUM"):
    raise Exception("proposal has no final rating of MEDIUM or lower")
```

## Deployment

Deploy `contracts/proposallens.py` on GenLayer Studio. The contract is submitted on its own; there is no frontend in this repository. `evidence/` holds sample proposal pages used for live testing. The sample prefix is `https://raw.githubusercontent.com/Mary1270/ProposalLens/main/evidence/`.

## Tests

```
python3 -m unittest discover -s tests -p "test_*.py"
```

No pip install is needed. The suite runs against a small offline stand-in for the GenLayer SDK (`tests/genlayer_stub.py`) with a scripted stand-in for the model. After every call it checks that the contract's real GEN balance equals its recorded obligations. It also covers the URL rules, each parser, every rule in the table, the validator checks against a tampered leader, all challenge paths, stuck-state exits, and a randomized sequence test. The stand-in model cannot show how a real model behaves; that is covered by the live tests on Studio.

See `ARCHITECTURE.md` for the state machine, consensus design, accounting and trust assumptions.

## License

MIT
