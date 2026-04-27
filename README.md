<!--
# SPDX-License-Identifier: Apache-2.0
# SPDX-FileCopyrightText: 2025 The Linux Foundation
-->

# 📌 Pinned Versions Action

Verifies action/workflow calls use SHA commit values.

## pinned-versions-action

This action wraps [`gha-workflow-linter`][gwl], a Python CLI tool from the
Linux Foundation Release Engineering team. It validates that GitHub Actions
workflow and action calls in the repository pin to immutable commit SHAs,
and that the referenced repositories, tags, branches and commits exist.

The CLI is fetched and run via [`uvx`][uvx] (`astral-sh/setup-uv`), so no
local Python environment setup is required.

[gwl]: https://github.com/lfreleng-actions/gha-workflow-linter
[uvx]: https://docs.astral.sh/uv/guides/tools/

## Recommended Event Triggers

```yaml
on:
    workflow_dispatch:
    pull_request:
        branches:
            - main
            - master
        paths: [".github/**"]
```

## Usage Example

<!-- markdownlint-disable MD013 -->

```yaml
jobs:
  check-actions:
    runs-on: ubuntu-24.04
    permissions:
      contents: read
    steps:
      - name: 'Check Pinned Versions'
        uses: lfreleng-actions/pinned-versions-action@main
```

<!-- markdownlint-enable MD013 -->

## Inputs

<!-- markdownlint-disable MD013 -->

| Variable Name | Required | Default                         | Description                                      |
| ------------- | -------- | ------------------------------- | ------------------------------------------------ |
| path_prefix   | False    | '.' (current working directory) | Directory location containing project code       |
| no_checkout   | False    | false                           | Don't perform a checkout of the local repository |

<!-- markdownlint-enable MD013 -->

## Behaviour

The action:

1. Checks out the repository (or the pull request HEAD).
2. Installs [`uv`][uvx] via `astral-sh/setup-uv`.
3. Invokes `gha-workflow-linter lint` against `path_prefix` using `uvx`,
   which downloads and runs the latest published release on demand.

By default `gha-workflow-linter` enforces SHA pinning for all action and
workflow calls; the action fails if any reference uses a tag, branch, or
unpinned value, or if a referenced repository or commit cannot be resolved.

The action passes `GITHUB_TOKEN` (the workflow's GitHub token) to the
linter so that GraphQL API calls authenticate and avoid rate limits.

### Pull Requests

When triggered against a pull request, the action checks out the PR head
and lints the workflows present in the change. The linter scans the entire
repository's workflow tree; this differs from earlier versions of the
action which only scanned the diff. Repositories that already use SHA
pinning across the tree see no change in behaviour; repositories that do
not should run the linter once with `--auto-fix` locally to migrate.

### Manual Invocation

When invoked via `workflow_dispatch` the action checks out and lints the
default branch state.

## Implementation Details

The previous implementation combined `tj-actions/changed-files` with
`zgosalvez/github-actions-ensure-sha-pinned-actions`. This release
replaces both with `gha-workflow-linter`, which provides SHA pinning
enforcement, repository/reference validation, caching and auto-fix
support in a single tool maintained alongside this action.
