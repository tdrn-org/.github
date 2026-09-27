# tdrn-org organization defaults

This repository holds workflows that repositories in the organization **reuse**.
Nothing here runs on its own — a reusable workflow only starts when a repository
calls it.

## Reusable Go library release workflow

`.github/workflows/release-go-library.yml` tags a Go library and creates the
matching GitHub release when, since the last tag, **nothing but dependencies**
changed.

### Usage: reference it, do not copy it

Reusable workflows cannot be referenced from a repository root — they need a small
caller workflow inside the consuming repository. Add `<repo>/.github/workflows/release.yml`:

```yaml
name: release

on:
  workflow_run:
    workflows: [build]
    types: [completed]
    branches: [main]
  workflow_dispatch:

permissions:
  contents: write          # required — a called workflow can only reduce permissions

jobs:
  release:
    if: github.event_name == 'workflow_dispatch' || github.event.workflow_run.conclusion == 'success'
    uses: tdrn-org/.github/.github/workflows/release-go-library.yml@main
    with:
      sha: ${{ github.event.workflow_run.head_sha || github.sha }}
      dry-run: true        # keep true until it has run once as expected, then set false
```

Rules

- `dry-run` defaults to `true` on purpose: the workflow evaluates and reports, and
  never tags. Set it to `false` only after it has run once in that repository with
  the expected result.
- The caller must grant `permissions: contents: write`. A reusable workflow can only
  narrow the caller's permissions, never extend them.
- `workflows: [build]` must name the workflow that guards the release; the caller
  filters on its successful conclusion, so the release gate never sees a red build.

## What the gate checks

A release happens only if all of the following hold:

| Check | Condition |
|-------|-----------|
| no source change | no `*.go` file differs since the last tag |
| no internal coupling | no `tdrn-org/` line in the `go.mod` diff |
| no floor jump | the `go` directive does not change, or only in its patch component |
| something to release | `go.mod` or `go.sum` differs — docs/workflow-only changes are skipped |
| version is free | the next patch tag does not exist yet |

If a check fails, the workflow reports the reason and exits without tagging.
The next version is the last tag with its patch component incremented.
