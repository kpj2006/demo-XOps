# demo-XOps

A throwaway repo for exercising the [XOps](https://github.com/kpj2006/XOps-me) GitHub Action
against real pull requests, before anything is pushed to the AOSSIE repo.

It tracks the `test-harness` branch of the action, so a push there changes what runs here on the
next comment — there is nothing to bump in this repo.

## The workflows

### `.github/workflows/xops-send.yml` — the payout loop

Runs on every PR comment containing `/send`. A maintainer comments:

```
/send 0x1111111111111111111111111111111111111111 2.50 USDC
```

and the workflow reads the command, checks policy, and settles. Two things about it are worth
knowing before you edit it:

- **`issue_comment` workflows always run from the default branch**, never from the PR branch.
  Editing this file on a branch has no effect until it lands on `main` — which is also why a
  fork PR cannot alter the payout logic.
- **Concurrency is repo-wide, not per-PR.** Every payout is signed by the same delegate EOA, and
  an EOA has one account nonce, so two runs signing at once would both read nonce N and one
  transaction would be silently dropped. `cancel-in-progress` stays `false`: killing a run
  mid-settlement is exactly how a payout happens without its receipt being written.

### `.github/workflows/xops-policy-test.yml` — the gate, on demand

`workflow_dispatch` with an `author_association` to present. It attempts a payout in a **real
settlement configuration** — same secrets, same driver, `mode: self` — and asserts that a
non-maintainer is refused with `POLICY_DENIED` and that no transaction hash exists, so nothing
was signed on the way to refusing.

The decision table itself is unit-tested offline in the action's `test/core/policy.test.ts`.
This workflow covers only what a unit test cannot: that the gate fires inside a real Action run.

## Whether it settles for real

`mode` is `${{ secrets.XOPS_DELEGATE_KEY != '' && 'self' || 'dry-run' }}`. With no delegate
secret configured the repo is a dry run — the command parses, the intent builds, the idempotency
key derives, and nothing moves. Add the secret and the same comment pays out for real. Real
settlement is never implicit.

## Setup

| Kind | Name | Purpose |
|---|---|---|
| Secret | `XOPS_DELEGATE_KEY` | Delegate private key. **Its presence is what enables real settlement.** |
| Secret | `XOPS_RPC_URL` | JSON-RPC endpoint for Ethereum Sepolia |
| Variable | `XOPS_SAFE` | The Safe holding the funds |
| Variable | `XOPS_TOKEN` | ERC-20 being paid out |
| Variable | `XOPS_ENABLED` | Optional kill switch. Set to `false` to refuse every payout |

`chain_id`, `allowance_module` and `explorer_url` are **not** configured here. They are functions
of the network, so XOps derives them from `network: sepolia` via its chain registry. Restating
`chain_id` is the dangerous one to get wrong: nothing compares it against `network`, so a
mismatch is signed and rejected only at broadcast, after the ledger has recorded the attempt.

The delegate key can only spend within the Safe's allowance and cannot change the recipient of
an existing one — but it **can** send that allowance anywhere, so size the period cap to what you
can afford to lose.

## Policy

Declared as inputs, not scripted in YAML. XOps evaluates all of it offline before anything is
resolved or signed, and prints a per-condition table on every run:

```
policy:
  [PASS] SETTLEMENT_ENABLED — settlement is enabled
  [PASS] MAINTAINER_APPROVED — the author may spend
  [FAIL] AMOUNT_WITHIN_CAP — 2500000 exceeds the cap of 1000000 (atomic units)
```

This repo sets `allowed_associations: OWNER,MEMBER,COLLABORATOR` and `max_per_payout: "5"` — five
USDC, written in the same units a `/send` is typed in.

## Idempotency

The receipt comment on the PR *is* the ledger. The idempotency key is derived from the platform,
repo, PR ref, recipient, network, asset and round — deliberately **not** the amount, so that
`/send alice 50` corrected to `/send alice 500` collides instead of paying 550. Re-running the
same `/send` reports `already-paid` and settles nothing; paying the same person twice for the
same PR on purpose needs a `round` bump.

## What this repo does not cover

- **Whether GitHub reports `CONTRIBUTOR` for a genuine third party.** That is GitHub's behaviour,
  not ours, and testing it needs a second account. Everything downstream of the association
  value is covered.
- **The `OWNER` path of the policy test**, which would move real money and so asserts nothing.
- Anything the action's own test suite covers offline — parsing, the decision table, idempotency
  key derivation, driver resolution, the ledger.
