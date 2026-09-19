# Day 47 – Advanced Triggers: PR Events, Cron Schedules & Event-Driven Pipelines

## Goal

Go beyond basic `push` and `pull_request` triggers and understand how GitHub Actions can react to:

- PR lifecycle events
- Scheduled cron jobs
- File and branch changes
- Completion of another workflow
- External events

---

## Task 1 – Pull Request Event Types

### Workflow

`.github/workflows/pr-lifecycle.yml`

```yaml
name: PR Lifecycle

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

jobs:
  pr-info:
    runs-on: ubuntu-latest

    steps:
      - name: Show PR Details
        run: |
          echo "Event type: ${{ github.event.action }}"
          echo "PR title: ${{ github.event.pull_request.title }}"
          echo "PR author: ${{ github.event.pull_request.user.login }}"
          echo "Source branch: ${{ github.event.pull_request.head.ref }}"
          echo "Target branch: ${{ github.event.pull_request.base.ref }}"

      - name: PR Merged
        if: github.event.action == 'closed' && github.event.pull_request.merged == true
        run: |
          echo "PR was merged successfully!"
          echo "Merged PR: ${{ github.event.pull_request.title }}"
```

### Key points

| Event | Meaning |
|---|---|
| `opened` | A new PR is created |
| `synchronize` | New commits are pushed to an existing PR |
| `reopened` | A closed PR is reopened |
| `closed` | A PR is closed or merged |

A merged PR generates the `closed` event, so the workflow checks both:

```yaml
github.event.action == 'closed'
```

and:

```yaml
github.event.pull_request.merged == true
```

### Tested

- Created a PR → `opened`
- Pushed another commit to the PR → `synchronize`
- Merged the PR → `closed` + merged condition

---

# Task 2 – PR Validation Workflow

### Workflow

`.github/workflows/pr-checks.yml`

```yaml
name: PR Checks

on:
  pull_request:
    branches:
      - main

jobs:
  file-size-check:
    name: File Size Check
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Check file sizes
        run: |
          large_files=$(find . -type f -not -path './.git/*' -size +1M)

          if [ -n "$large_files" ]; then
            echo "Files larger than 1 MB found:"
            echo "$large_files"
            exit 1
          else
            echo "All files are within the 1 MB limit."
          fi

  branch-name-check:
    name: Branch Name Check
    runs-on: ubuntu-latest

    steps:
      - name: Validate branch name
        run: |
          branch="${{ github.head_ref }}"

          if [[ "$branch" == feature/* || "$branch" == fix/* || "$branch" == docs/* ]]; then
            echo "Valid branch name: $branch"
          else
            echo "Invalid branch name: $branch"
            echo "Branch must start with feature/, fix/, or docs/"
            exit 1
          fi

  pr-body-check:
    name: PR Body Check
    runs-on: ubuntu-latest

    steps:
      - name: Check PR description
        run: |
          if [ -z "${{ github.event.pull_request.body }}" ]; then
            echo "::warning::PR description is empty."
          else
            echo "PR description is present."
          fi
```

### Branch naming rule

Valid:

```text
feature/add-login
fix/login-bug
docs/update-readme
```

Invalid:

```text
test-branch
dev
my-feature
```

### PR body rule

An empty PR description generates a warning but does not fail the job.

### Tested

Opened a PR from a badly named branch and verified that `branch-name-check` failed.

---

# Task 3 – Scheduled Workflows

### Workflow

`.github/workflows/scheduled-tasks.yml`

```yaml
name: Scheduled Tasks

on:
  schedule:
    - cron: '30 2 * * 1'
    - cron: '0 */6 * * *'
  workflow_dispatch:

jobs:
  scheduled-task:
    runs-on: ubuntu-latest

    steps:
      - name: Show triggered schedule
        run: |
          echo "Triggered by schedule: ${{ github.event.schedule }}"

      - name: Health check
        run: |
          response=$(curl -s -o /dev/null -w "%{http_code}" https://example.com)

          echo "HTTP response code: $response"

          if [ "$response" -ne 200 ]; then
            echo "Health check failed!"
            exit 1
          fi

          echo "Health check passed!"
```

### Cron syntax

```text
minute hour day-of-month month day-of-week
```

### Required expressions

**Every Monday at 2:30 AM UTC**

```text
30 2 * * 1
```

**Every 6 hours**

```text
0 */6 * * *
```

**Every weekday at 9 AM IST**

IST is UTC+5:30, so 9:00 AM IST = 3:30 AM UTC.

```text
30 3 * * 1-5
```

**First day of every month at midnight UTC**

```text
0 0 1 * *
```

### Why scheduled workflows can be delayed or disabled

Scheduled workflows can be delayed during periods of high GitHub load, especially around the start of an hour. GitHub also automatically disables scheduled workflows in repositories with no activity for 60 days.

Scheduled workflows run only from the repository's default branch.

### Manual testing

`workflow_dispatch` was added so the workflow could be run immediately from:

**GitHub → Actions → Scheduled Tasks → Run workflow**

---

# Task 4 – Path & Branch Filters

## Workflow 1: `paths`

`.github/workflows/smart-triggers.yml`

```yaml
name: Smart Triggers

on:
  push:
    branches:
      - main
      - 'release/*'
    paths:
      - 'src/**'
      - 'app/**'

jobs:
  app-change:
    runs-on: ubuntu-latest

    steps:
      - name: Show trigger
        run: echo "Workflow triggered by a change in src/ or app/"
```

This requires both:

- Branch = `main` or `release/*`
- Changed files = `src/**` or `app/**`

## Workflow 2: `paths-ignore`

`.github/workflows/docs-filter.yml`

```yaml
name: Docs Filter

on:
  push:
    branches:
      - main
      - 'release/*'
    paths-ignore:
      - '*.md'
      - 'docs/**'

jobs:
  code-change:
    runs-on: ubuntu-latest

    steps:
      - name: Show trigger
        run: echo "Workflow triggered because a non-documentation file changed."
```

### `paths` vs `paths-ignore`

**Use `paths`** when a workflow should run only for specific paths.

```yaml
paths:
  - 'src/**'
  - 'app/**'
```

**Use `paths-ignore`** when a workflow should run for most changes but skip changes limited to specific paths.

```yaml
paths-ignore:
  - '*.md'
  - 'docs/**'
```

### Tested

A Markdown-only change was used to verify that the path-filtered workflow was skipped.

---

# Task 5 – `workflow_run` – Chain Workflows

## Workflow 1: Tests

`.github/workflows/tests.yml`

```yaml
name: Run Tests

on:
  push:

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run tests
        run: |
          echo "Running tests..."
          echo "All tests passed!"
```

## Workflow 2: Deploy after tests

`.github/workflows/deploy-after-tests.yml`

```yaml
name: Deploy After Tests

on:
  workflow_run:
    workflows: ["Run Tests"]
    types: [completed]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Check test result
        if: github.event.workflow_run.conclusion != 'success'
        run: |
          echo "::warning::Tests did not pass. Deployment skipped."
          exit 1

      - name: Deploy
        if: github.event.workflow_run.conclusion == 'success'
        run: |
          echo "Tests passed successfully!"
          echo "Starting deployment..."
```

### Flow

```text
git push
   ↓
Run Tests
   ↓
workflow completes
   ↓
Deploy After Tests
   ↓
Check conclusion
   ↓
success → deploy
failure → warning + stop
```

### Tested

A commit was pushed and the test workflow ran first. After it completed successfully, the deploy workflow was triggered.

---

# Task 6 – `repository_dispatch` – External Event Triggers

### Workflow

`.github/workflows/external-trigger.yml`

```yaml
name: External Trigger

on:
  repository_dispatch:
    types: [deploy-request]

jobs:
  external-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Show deployment request
        run: |
          echo "External deployment request received."
          echo "Environment: ${{ github.event.client_payload.environment }}"
```

### Trigger with GitHub CLI

```bash
gh api repos/<owner>/<repo>/dispatches \
  -f event_type=deploy-request \
  -f client_payload='{"environment":"production"}'
```

Example:

```bash
gh api repos/CrystallyRains/90DaysOfDevOps/dispatches \
  -f event_type=deploy-request \
  -f client_payload='{"environment":"production"}'
```

The workflow receives the value through:

```yaml
${{ github.event.client_payload.environment }}
```

Expected output:

```text
External deployment request received.
Environment: production
```

### When would an external system trigger a pipeline?

An external system can use `repository_dispatch` when an event outside GitHub needs to start a GitHub Actions workflow.

Examples:

- Slack bot receives a `/deploy` command
- Monitoring system detects an event that requires remediation
- External deployment platform requests a deployment
- Another application needs to start a GitHub workflow

The external system can also send data through `client_payload`, such as the target environment.

---

# `workflow_run` vs `workflow_call`

| `workflow_run` | `workflow_call` |
|---|---|
| Starts after another workflow runs | Lets one workflow call another workflow |
| Triggered by workflow completion | Explicitly invoked by another workflow |
| Can inspect the previous workflow's conclusion | Can receive inputs and secrets |
| Useful for chaining independent workflows | Useful for reusable workflows |
| Example: tests → deploy | Example: multiple pipelines → shared deployment workflow |

### In simple words

**`workflow_run`** is useful when I want GitHub to react after another workflow finishes.

```text
Tests finish → trigger deployment workflow
```

**`workflow_call`** is useful when I want to build a reusable workflow that other workflows can call.

```text
Workflow A ──┐
Workflow B ──┼──→ Reusable workflow
Workflow C ──┘
```

---

# Advanced Trigger Cheat Sheet

| Trigger | Use case |
|---|---|
| `pull_request` | React to PR activity |
| `schedule` | Run jobs on a cron schedule |
| `workflow_dispatch` | Run a workflow manually |
| `push` + `paths` | Run only for specific files |
| `push` + `paths-ignore` | Skip specific files |
| `workflow_run` | React after another workflow completes |
| `repository_dispatch` | Receive events from external systems |

---

# Key Interview Points

### `github.event.action`

Identifies the specific activity that triggered a PR workflow.

Example:

```yaml
${{ github.event.action }}
```

Possible values in this task:

```text
opened
synchronize
reopened
closed
```

### `github.head_ref`

For a pull request, this gives the source branch.

### `github.event.pull_request.base.ref`

Gives the target branch.

### `github.event.pull_request.merged`

Indicates whether a closed PR was actually merged.

### `github.event.schedule`

Identifies which cron expression triggered a scheduled workflow.

### `github.event.workflow_run.conclusion`

Provides the result of the workflow that triggered a `workflow_run` workflow.

### `github.event.client_payload`

Contains custom data sent through a `repository_dispatch` event.

---

# Day 47 Summary

This day covered how GitHub Actions can move beyond simple push-based automation.

I practiced:

- PR lifecycle event filtering
- PR validation gates
- Branch and path filters
- Cron-based scheduled workflows
- Manual workflow execution
- Workflow chaining with `workflow_run`
- External event triggers with `repository_dispatch`
- Passing data through event payloads
- Comparing `workflow_run` and `workflow_call`

The main idea is that GitHub Actions workflows don't have to run only when code is pushed. They can react to **PR events, schedules, other workflows, file changes, branches, and external systems**.
