# Day 46 - Reusable Workflows & Composite Actions

## Goal

Learn how to reuse GitHub Actions automation using:

- Reusable workflows with `workflow_call`
- Workflow inputs, secrets, and outputs
- Composite actions
- The difference between reusable workflows and composite actions

---

## Task 1: Understand `workflow_call`

### 1. What is a reusable workflow?

A **reusable workflow** is a GitHub Actions workflow designed to be called by another workflow.

It allows us to write common CI/CD logic once and reuse it instead of duplicating the same YAML across workflows.

Benefits:

- Avoids duplicate workflow code
- Standardizes CI/CD processes
- Makes workflows easier to maintain
- Can contain complete jobs and multiple steps

### 2. What is the `workflow_call` trigger?

`workflow_call` makes a workflow callable by another workflow.

```yaml
on:
  workflow_call:
```

A reusable workflow can define:

- Inputs
- Secrets
- Outputs

### 3. Reusable workflow vs regular action

A regular action is used inside a job's `steps`:

```yaml
steps:
  - uses: actions/checkout@v4
```

A reusable workflow is called at the **job level**:

```yaml
jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml
```

Easy way to remember:

```text
Action
→ Used inside steps
→ Performs a specific task

Reusable Workflow
→ Used at the job level
→ Can contain complete jobs and multiple steps
```

### 4. Where does a reusable workflow live?

Reusable workflows must live directly inside:

```text
.github/workflows/
```

Example:

```text
.github/
└── workflows/
    ├── call-workflow.yml
    └── reusable-build.yml
```

---

# Task 2: Create Your First Reusable Workflow

File:

```text
.github/workflows/reusable-build.yml
```

```yaml
name: Reusable Build

on:
  workflow_call:
    inputs:
      app_name:
        description: "Name of the application"
        required: true
        type: string

      environment:
        description: "Deployment environment"
        required: true
        type: string
        default: staging

    secrets:
      docker_token:
        required: true

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build application
        run: |
          echo "Building ${{ inputs.app_name }} for ${{ inputs.environment }}"

      - name: Check Docker token
        run: |
          if [ -n "${{ secrets.docker_token }}" ]; then
            echo "Docker token is set: true"
          else
            echo "Docker token is set: false"
          fi
```

### Important

A workflow using only:

```yaml
on:
  workflow_call:
```

does **not** get a **Run workflow** button in GitHub Actions.

It needs a caller workflow.

The caller supplies the input values:

```yaml
with:
  app_name: my-app
  environment: staging
```

The reusable workflow defines the inputs, while the caller provides the actual values.

---

# Task 3: Call the Reusable Workflow

Example caller:

```yaml
name: Call Reusable Build

on:
  workflow_dispatch:

jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml

    with:
      app_name: my-app
      environment: staging

    secrets:
      docker_token: ${{ secrets.DOCKER_TOKEN }}
```

The caller has `workflow_dispatch`, so **this workflow** gets the **Run workflow** button.

The reusable workflow itself does not.

### Input flow

```text
reusable-build.yml
        ↓
Defines inputs
        ↓
app_name
environment
        ↓
call-workflow.yml
        ↓
Supplies values
        ↓
my-app
staging
```

If inputs need to be selected from the GitHub UI, the caller can define `workflow_dispatch.inputs` and pass those values into the reusable workflow.

---

# Task 4: Add Outputs to the Reusable Workflow

The reusable workflow can generate an output and expose it to the caller.

## Updated `reusable-build.yml`

```yaml
name: Reusable Build

on:
  workflow_call:
    inputs:
      app_name:
        description: "Name of the application"
        required: true
        type: string

      environment:
        description: "Deployment environment"
        required: true
        type: string
        default: staging

    secrets:
      docker_token:
        required: true

    outputs:
      build_version:
        description: "Generated build version"
        value: ${{ jobs.build.outputs.build_version }}

jobs:
  build:
    runs-on: ubuntu-latest

    outputs:
      build_version: ${{ steps.version.outputs.build_version }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Generate build version
        id: version
        run: |
          SHORT_SHA=$(git rev-parse --short HEAD)
          VERSION="v1.0-${SHORT_SHA}"
          echo "build_version=$VERSION" >> "$GITHUB_OUTPUT"
          echo "Generated version: $VERSION"

      - name: Build application
        run: |
          echo "Building ${{ inputs.app_name }} for ${{ inputs.environment }}"

      - name: Check Docker token
        run: |
          if [ -n "${{ secrets.docker_token }}" ]; then
            echo "Docker token is set: true"
          else
            echo "Docker token is set: false"
          fi
```

## Output flow

```text
Step output
    ↓
steps.version.outputs.build_version
    ↓
Job output
    ↓
jobs.build.outputs.build_version
    ↓
Reusable workflow output
    ↓
workflow_call.outputs.build_version
    ↓
Caller workflow
    ↓
needs.build.outputs.build_version
```

## Add a second job to the caller

```yaml
show-version:
  needs: build
  runs-on: ubuntu-latest

  steps:
    - name: Print build version
      run: |
        echo "Build version: ${{ needs.build.outputs.build_version }}"
```

`needs: build` means the second job waits for the `build` job to finish.

The output is accessed with:

```yaml
${{ needs.build.outputs.build_version }}
```

### Expected result

If the short commit SHA is:

```text
a1b2c3d
```

the reusable workflow generates:

```text
v1.0-a1b2c3d
```

The second job should print:

```text
Build version: v1.0-a1b2c3d
```

---

# Task 5: Create a Composite Action

A **composite action** packages multiple workflow steps into a reusable action.

Create:

```text
.github/actions/setup-and-greet/action.yml
```

## `action.yml`

```yaml
name: Setup and Greet
description: Greet the user and display runner information

inputs:
  name:
    description: "Name to greet"
    required: true

  language:
    description: "Greeting language"
    required: false
    default: en

outputs:
  greeted:
    description: "Whether the greeting was completed"
    value: ${{ steps.greet.outputs.greeted }}

runs:
  using: composite

  steps:
    - name: Print greeting
      id: greet
      shell: bash
      run: |
        case "${{ inputs.language }}" in
          en)
            echo "Hello, ${{ inputs.name }}!"
            ;;
          hi)
            echo "Namaste, ${{ inputs.name }}!"
            ;;
          fr)
            echo "Bonjour, ${{ inputs.name }}!"
            ;;
          es)
            echo "Hola, ${{ inputs.name }}!"
            ;;
          *)
            echo "Hello, ${{ inputs.name }}!"
            ;;
        esac

        echo "greeted=true" >> "$GITHUB_OUTPUT"

    - name: Print runner information
      shell: bash
      run: |
        echo "Current date: $(date)"
        echo "Runner OS: $RUNNER_OS"
```

## Test the composite action

Create:

```text
.github/workflows/test-composite-action.yml
```

```yaml
name: Test Composite Action

on:
  workflow_dispatch:

jobs:
  greet:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Run custom greeting action
        id: greeting
        uses: ./.github/actions/setup-and-greet
        with:
          name: Snigdha
          language: en

      - name: Check output
        run: |
          echo "Greeted: ${{ steps.greeting.outputs.greeted }}"
```

### Expected result

The workflow should print something similar to:

```text
Hello, Snigdha!
Current date: ...
Runner OS: Linux
Greeted: true
```

The composite action itself is not run independently. A workflow runs it using:

```yaml
uses: ./.github/actions/setup-and-greet
```

---

# Task 6: Reusable Workflow vs Composite Action

| | **Reusable Workflow** | **Composite Action** |
|---|---|---|
| **Triggered by** | `workflow_call` | `uses:` in a step |
| **Can contain jobs?** | Yes | No |
| **Can contain multiple steps?** | Yes, across jobs | Yes, within the action |
| **Lives where?** | `.github/workflows/` | `.github/actions/<action-name>/action.yml` |
| **Can accept secrets directly?** | Yes, through `workflow_call.secrets` | No dedicated `secrets` interface |
| **Best for** | Reusing entire CI/CD workflows and jobs | Reusing a group of steps as a single action |

## Key difference

```text
Reusable Workflow
        ↓
Can contain JOBS
        ↓
Jobs contain STEPS
        ↓
Good for reusing an entire workflow pattern
```

```text
Composite Action
        ↓
Cannot contain JOBS
        ↓
Contains STEPS
        ↓
Good for packaging repeated steps
```

### Interview memory trick

> **Reusable workflow = reuse jobs.**  
> **Composite action = reuse steps.**

### File locations

```text
Reusable workflow
→ .github/workflows/

Composite action
→ .github/actions/
```

---

# Day 46 Quick Revision

```text
workflow_call
→ Makes a workflow reusable

workflow_dispatch
→ Allows manual "Run workflow" execution from the UI

Reusable workflow
→ Called at job level
→ Can contain multiple jobs
→ Supports workflow inputs, secrets and outputs
→ Lives in .github/workflows/

Composite action
→ Called inside steps
→ Groups multiple steps into one reusable action
→ Cannot contain jobs
→ Lives in .github/actions/<action-name>/action.yml

Outputs
→ Step output
→ Job output
→ Reusable workflow output
→ Caller reads it with needs.<job>.outputs.<output>
```
