# ready-when-green

Marks a labelled draft pull request ready for review once a named check has passed on its head commit. It is [`merge-when`](https://github.com/ExaDev/merge-when) with `then: ready` and the conditions fixed to "labelled and the named check green"; use `merge-when` directly to choose other conditions.

## When to use this

Open pull requests as drafts while CI is still deciding, and let them become reviewable, and mergeable, only once it is green. That suits agents and bots that open a pull request as soon as a branch exists, and anyone who would rather reviewers and merge automation never see a red pull request. Run it beside [`merge-when-green`](https://github.com/ExaDev/merge-when-green), which leaves drafts alone: this action marks the draft ready, the `ready_for_review` event that causes starts `merge-when-green` again, and it merges.

A pull request is marked ready when all of these hold: it carries the opt-in label, it is a draft, it still has the commit being checked at its head, and the named check run succeeded on that commit. One that is already ready is left alone.

## Use

The action has no triggers of its own, so a workflow supplies them. Run it when CI finishes, in a workflow of its own beside the one that runs `merge-when-green`:

```yaml
name: Ready when green

on:
  workflow_run:
    workflows: [CI]
    types: [completed]

concurrency:
  group: ready-when-green-${{ github.event.workflow_run.head_sha }}
  cancel-in-progress: false

permissions:
  contents: read
  pull-requests: read
  checks: read

jobs:
  ready:
    if: github.event.workflow_run.event == 'pull_request' && github.event.workflow_run.conclusion == 'success'
    runs-on: ubuntu-latest
    steps:
      - uses: ExaDev/ready-when-green@v1
        with:
          merge-token: ${{ secrets.MERGE_TOKEN }}
          required-check: Required checks
```

The `merge-when-green` workflow needs `ready_for_review` among its `pull_request_target` types, as its own example already has, so that a pull request this action marks ready is merged at once:

```yaml
on:
  workflow_run:
    workflows: [CI]
    types: [completed]
  pull_request_target:
    types: [labeled, ready_for_review]
```

Three things make the pair work:

- **CI has to run on drafts.** The required check must report on a draft's head commit, so don't skip CI jobs with `if: github.event.pull_request.draft == false` or start CI only on `ready_for_review`; GitHub's default `pull_request` types already include drafts.
- **The token must not be `GITHUB_TOKEN`.** GitHub starts no workflow from an event that token caused, so a pull request it marked ready would never start the merge. Give `merge-token`, or `app-id` and `app-private-key`; the action refuses to run without one.
- **The same label opts in to both.** With the default `automerge`, a draft labelled once is readied and then merged. Give a different `label` to have pull requests readied without being merged.

Marking a pull request ready is not pinned to a commit the way the merge is, so a push landing between the check and the call is readied with it. A pull request readied wrongly goes back to draft with `gh pr ready <number> --undo`. Do not add a checkout step to a workflow that runs on `pull_request_target`.

## Inputs

| Input                       | Default             | Meaning                                                                                                                                                                                          |
| --------------------------- | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `required-check`            | required            | Name of the check run that must have succeeded on the head commit. Point it at one aggregate job that `needs` every other job and runs with `if: always()`, so it passes only when they all did. |
| `label`                     | `automerge`         | Label that opts a pull request in.                                                                                                                                                               |
| `merge-token`               |                     | An API token (a personal access token or an app token) that marks the pull request ready. It needs write on pull requests.                                                                       |
| `app-id`, `app-private-key` |                     | A GitHub App to mint an installation token from, instead of `merge-token`. The app needs write on pull requests.                                                                                 |
| `read-token`                | `github.token`      | Token used to read pull requests and checks.                                                                                                                                                     |
| `head-sha`                  | event's head commit | Commit to act on.                                                                                                                                                                                |

## Development

This repository readies its own labelled drafts with the action (`.github/workflows/ready-when-green.yml`) and merges them with `merge-when-green` beside it (`.github/workflows/merge-when-green.yml`), both with the organisation's GitHub App. The work itself is done by `merge-when`, which holds the script and its tests; `npm test` here runs the unit tests for the release plugins. CI also runs the composite action from the checkout, and commitlint, actionlint, typecheck, lint and format checks, aggregated into one `Required Checks` job.

Releases are made by semantic-release from conventional commits on `main`: it tags the version, publishes the GitHub Release, updates `CHANGELOG.md` and moves the major tag (`v1`) that consumers pin to.
