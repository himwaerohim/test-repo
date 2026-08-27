# SEC-003 code-owner lock — test matrix

Two independent mechanisms are under test. They fail in different ways and are
gated by different GitHub features, so they are listed separately.

| # | Scenario | Mechanism | Expected | Needs |
|---|----------|-----------|----------|-------|
| 1 | PR touches `open/notes.txt` only | workflow | lock **not engaged**, job passes | — |
| 2 | PR touches `boundary/killswitch.txt` | workflow | job **fails**, names every missing approver | — |
| 3 | PR touches `.github/CODEOWNERS` | workflow | job **fails** — the lock guards itself | — |
| 4 | PR touches `boundary/startup.txt` | workflow | lock **not engaged** — reproduces the real gap | — |
| 5 | PR touches `globtest/nested/deep.txt` | workflow | not engaged — `grep -Fxq` has no glob support | — |
| 6 | Approve, then push a new commit | workflow | job **fails again** — approval is not fresh | 2nd account |
| 7 | One of two named approvers approves | workflow | job still **fails** — this is an AND, not an OR | 2nd account |
| 8 | Code-owner review required on `main` | ruleset | merge blocked until an owner approves | ruleset + 2nd account |
| 9 | Owner listed on a line has no access | CODEOWNERS | owner silently **ignored**, does not block | ruleset + 2nd account |
| 10 | Two owners on one line | CODEOWNERS | **either** satisfies it — OR within a line | ruleset + 2nd account |

## Why the split matters

Scenarios 1–5 exercise the **workflow**, which is ordinary GitHub Actions and runs
on a private repository on a Free account. They are runnable immediately.

Scenarios 8–10 exercise the **ruleset**, which is the paid/public-only feature. They
also need a second account, because a pull request author's own approval never
satisfies a review requirement.

## The distinction the real lock rests on

CODEOWNERS gives **OR** within a line: any one listed owner satisfies it. The lock
needs **AND** — every named approver. No CODEOWNERS or ruleset configuration
expresses that, which is the entire reason the workflow exists alongside the
ruleset rather than instead of it.

## Case B, the sharpest footgun

`.github/CODEOWNERS` lists `/boundary/killswitch.txt` with two owners, then
`/boundary/` with one. The last matching pattern wins and **replaces** the earlier
line rather than merging with it, so the narrow security line above is silently
neutralised by the broad line below. Ordering, not specificity, decides.

## Case C, and why "individual and team" cannot be tested here

A team is always written `@org/team`, so a team owner can only resolve inside that
organisation. In a personal repository every team reference is unresolvable and is
dropped without error. Confirming the individual-and-team shape requires an
organisation repository.

## Observed results — 2026-08-27

| # | Expected | Observed | |
|---|----------|----------|---|
| 1 | not engaged | `named-approvers: SUCCESS` | as expected |
| 2 | blocks | `named-approvers: FAILURE` | as expected |
| 3 | blocks | `named-approvers: FAILURE` | as expected |
| 4 | not engaged | `named-approvers: SUCCESS` | **confirms the real gap** |
| 5 | not engaged | `named-approvers: SUCCESS` | as expected |

Case C is confirmed directly by the CODEOWNERS errors API, with no second account
needed. All three unresolvable owners are reported as "Unknown owner" and dropped
silently rather than blocking the merge:

    line 11: @purett — does not exist or lacks write access
    line 20: @flowaccount/team-infrastructure — cannot resolve outside its org
    line 20: @not-a-real-user-99x — does not exist

This is the decisive argument for keeping a workflow alongside CODEOWNERS. A
CODEOWNERS-only lock naming people who lack write access is silently unarmed.
