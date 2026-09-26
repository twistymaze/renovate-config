# renovate-config

Shared [Renovate](https://docs.renovatebot.com/) configuration presets for
[twistymaze](https://github.com/twistymaze) repositories, plus a reusable
auto-merge workflow.

## Presets

Reference these from a repo's `.github/renovate.json` via `extends`.

### `:default`

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>twistymaze/renovate-config:default"]
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
- No hourly PR creation limit; Renovate's default concurrency limits still apply

Use this for repos that should review/merge dependency PRs by hand.

### `:auto-merge`

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>twistymaze/renovate-config:auto-merge"]
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
    types: [opened, reopened, synchronize, labeled, review_requested]
permissions:
  contents: write
  pull-requests: write
jobs:
  call:
    uses: twistymaze/renovate-config/.github/workflows/automerge.yml@main
```

The reusable workflow enables GitHub squash auto-merge on a Renovate PR once
its required checks pass, gated to same-repo `renovate[bot]` PRs carrying the
`automerge` label.

If Copilot cloud agent pushes fixes to a Renovate PR, auto-merge is turned off
until Copilot finishes and requests a review, then re-enabled for its final
commit. This needs `review_requested` in the caller's trigger types.

### Optional authentication

Existing callers need no changes: without credentials the workflow uses
`GITHUB_TOKEN`. This can merge PRs, but GitHub suppresses subsequent `push`
workflows, including builds and release automation.

For repositories that need those workflows, use a private GitHub App:

```yaml
jobs:
  call:
    uses: twistymaze/renovate-config/.github/workflows/automerge.yml@main
    with:
      app-client-id: ${{ vars.AUTOMERGE_APP_CLIENT_ID }}
    secrets:
      app-private-key: ${{ secrets.AUTOMERGE_APP_PRIVATE_KEY }}
```

Pin the workflow reference to a reviewed commit SHA in consumers. Create the App
under your GitHub account, disable webhooks, and allow installation only on your
account. Grant repository Contents and Pull requests read/write, and Workflows
write (needed for dependency PRs that update workflow files). Metadata read access
is automatic. No account, organization, or administration permissions are needed.
Install it on selected repositories only. Save its client ID as the repository
variable `AUTOMERGE_APP_CLIENT_ID`, and its generated PEM private key as the Actions
secret `AUTOMERGE_APP_PRIVATE_KEY`. No OAuth client secret or callback is needed.

The official token action creates a short-lived installation token scoped to the
calling repository and revokes it after the job. The App must not appear in any
branch-protection or ruleset bypass list. Configure required checks and require
the branch to be up to date; the workflow relies on those GitHub protections.
An owner's separate admin bypass can remain enabled.

Alternatively, pass a token explicitly:

```yaml
jobs:
  call:
    uses: twistymaze/renovate-config/.github/workflows/automerge.yml@main
    secrets:
      automerge-token: ${{ secrets.AUTOMERGE_TOKEN }}
```

A PAT acts as its owner, including their bypass privileges. Prefer the App when
owners can bypass required checks. Never put a token or private key in workflow
inputs or source control. Incomplete App credentials, conflicting authentication
methods, or token-generation failures fail the job; only absent credentials use
the default fallback.

The `manual-review` label prevents this workflow from enabling auto-merge. Adding
it does not cancel auto-merge that was already enabled; disable that on the PR
separately. The merge command checks the event's head SHA to avoid acting on a
newer revision from a stale workflow run.

See GitHub's [workflow token behavior](https://docs.github.com/en/actions/concepts/security/github_token)
and [App setup guide](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/registering-a-github-app).
