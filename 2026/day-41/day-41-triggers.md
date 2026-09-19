# Day 41 – GitHub Actions Triggers, Schedules & Matrix Strategies

## Goal

Go beyond basic `push` workflows and understand:

* Pull request triggers
* Scheduled workflows with cron
* Manual workflow triggers with inputs
* Matrix strategies
* Matrix `exclude`
* Matrix `fail-fast`

---

## Task 1 – Pull Request Triggers

A `pull_request` trigger runs a workflow when activity happens on a pull request.

```yaml
name: Pull Request Workflow

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  pr-check:
    runs-on: ubuntu-latest

    steps:
      - name: Show PR details
        run: |
          echo "PR action: ${{ github.event.action }}"
          echo "Source branch: ${{ github.event.pull_request.head.ref }}"
```

### Events

| Event         | Meaning                                  |
| ------------- | ---------------------------------------- |
| `opened`      | A new pull request is created            |
| `synchronize` | New commits are pushed to an existing PR |

### Useful context

```yaml
${{ github.event.pull_request.head.ref }}
```

Returns the source branch of the pull request.

Example:

```text
feature/login → main
```

returns:

```text
feature/login
```

A PR can trigger the workflow multiple times:

```text
Create PR
   ↓
opened

Push another commit
   ↓
synchronize
```

---

## Task 2 – Scheduled Workflows

GitHub Actions supports cron-based scheduled workflows using the `schedule` trigger.

```yaml
on:
  schedule:
    - cron: '0 0 * * *'
```

This runs every day at `00:00 UTC`.

### Cron syntax

```text
minute hour day-of-month month day-of-week
```

### Monday at 9 AM UTC

```text
0 9 * * 1
```

Breakdown:

```text
0   → minute
9   → hour
*   → every day of month
*   → every month
1   → Monday
```

### Key point

Scheduled workflows use **UTC**, so local times must be converted to UTC before creating the cron expression.

---

## Task 3 – Manual Workflow Trigger

`workflow_dispatch` allows a workflow to be started manually from the GitHub Actions UI.

It can also accept inputs:

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Select deployment environment'
        required: true
        type: choice
        options:
          - staging
          - production
```

The selected value can be accessed with:

```yaml
${{ inputs.environment }}
```

Example:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Show environment
        run: |
          echo "Selected environment: ${{ inputs.environment }}"
```

### Why use `workflow_dispatch`?

It is useful when a workflow should be started manually instead of automatically.

```text
GitHub Actions
      ↓
Run workflow
      ↓
Select environment
      ↓
staging / production
```

---

## Task 4 – Matrix Strategy

A matrix lets one job run across multiple combinations of values.

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        python-version: ['3.10', '3.11']

    steps:
      - name: Show configuration
        run: |
          echo "OS: ${{ matrix.os }}"
          echo "Python: ${{ matrix.python-version }}"
```

This creates combinations such as:

```text
ubuntu-latest + Python 3.10
ubuntu-latest + Python 3.11
windows-latest + Python 3.10
windows-latest + Python 3.11
```

Matrix strategies are useful when the same tests need to run across multiple environments or versions.

---

## Task 5 – Matrix `exclude`

A matrix can exclude combinations that are not required.

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    python-version: ['3.10', '3.11']

    exclude:
      - os: windows-latest
        python-version: '3.10'
```

The excluded combination is:

```text
windows-latest + Python 3.10
```

Remaining combinations:

```text
ubuntu-latest + Python 3.10
ubuntu-latest + Python 3.11
windows-latest + Python 3.11
```

### Why use `exclude`?

Use it when a particular combination is unsupported, unnecessary, or not required for testing.

---

## Task 6 – Matrix `fail-fast`

Matrix jobs support:

```yaml
strategy:
  fail-fast: false
```

### `fail-fast: false`

If one matrix job fails, the other matrix jobs continue running.

```yaml
strategy:
  fail-fast: false
  matrix:
    os: [ubuntu-latest, windows-latest]
    python-version: ['3.10', '3.11']
```

### Default behavior

The default is:

```yaml
fail-fast: true
```

With `true`, GitHub can cancel in-progress and queued matrix jobs when a matrix job fails.

### Comparison

| Setting            | Behavior                                           |
| ------------------ | -------------------------------------------------- |
| `fail-fast: true`  | Other matrix work can be cancelled after a failure |
| `fail-fast: false` | Other matrix combinations continue running         |

`fail-fast: false` is useful when you want to see the result of every environment/version combination even if one fails.

---

# Key GitHub Actions Concepts

## `pull_request`

Used when the workflow should respond to pull request activity.

```yaml
on:
  pull_request:
    types: [opened, synchronize]
```

## `schedule`

Used for automatic time-based execution.

```yaml
on:
  schedule:
    - cron: '0 0 * * *'
```

## `workflow_dispatch`

Used to manually start a workflow from GitHub.

It can also accept inputs such as:

```text
staging
production
```

## Matrix

Runs the same job across multiple combinations.

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
```

## `exclude`

Removes specific combinations from a matrix.

```yaml
exclude:
  - os: windows-latest
    python-version: '3.10'
```

## `fail-fast`

Controls whether other matrix jobs should continue when one combination fails.

```yaml
fail-fast: false
```

---

# Interview Revision

### What is `pull_request`?

It triggers a workflow based on pull request activity such as opening a PR or pushing new commits to an existing PR.

### What is `schedule`?

It runs workflows automatically according to a cron expression.

### What is `workflow_dispatch`?

It allows a user to manually trigger a workflow from the GitHub Actions UI and optionally provide inputs.

### What is a matrix strategy?

It runs the same job against multiple combinations of values such as operating systems and language versions.

### Why use `exclude`?

To remove specific combinations from a matrix that should not be tested.

### What does `fail-fast: false` do?

It allows the remaining matrix jobs to continue even if one matrix combination fails.

---

# Day 41 Summary

This day expanded GitHub Actions triggers and job strategies beyond basic `push` workflows.

I practiced:

* `pull_request` events
* `opened` and `synchronize`
* Reading PR information with `github.event`
* Cron-based scheduled workflows
* `workflow_dispatch` with environment inputs
* Matrix strategies
* Matrix `exclude`
* Matrix `fail-fast`

The key idea is that GitHub Actions can be made more flexible by controlling **when workflows run** and **which environments a job runs against**.
