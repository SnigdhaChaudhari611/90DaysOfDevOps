# Day 43 - GitHub Actions Workflow Logic

## Goal

Learn how to control GitHub Actions workflows using:

- Multiple jobs and dependencies
- Environment variables
- GitHub context variables
- Job outputs
- Conditionals
- Parallel jobs
- Passing information between jobs

---

# Task 1: Multi-Job Workflow

## Goal

Create 3 jobs:

- `build`
- `test`
- `deploy`

`test` should run after `build`, and `deploy` should run after `test`.

## Solution

```yaml
name: Multi-Job Workflow

on:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Build the app
        run: echo "Building the app"

  test:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Test the app
        run: echo "Running tests"

  deploy:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - name: Deploy the app
        run: echo "Deploying"
```

## Key Concept

`needs` creates a dependency between jobs.

```yaml
needs: build
```

means the job waits for `build` to complete successfully.

Flow:

```text
build
  ↓
test
  ↓
deploy
```

---

# Task 2: Environment Variables and GitHub Context

## Goal

Use environment variables at three different levels:

- Workflow level
- Job level
- Step level

Also print GitHub context information.

## Solution

```yaml
name: Environment Variables and GitHub Context

on:
  workflow_dispatch:

env:
  APP_NAME: myapp

jobs:
  show-variables:
    runs-on: ubuntu-latest

    env:
      ENVIRONMENT: staging

    steps:
      - name: Print variables
        env:
          VERSION: 1.0.0
        run: |
          echo "App Name: $APP_NAME"
          echo "Environment: $ENVIRONMENT"
          echo "Version: $VERSION"
          echo "Commit SHA: ${{ github.sha }}"
          echo "Triggered by: ${{ github.actor }}"
```

## Variable Scope

| Level | Variable | Available In |
|---|---|---|
| Workflow | `APP_NAME` | Entire workflow |
| Job | `ENVIRONMENT` | All steps in that job |
| Step | `VERSION` | Only that step |

## GitHub Context

```yaml
${{ github.sha }}
```

Returns the commit SHA for the workflow run.

```yaml
${{ github.actor }}
```

Returns the GitHub user/account that triggered the workflow.

## Remember

Environment variable:

```bash
$APP_NAME
```

GitHub context:

```yaml
${{ github.sha }}
```

---

# Task 3: Job Outputs

## Goal

Generate today's date in one job and pass it to another job.

## Solution

```yaml
name: Job Outputs

on:
  workflow_dispatch:

jobs:
  generate-date:
    runs-on: ubuntu-latest

    outputs:
      today: ${{ steps.date.outputs.today }}

    steps:
      - name: Get today's date
        id: date
        run: echo "today=$(date +%Y-%m-%d)" >> "$GITHUB_OUTPUT"

  print-date:
    runs-on: ubuntu-latest
    needs: generate-date

    steps:
      - name: Print date
        run: echo "Today's date is ${{ needs.generate-date.outputs.today }}"
```

## How It Works

### Step 1: Create a Step Output

```bash
echo "today=$(date +%Y-%m-%d)" >> "$GITHUB_OUTPUT"
```

The step has:

```yaml
id: date
```

So the output can be referenced as:

```yaml
steps.date.outputs.today
```

### Step 2: Expose It as a Job Output

```yaml
outputs:
  today: ${{ steps.date.outputs.today }}
```

### Step 3: Read It From Another Job

The second job needs the first:

```yaml
needs: generate-date
```

Then access the output:

```yaml
${{ needs.generate-date.outputs.today }}
```

## Important Pattern

```text
Step output
     ↓
Job output
     ↓
Another job
```

---

# Task 4: Conditionals

## Goal

Use conditions to control when steps and jobs run.

## Solution

```yaml
name: Conditionals

on:
  push:
  pull_request:
  workflow_dispatch:

jobs:
  conditional-steps:
    runs-on: ubuntu-latest

    steps:
      - name: Run on main branch
        if: github.ref == 'refs/heads/main'
        run: echo "This step runs only on main"

      - name: Successful step
        run: echo "This step succeeds"

      - name: Run if previous step failed
        if: failure()
        run: echo "The previous step failed"

      - name: Continue on error
        continue-on-error: true
        run: |
          echo "This step will fail"
          exit 1

      - name: Runs after continue-on-error
        run: echo "The workflow continues"

  push-only-job:
    if: github.event_name == 'push'
    runs-on: ubuntu-latest

    steps:
      - name: Push only
        run: echo "This job runs only for push events"
```

## Important Conditions

### Run only on main

```yaml
if: github.ref == 'refs/heads/main'
```

### Run when a previous step failed

```yaml
if: failure()
```

### Run only for push events

```yaml
if: github.event_name == 'push'
```

### Continue after a failed step

```yaml
continue-on-error: true
```

This allows the step to fail without stopping the job.

---

# Task 5: Putting It Together

## Goal

Create a pipeline where:

1. The workflow triggers on pushes to any branch.
2. `lint` and `test` run independently and can run in parallel.
3. `summary` runs after both.
4. `summary` identifies whether the push was to `main` or another branch.
5. `summary` prints the commit message.

## Solution

`.github/workflows/smart-pipeline.yml`

```yaml
name: Smart Pipeline

on:
  push:

jobs:
  lint:
    runs-on: ubuntu-latest

    steps:
      - name: Run lint
        run: echo "Running lint"

  test:
    runs-on: ubuntu-latest

    steps:
      - name: Run tests
        run: echo "Running tests"

  summary:
    runs-on: ubuntu-latest
    needs: [lint, test]

    steps:
      - name: Branch summary
        run: |
          if [ "${{ github.ref }}" = "refs/heads/main" ]; then
            echo "This is a main branch push"
          else
            echo "This is a feature branch push"
          fi

      - name: Print commit message
        env:
          COMMIT_MESSAGE: ${{ toJSON(github.event.head_commit.message) }}
        run: |
          echo "Commit message:"
          printf '%s\n' "$COMMIT_MESSAGE"
```

## Parallel Jobs

`lint` and `test` do not have a `needs` relationship with each other.

Therefore, they can run independently:

```text
             ┌── lint ──┐
push ────────┤          ├── summary
             └── test ──┘
```

The `summary` job waits for both:

```yaml
needs: [lint, test]
```

## Key Concept

There is no:

```yaml
needs: lint
```

inside `test`, and no:

```yaml
needs: test
```

inside `lint`.

That is why they can run in parallel.

---

# Quick Revision

## Job Dependency

```yaml
needs: build
```

One job waits for another.

## Multiple Dependencies

```yaml
needs: [lint, test]
```

The job waits for both.

## Environment Variable

```yaml
env:
  APP_NAME: myapp
```

Access with:

```bash
echo "$APP_NAME"
```

## GitHub Context

```yaml
${{ github.sha }}
${{ github.actor }}
${{ github.ref }}
${{ github.event_name }}
```

## Job Output

```yaml
outputs:
  value: ${{ steps.step-id.outputs.value }}
```

Read from another job:

```yaml
${{ needs.job-name.outputs.value }}
```

## Conditional Step

```yaml
if: condition
```

## Previous Failure

```yaml
if: failure()
```

## Allow Step Failure

```yaml
continue-on-error: true
```

## Parallel Jobs

Jobs without a dependency on each other can run in parallel.

```text
        ┌── job A ──┐
start ──┤           ├── job C
        └── job B ──┘
```

`job C` uses:

```yaml
needs: [job-A, job-B]
```
