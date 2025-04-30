<!--
# SPDX-License-Identifier: Apache-2.0
# SPDX-FileCopyrightText: 2025 The Linux Foundation
-->

# 📌 Pinned Versions Action

Verifies action/workflow calls use SHA commit values.

## pinned-versions-action

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

### Pull Requests

When triggered against a pull request, will audit the change content for any
calls to GitHub actions/workflows that are not pinned to a SHA commit value.
This scans files changed in the pull request, and will NOT block merges
where GitHub actions elsewhere in the repository do not use SHA/commit values.

### Manual Invocation

Operates differently when explicitly called using "workflow_dispatch" trigger.
Will scan the entire repository for action/workflow calls and report results
for all GitHub actions/workflows found.
