# demo-XOps

A throwaway repo for testing the [XOps](https://github.com/kpj2006/XOps-me) GitHub Action
against real pull requests, before anything is pushed to the AOSSIE repo.

## What the workflow tests

`.github/workflows/xops-dry-run.yml` runs on every pull request and checks that:

- the action's committed `dist/` bundle loads under `node20`
- inputs plumb through as `INPUT_*` environment variables
- the intent parses into a valid `Intent`
- the inline-address resolver produces a payout target
- the idempotency key derives, and is **identical** across two runs with the same inputs
- outputs land in `GITHUB_OUTPUT` and are readable by later steps

## What it does not test

Nothing is settled and no tokens move. There is no settlement driver yet, so any
`mode` other than `dry-run` deliberately fails against an empty driver registry.

Status line: first live run, triggered by PR.
