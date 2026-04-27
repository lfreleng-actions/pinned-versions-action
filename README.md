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
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - name: 'Check Pinned Versions'
        uses: lfreleng-actions/pinned-versions-action@main
```

<!-- markdownlint-enable MD013 -->

## Inputs

<!-- markdownlint-disable MD013 -->

| Variable Name | Required | Default                         | Description                                              |
| ------------- | -------- | ------------------------------- | -------------------------------------------------------- |
| path_prefix   | False    | '.' (current working directory) | Directory location containing project code               |
| no_checkout   | False    | false                           | Don't perform a checkout of the local repository         |
| full_scan     | False    | false                           | Force a full repository scan, even on `pull_request`     |
| linter_version| False    | (latest)                        | Pin `gha-workflow-linter` to a specific PyPI version     |

<!-- markdownlint-enable MD013 -->

### Pinning the linter version

By default the action runs the **latest** released `gha-workflow-linter`
from PyPI on each invocation. To pin a specific version (and avoid
non-deterministic behaviour across runs), set `linter_version`:

```yaml
- name: 'Check Pinned Versions'
  uses: lfreleng-actions/pinned-versions-action@main
  with:
    linter_version: '1.0.2'
```

When set, the action invokes
`uvx --from gha-workflow-linter==<version> ...`.

## Behaviour

### Pull Requests

When triggered against a `pull_request` event, the action:

1. Checks out the PR head (with full history).
2. Calls [`repository-metadata-action`][rma] to obtain the list of files
   changed in the pull request. All summary outputs and artifact uploads
   from that action are suppressed (`github_summary`, `gerrit_summary`,
   `files_summary`, and `artifact_upload` all set to `false`) so that
   workflows already invoking that action elsewhere do not see duplicated
   `GITHUB_STEP_SUMMARY` content.
3. Filters the changed-files list to YAML files under
   `<path_prefix>/.github/` (workflows under `.github/workflows/` and
   composite-action `action.yaml` files under `.github/actions/`).
4. Invokes `gha-workflow-linter lint` with one `--files` argument per
   matched file, validating only those.

If the change does not touch any workflow or composite-action YAML, the
linter step is skipped and the action exits successfully. Set the
`full_scan` input to `true` to force a full-tree scan even on pull
requests.

[rma]: https://github.com/lfreleng-actions/repository-metadata-action

### Manual / push / scheduled invocation

For any non-`pull_request` event (or when `full_scan: 'true'`), the
action runs `gha-workflow-linter lint <path_prefix>` against the entire
workflow tree.

### Auto-fix

The linter's auto-fix mode is **disabled** when run from this action
(`--no-auto-fix`). Rewrites in an ephemeral CI runner are discarded and
silently mask validation errors. Run `gha-workflow-linter lint
--auto-fix` locally to migrate a repository, then commit the result.

The action passes `GITHUB_TOKEN` (the workflow's GitHub token) to the
linter so that GraphQL API calls authenticate and avoid rate limits.

## Implementation Details

The previous implementation combined `tj-actions/changed-files` with
`zgosalvez/github-actions-ensure-sha-pinned-actions`. This release
replaces both with `gha-workflow-linter` (and uses our own
`repository-metadata-action` to scope pull-request runs), providing
SHA pinning enforcement, repository/reference validation, caching and
auto-fix support in a single tool maintained alongside this action.
