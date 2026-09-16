# Day 42 - Runners: GitHub-Hosted & Self-Hosted

## Goal

Understand how GitHub Actions jobs are executed using GitHub-hosted runners and self-hosted runners.

---

## Task 1: GitHub-Hosted Runners

Created a workflow with three jobs using different GitHub-hosted runners:

- `ubuntu-latest`
- `windows-latest`
- `macos-latest`

Each job prints:

- OS name
- Runner hostname
- Current user

### Key Learning

A GitHub-hosted runner is a virtual machine provided and managed by GitHub to execute GitHub Actions jobs.

Because the jobs do not depend on each other, GitHub can run them in parallel.

### Example

```yaml
jobs:
  ubuntu:
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "OS: Ubuntu"
          echo "Hostname: $(hostname)"
          echo "User: $(whoami)"

  windows:
    runs-on: windows-latest
    steps:
      - shell: pwsh
        run: |
          Write-Host "OS: Windows"
          Write-Host "Hostname: $env:COMPUTERNAME"
          Write-Host "User: $env:USERNAME"

  macos:
    runs-on: macos-latest
    steps:
      - run: |
          echo "OS: macOS"
          echo "Hostname: $(hostname)"
          echo "User: $(whoami)"
```

---

## Task 2: Explore What's Pre-installed

Checked the following tools on the `ubuntu-latest` runner:

- Docker
- Python
- Node.js
- Git

### Example

```yaml
- name: Check pre-installed tools
  run: |
    echo "Docker:"
    docker --version

    echo "Python:"
    python3 --version

    echo "Node:"
    node --version

    echo "Git:"
    git --version
```

### Why Pre-installed Tools Matter

GitHub-hosted runners already include many commonly used development and DevOps tools.

This means we do not need to install these tools manually before every workflow run, which makes CI/CD workflows faster and easier to set up.

However, pre-installed software versions can change when GitHub updates the runner image. If a project requires a specific version, it is better to explicitly configure that version.

### Documentation

GitHub maintains the runner images and their installed software lists.

The exact software list can also be viewed from the workflow run under the runner image details.

---

## Task 3: Set Up a Self-Hosted Runner

Configured a Linux self-hosted runner on a cloud VM/EC2 instance.

### Key Learning

A self-hosted runner is a machine that we provide and manage ourselves to execute GitHub Actions jobs.

GitHub provides the runner application and connects it to the repository, but we are responsible for the underlying machine.

### Runner Responsibilities

With a self-hosted runner, we are responsible for:

- Machine/VM
- Operating system
- Software and dependencies
- Security
- Updates and patching
- Runner maintenance

The runner appears in GitHub under:

`Settings -> Actions -> Runners`

with an **Idle** status when it is available to execute a job.

---

## Task 4: Use the Self-Hosted Runner

Created:

```text
.github/workflows/self-hosted.yml
```

The workflow uses:

```yaml
runs-on: self-hosted
```

The workflow:

1. Prints the hostname
2. Prints the working directory
3. Creates a test file
4. Verifies that the file exists

### Example

```yaml
name: Self-Hosted Runner

on:
  workflow_dispatch:

jobs:
  self-hosted-job:
    runs-on: self-hosted

    steps:
      - name: Show hostname
        run: |
          echo "Hostname:"
          hostname

      - name: Show working directory
        run: |
          echo "Working directory:"
          pwd

      - name: Create test file
        run: |
          echo "Created by GitHub Actions self-hosted runner" > runner-test.txt
          ls -l runner-test.txt

      - name: Verify file
        run: |
          if [ -f runner-test.txt ]; then
            echo "File exists successfully!"
            cat runner-test.txt
          else
            echo "File was not created."
            exit 1
          fi
```

### Key Learning

The hostname shown in the workflow logs should match the machine running the self-hosted runner.

This proves that the GitHub Actions job is executing on our own VM/machine instead of a GitHub-hosted runner.

---

## Task 5: Labels

Added a custom label:

```text
my-linux-runner
```

Updated the workflow to:

```yaml
runs-on: [self-hosted, my-linux-runner]
```

### Key Learning

GitHub looks for a runner that has **all** the labels specified in `runs-on`.

For example:

```yaml
runs-on: [self-hosted, my-linux-runner]
```

requires a runner with both:

```text
self-hosted
my-linux-runner
```

Labels become useful when an organization has multiple self-hosted runners and needs to target runners with specific operating systems, tools, environments, or capabilities.

### Example

```text
Runner 1
self-hosted
linux
my-linux-runner

Runner 2
self-hosted
linux
docker-runner

Runner 3
self-hosted
linux
production-runner
```

A workflow using:

```yaml
runs-on: [self-hosted, docker-runner]
```

will target a runner matching those labels.

---

## Task 6: GitHub-Hosted vs Self-Hosted

| | GitHub-Hosted | Self-Hosted |
|---|---|---|
| **Who manages it?** | GitHub manages the runner, OS and maintenance | You or your organization manages the machine, OS and maintenance |
| **Cost** | Uses GitHub Actions hosted-runner usage; included usage depends on the GitHub plan | No GitHub-hosted runner charge for execution, but you pay for the underlying infrastructure such as EC2 |
| **Pre-installed tools** | Many commonly used tools are already installed | You decide what to install and maintain |
| **Good for** | Most CI/CD workflows where a standard environment is enough | Custom environments, specific tools, private infrastructure and workloads needing more control |
| **Security concern** | Less control over the underlying runner environment | You are responsible for securing, patching and maintaining the runner |

---

## Important Concept: Self-Hosted Does NOT Mean You Stop Using GitHub Actions

A self-hosted runner is still part of GitHub Actions.

For example:

```yaml
runs-on: [self-hosted, my-linux-runner]
```

GitHub still:

1. Detects the workflow trigger
2. Creates the workflow run
3. Finds an available runner matching the labels
4. Sends the job to the self-hosted runner
5. The self-hosted machine executes the workflow steps
6. Logs and results are sent back to GitHub

The main difference is **who provides and manages the machine**.

### GitHub-hosted

```text
GitHub Repository
       |
       v
GitHub Actions
       |
       v
GitHub-managed Runner
```

### Self-hosted

```text
GitHub Repository
       |
       v
GitHub Actions
       |
       v
Your Self-Hosted Runner
       |
       v
Your EC2 / VM
```

### Cost Understanding

For a self-hosted runner on EC2:

- AWS charges apply for the EC2 instance and any other AWS resources used.
- There is no GitHub-hosted runner-minute charge for the self-hosted runner itself.
- GitHub Actions is still used for workflow orchestration.

---

## Key Takeaways

- A **runner** is the machine that executes a GitHub Actions job.
- `ubuntu-latest`, `windows-latest` and `macos-latest` use GitHub-hosted runners.
- GitHub manages GitHub-hosted runners.
- Self-hosted runners run jobs on infrastructure that we manage.
- `runs-on: self-hosted` targets an available self-hosted runner.
- Labels allow us to target specific self-hosted runners.
- Self-hosted runners still use GitHub Actions.
- Self-hosted runners provide more control, but also more responsibility for security and maintenance.

---

## Screenshots

### Self-Hosted Runner Showing as Idle

<!-- Add screenshot here -->

### Job Running on Self-Hosted Runner

<!-- Add screenshot here -->

### File Created on Self-Hosted Runner

<!-- Add screenshot here -->

---

## Interview Revision

### What is a GitHub Actions runner?

A runner is a machine that executes the steps defined in a GitHub Actions workflow.

### What is the difference between GitHub-hosted and self-hosted runners?

GitHub-hosted runners are provided and managed by GitHub. Self-hosted runners use infrastructure managed by the user or organization.

### Can I use GitHub Actions with a self-hosted runner?

Yes. GitHub Actions still handles the workflow and job orchestration, while the self-hosted machine executes the job.

### Why use labels?

Labels allow jobs to target runners with specific characteristics or capabilities.

### What is the main trade-off?

Self-hosted runners provide more control and customization, but the user or organization is responsible for infrastructure, security, maintenance and updates.
