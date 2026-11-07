# ProposalLens architecture

## Entities

- **Prefix**: an https URL prefix with a label, added by the contract owner. Proposals can only be registered under a current prefix. The label is copied onto each proposal as its DAO name.
- **Proposal**: id `p1`, `p2`, ... with the canonical page URL, registrant, state, optional assurance, rating, and challenge data. One URL maps to one proposal; a URL whose proposal ended UNRATED may be registered again.
- **Rating**: risk level, score, the list of rule hits (`why`), the derived fields, and the grounded quotes (`raw`) they came from.

## Proposal state machine

```
register_proposal -> REGISTERED
REGISTERED --assess RATED--> RATED
REGISTERED --assess UNDETERMINED (attempts+1)--> REGISTERED, or UNRATED on the 3rd attempt (assurance refunded)
REGISTERED --stale, expire_proposal--> UNRATED (assurance refunded)
RATED --challenge_assessment, once, before challenge_end--> CHALLENGED
RATED --finalize after challenge_end--> FINAL (assurance refunded)
CHALLENGED --resolve, merged risk differs--> FINAL (challenger: bond + assurance), rating replaced, original_risk kept
CHALLENGED --resolve, merged risk equal--> FINAL (registrant: bond + assurance)
CHALLENGED --resolve, counter page lacks the proposal URL--> FINAL (registrant: bond + assurance)
CHALLENGED --resolve, undetermined--> RATED, bond refunded, challenge_end = now
CHALLENGED --stale, cancel_challenge--> RATED, bond refunded, challenge_end = now
```

Every state has a path to a terminal state (FINAL or UNRATED) that needs no particular party. A consensus failure on the network leaves storage untouched, and the stale timers (`stale_after_seconds`) then open `expire_proposal` or `cancel_challenge`. Only FINAL ratings are served by the registry views.

## Extraction and rating pipeline

`_assess_single` (and, for the counter page, `_assess_challenge`) runs identically on the leader and on every validator:

1. Fetch the page (first 12,000 characters). A failed fetch is UNDETERMINED.
2. Ask the model for a fixed JSON object: `change_type` plus, for amount, recipients, timelock, admin powers and audit, an exact quote and a YES / NO / UNSTATED flag. The prompt declares the page untrusted and forbids rating.
3. `_ground`: each quote is kept only if it is at least 3 characters and, after whitespace and case normalisation, occurs in the page. Invalid enumerations become UNKNOWN or UNSTATED.
4. `_derive`: a YES or NO flag without a surviving quote becomes UNSTATED. The amount, recipient count and timelock length are parsed from the quotes by regular expressions, never by the model. A USD amount needs a USD-pegged token or `$` and no foreign asset token; otherwise it is UNPRICED. Numbers are bucketed into bands.
5. `_rate`: points are summed (see README) and mapped to LOW / MEDIUM / HIGH. UNKNOWN change type or fewer than two grounded signals gives UNDETERMINED.

Conservative defaults: a fact that cannot be grounded or parsed is treated as unstated, and an unstated timelock, recipient count or amount adds points rather than removing them. Text the model invents is dropped by the quote check, so it cannot lower a score.

## Consensus

Both consensus calls use `gl.vm.run_nondet_unsafe(leader, validator)` from module-level functions that capture only plain strings, never the contract.

- The leader returns the full result as canonical JSON. Errors inside the closure become an ordinary UNDETERMINED result.
- A validator re-runs the whole pipeline on its own copy of the page and requires an exact match of the consensus key: status, change type, amount band, timelock class, score and risk level, plus every other extracted fact at the granularity the rules use: the recipient band (SINGLE / MULTIPLE / UNSTATED) when the proposal spends funds or the amount is LARGE or XLARGE, admin powers as YES or not, and the audit flag as YES or not for contract upgrades. Facts that cannot change any rule (for example an audit answer on a treasury spend) are not part of the key, so harmless wording differences do not block consensus. Raw numbers are not compared; because the risk level and every band that feeds it are compared exactly, two validators cannot take different decisions near a band edge.
- The validator also checks the leader's output deterministically: every leader quote occurs in the validator's page, and re-deriving and re-scoring the leader's quotes reproduces the leader's fields, score and risk. A leader therefore cannot store a fabricated quote or a number that does not follow from its quotes. A tampered or malformed leader result is rejected.
- A challenge does not re-extract the original page. It starts from the grounded quotes stored with the consensus-accepted rating, so a challenger cannot win just because a second extraction of the same page came out differently. Only the counter page is fetched and extracted. It must contain the full stored proposal URL as a whole token (case-insensitive, within the first 12,000 characters; a longer URL that merely starts with it does not count); without it the result is NO_ANCHOR and no model call is made. Counter fields replace stored fields only where the counter page states them with a grounded quote; a counter page whose change type is OTHER or UNKNOWN does not replace the stored change type. The merged fields are scored by the same rules and compared with the stored risk level. Validators check that the leader's counter quotes occur on their own copy of the counter page and that the merge reproduces the leader's result.

Pages should be static text, ideally pinned to a commit. If the page changes between validators' fetches, validators disagree and the transaction does not complete; the stale paths cover that. A model answer that is not a string (for example `null`) is treated as empty, never as the text "None".

## URL rules

Proposal and counter-evidence URLs must be https, ASCII with no whitespace or control characters, no credentials, no non-443 port, no `..`, `.` segments, `//` in the path, `%2e`, `%2f`, `%5c` or `%00`, no query and no fragment, and, after lower-casing the host and dropping `:443`, start with one of the owner's prefixes. Prefixes themselves must end with `/`, have a non-empty path and no query. Look-alike hosts and `host@evil` forms fail the parsed-URL checks.

## Money

All amounts are wei internally; users enter whole GEN. Payouts are pull-based: every credit goes to `balances` and `withdraw` transfers. Payable methods never raise on bad input; they credit the attached value back to the caller's balance. Invariant checked in tests after every call:

```
contract balance == sum(balances) + sum(open challenge bonds) + sum(open assurances)
```

`get_accounting` computes the right side by scanning every proposal and also reports whether it matches the running totals.

## Trust assumptions

- The owner is trusted to choose prefixes under content the DAO controls. The owner cannot change ratings, windows, bonds or any balance, and removing a prefix does not affect existing proposals. A prefix on a place where anyone can publish lets anyone author a proposal page; consumers of the registry should check the proposal's `label` and URL.
- A counter-evidence page is treated as an authoritative amendment for the fields it states. Challenging therefore requires a page under an approved prefix that cites the proposal URL.
- The risk rules are a transparent summary of stated facts. They do not read contract code and say nothing about facts the page leaves out beyond the UNSTATED penalties.
- Validators are trusted only to locate quoted facts. They never produce the risk level, and the code checks their quotes.
- A registrant who attaches assurance accepts that a successful challenge pays it to the challenger, including when the cause is an ambiguous page.

## Known limits

- One challenge per proposal; one rating per URL. A changed proposal needs a new URL.
- Amounts are only priced when the quote itself is in USD or a USD-pegged token.
- Quotes are capped at 200 characters and pages at 12,000 characters.
- At most 500 proposals and 10 prefixes; list views are paginated (50 per page).
- Time comes from `datetime.datetime.now()` as seen by GenVM.
