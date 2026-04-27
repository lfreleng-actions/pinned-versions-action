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

The action installs and runs the CLI via [`uvx`][uvx]
(`astral-sh/setup-uv`), so callers need no local Python environment.

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
      # pull-requests: read lets the metadata step list PR-changed
      # files on pull_request events. Safe to grant unconditionally;
      # ignored on other events.
      pull-requests: read
    steps:
      - name: 'Check Pinned Versions'
        # yamllint disable-line rule:line-length
        uses: lfreleng-actions/pinned-versions-action@38f4a2758d6314213f744dba3f0d68335e5562bf # v0.1.2
```

<!-- markdownlint-enable MD013 -->

### Required permissions

The action needs the following permissions on `GITHUB_TOKEN`:

- `contents: read` — to check out the repository.
- `pull-requests: read` — needed on `pull_request` events so the
  embedded `repository-metadata-action` can list the files that the
  pull request changes. Without this permission, PR-scoped runs fail
  in the metadata step.

## Inputs

<!-- markdownlint-disable MD013 -->

| Variable Name  | Required | Default                         | Description                                          |
| -------------- | -------- | ------------------------------- | ---------------------------------------------------- |
| path_prefix    | False    | '.' (current working directory) | Directory location containing project code           |
| no_checkout    | False    | false                           | Don't perform a checkout of the local repository     |
| full_scan      | False    | false                           | Force a full repository scan, even on `pull_request` |
| linter_version | False    | (latest)                        | Pin `gha-workflow-linter` to a specific PyPI version |

<!-- markdownlint-enable MD013 -->

### Pinning the linter version

By default the action runs the **latest** released `gha-workflow-linter`
from PyPI on each invocation. To pin a specific version (and avoid
non-deterministic behaviour across runs), set `linter_version`:

<!-- markdownlint-disable MD013 -->

```yaml
- name: 'Check Pinned Versions'
  # yamllint disable-line rule:line-length
  uses: lfreleng-actions/pinned-versions-action@38f4a2758d6314213f744dba3f0d68335e5562bf # v0.1.2
  with:
    linter_version: '1.0.2'
```

<!-- markdownlint-enable MD013 -->

When set, the action invokes
`uvx --from gha-workflow-linter==<version> ...`.

## Behaviour

### Pull Requests

When triggered against a `pull_request` event, the action:

1. Checks out the PR head (default shallow checkout).
2. Calls [`repository-metadata-action`][rma] to fetch the list of files
   that the pull request changes. The action sets all summary outputs
   and artifact uploads from that action to `false` (`github_summary`,
   `gerrit_summary`, `files_summary`, and `artifact_upload`) so that
   workflows already invoking that action elsewhere do not see duplicate
   `GITHUB_STEP_SUMMARY` content.
3. Restricts the changed-files list to workflow / composite-action
   definitions: any `*.yml` or `*.yaml` under
   `<path_prefix>/.github/workflows/`, plus `action.yml` / `action.yaml`
   files under `<path_prefix>/.github/actions/`. Other YAML under
   `.github/` (for example `dependabot.yml` or `actionlint.yaml`) falls
   outside this scope and the action ignores it.
4. Invokes `gha-workflow-linter lint` with one `--files` argument per
   matched file, validating that subset of files.

If the change touches no workflow or composite-action YAML, the action
skips the linter step and exits with success. Set the `full_scan` input
to `true` to force a full-tree scan even on pull requests.

[rma]: https://github.com/lfreleng-actions/repository-metadata-action

### Manual / push / scheduled invocation

For any non-`pull_request` event (or when `full_scan: 'true'`), the
action runs `gha-workflow-linter lint <path_prefix>` against the entire
workflow tree.

### Auto-fix

The action **disables** the linter's auto-fix mode (`--no-auto-fix`).
Ephemeral CI runners throw away any rewrites the linter makes, which
hides validation errors from the workflow log. Run `gha-workflow-linter
lint --auto-fix` locally to migrate a repository, then commit the
result.

The action passes `GITHUB_TOKEN` (the workflow's GitHub token) to the
linter so that GraphQL API calls authenticate and avoid rate limits.

## Implementation Details

The previous implementation combined `tj-actions/changed-files` with
`zgosalvez/github-actions-ensure-sha-pinned-actions`. This release
replaces both with `gha-workflow-linter` (and uses our own
`repository-metadata-action` to scope pull-request runs), providing
SHA pinning enforcement, repository/reference validation, caching and
auto-fix support in a single tool maintained alongside this action.
