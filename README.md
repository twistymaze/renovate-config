# renovate-config

Shared [Renovate](https://docs.renovatebot.com/) configuration presets for
[@ervwalter](https://github.com/ervwalter)'s repositories, plus a reusable
auto-merge workflow.

## Presets

Reference these from a repo's `.github/renovate.json` via `extends`.

### `:default`

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>ervwalter/renovate-config:default"]
}
```

Shared baseline policy, **no auto-merge**. Every PR is assigned to `ervwalter`
so it gets noticed and merged manually. Includes:

- `config:recommended`, semantic commits, dependency dashboard
- Digest pinning for Docker images and GitHub Actions (supply-chain hygiene)
- 5-day minimum release age (supply-chain cooldown); security fixes fast-tracked to 12h
  (via `vulnerabilityAlerts`, whose default would otherwise skip the cooldown entirely)
- Updates with no release timestamp (e.g. GitHub Action digest bumps) are treated as
  stable rather than leaving `renovate/stability-days` pending forever
  (`minimumReleaseAgeBehaviour: timestamp-optional`); `pin`/`pinDigest` updates skip the
  cooldown since they only pin what is already in use
- OSV vulnerability alerts
- Major updates batched into one PR per dependency manager, separate from the non-major batch
- All non-major updates batched into a single PR

Use this for repos that should review/merge dependency PRs by hand (e.g. trendweight).

### `:auto-merge`

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>ervwalter/renovate-config:auto-merge"]
}
```

Extends `:default` and adds the `automerge` label to non-major and security
updates (which the auto-merge workflow acts on), and removes the assignee from
exactly those auto-merged PRs to avoid notification spam. It also auto-merges
**major GitHub Actions updates only**. Other major updates are not labeled, so
they stay assigned to `ervwalter` for manual review.

Repos using this preset must also add the caller workflow below and have a
required status check + `allow_auto_merge` + squash enabled.

## Auto-merge workflow

Repos on the `:auto-merge` preset add a thin caller at
`.github/workflows/automerge.yml`:

```yaml
name: automerge
on:
  pull_request:
    types: [opened, reopened, synchronize, labeled]
permissions:
  contents: write
  pull-requests: write
jobs:
  call:
    uses: ervwalter/renovate-config/.github/workflows/automerge.yml@main
```

The reusable workflow enables GitHub squash auto-merge on a Renovate PR once
its required checks pass, gated to same-repo `renovate[bot]` PRs carrying the
`automerge` label.
