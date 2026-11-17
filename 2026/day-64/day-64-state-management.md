# Day 64 -- Terraform State Management and Remote Backends

## Goal

Understand how Terraform tracks infrastructure, stores state remotely, prevents concurrent state changes, imports existing resources, performs state operations, and detects infrastructure drift.

This document records the Day 64 hands-on work and explains the concepts in an interview-ready way.

---

## 1. Terraform state: what it is and why it matters

Terraform uses a state file to keep track of the relationship between resource addresses in the Terraform configuration and the real infrastructure those resources represent.

For example:

```hcl
resource "aws_instance" "web" {
  # EC2 configuration
}
```

Terraform tracks this resource using the address:

```text
aws_instance.web
```

The state records information Terraform needs about that resource, including its real AWS ID and many attributes returned by AWS.

### Configuration, state, and real infrastructure are different

| Item | What it represents |
|---|---|
| `.tf` files | Desired configuration: what Terraform should build or maintain |
| Terraform state | Terraform's recorded mapping and information about managed resources |
| AWS infrastructure | What actually exists in AWS right now |

A normal `terraform plan` refreshes information from the provider and compares the desired configuration with the real infrastructure as represented through state.

**Interview answer:** Terraform state maps configuration resource addresses to real infrastructure objects. Terraform uses it to understand what it manages and to plan changes safely. It is not the desired configuration itself; the `.tf` files describe the desired configuration.

### Commands used to inspect state

```bash
terraform show
terraform state list
terraform state show aws_instance.web
terraform state show aws_vpc.main
```

- `terraform show`: displays the state in a human-readable format.
- `terraform state list`: lists resource addresses tracked in state.
- `terraform state show`: displays the recorded attributes of one resource.

### What did the state contain?

Before importing the additional S3 bucket, the configuration tracked:

- 8 managed resources
- 2 data sources

The 8 managed resources were:

```text
aws_instance.web
aws_internet_gateway.igw
aws_route_table.public
aws_route_table_association.public
aws_s3_bucket.app_logs
aws_security_group.web
aws_subnet.public
aws_vpc.main
```

The two data sources were:

```text
data.aws_ami.amazon_linux_2
data.aws_availability_zones.available
```

After importing the additional S3 bucket, the managed-resource count became 9. Data sources are read-only lookups, not resources Terraform creates and manages in the same way.

### What attributes does state store?

For an EC2 instance, state can contain much more than the few arguments written in the resource block. Examples include:

- EC2 instance ID and ARN
- Instance type and current state
- Availability zone
- Private and public IP addresses, when available
- Network interface details
- Security group information
- CPU options
- Tags
- EBS volume details

The provider returns many of these values during refresh.

### What is the `serial` field?

The `serial` field in the state JSON is a monotonically increasing number for state snapshots. It helps Terraform distinguish newer state snapshots from older ones and protect against accidentally overwriting newer state with an older snapshot.

It is not the number of resources and it is not the Terraform version.

---

## 2. Remote backend: storing state in S3

### Why move state out of the local project?

With local state, the state file is stored on one machine. This creates risks:

- Other team members may not have the latest state.
- A lost or damaged local file can cause serious management problems.
- Multiple people working independently can create coordination problems.
- State may contain sensitive infrastructure information.

A remote backend stores state in a shared remote location so authorised team members and automation can use the same state.

### Local state versus remote state

```text
LOCAL BACKEND

Engineer / local machine
        |
        v
Terraform configuration
        |
        v
terraform.tfstate on local disk


REMOTE S3 BACKEND

Engineer / CI pipeline
        |
        v
Terraform configuration
        |
        v
S3 backend
  |
  +-- State object: dev/terraform.tfstate
  |
  +-- Versioning: previous object versions
  |
  +-- Locking: prevents conflicting operations
```

### Backend resources created

| Setting | Value |
|---|---|
| S3 bucket | `terraweek-state-snigdha` |
| State object key | `dev/terraform.tfstate` |
| Backend region | `ap-south-1` |
| DynamoDB lock table | `terraweek-state-lock` |
| S3 versioning | Enabled |
| State encryption setting | Enabled in backend configuration |

The S3 bucket and DynamoDB table were created separately from the infrastructure configuration being migrated. This avoids depending on the same state backend to create the backend resources themselves.

### S3 backend configuration used

Added this block to `terraform.tf`:

```hcl
terraform {
  backend "s3" {
    bucket         = "terraweek-state-snigdha"
    key            = "dev/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraweek-state-lock"
    encrypt        = true
  }
}
```

### Migrate the existing local state

```bash
terraform init
```

Terraform detected that the backend configuration had changed and asked whether to copy the existing state to the new backend. Confirmed by entering `yes`.

You can also explicitly request migration with:

```bash
terraform init -migrate-state
```

**Important distinction:**

- `terraform init -migrate-state` asks Terraform to migrate existing state to the newly configured backend.
- `terraform init -reconfigure` tells Terraform to use the new backend configuration without migrating the old state. Do not use it as a substitute for migration when you need to preserve existing state.

### What happened after migration?

- Terraform successfully configured the S3 backend.
- The existing state was copied to the S3 object `dev/terraform.tfstate`.
- The S3 backend became the active state location.
- The local state file was no longer the active backend state.
- A subsequent `terraform plan` was used to verify that the migration had not caused infrastructure changes.

**Interview answer:** A remote backend stores Terraform state centrally. For an S3 backend, the bucket stores the state object. Versioning helps recover earlier state versions, while locking prevents conflicting Terraform operations from working on the same state at the same time.

### Why enable S3 versioning?

If a state object is overwritten or damaged, S3 versioning can preserve earlier versions so an administrator can recover a previous version.

Versioning is a recovery aid. It does not replace access control, backups, or careful state operations.

### Important security point

Terraform state can contain sensitive values. Protect the bucket with restrictive IAM and bucket policies, encryption, and appropriate access controls. Do not commit state files or credentials to Git.

---

## 3. State locking: preventing concurrent operations

### What problem does locking solve?

Imagine two engineers run Terraform operations against the same state at nearly the same time.

Both operations could attempt to update the state. Without locking, concurrent writes can cause conflicting or inconsistent state updates.

State locking allows one operation to hold the lock while another operation waits or fails to acquire it.

### Hands-on test

Used two terminals in the same Terraform project:

**Terminal 1**

```bash
terraform apply
```

Kept the operation waiting at the approval prompt, while it held the state lock.

**Terminal 2**

```bash
terraform plan
```

The second operation attempted to acquire the same state lock and was blocked by the lock held by the first operation.

The exact wording of the lock error depends on the Terraform version and backend. The important result is that the second operation could not proceed while the first held the lock.

### Why is locking important in a team?

It protects shared state from simultaneous Terraform operations. This matters when multiple engineers or CI/CD jobs work against the same environment.

### What if a lock becomes stale?

A stale lock can remain if an operation terminates unexpectedly. First verify that no Terraform operation is still running and that the lock is genuinely stale.

Only then consider:

```bash
terraform force-unlock <LOCK_ID>
```

Do not force-unlock a state that is actively being used. Doing so can allow concurrent operations and risk state corruption.

### DynamoDB deprecation note

The Day 64 exercise used the S3 backend's `dynamodb_table` setting for locking. Terraform printed a deprecation warning in this environment and recommended `use_lockfile` instead.

For newer Terraform versions, the S3 backend supports native lock files using:

```hcl
terraform {
  backend "s3" {
    bucket       = "your-state-bucket"
    key          = "dev/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

The DynamoDB locking option is deprecated in current Terraform documentation. The warning did not mean the backend migration had failed; it indicated that the configuration uses the older locking method. For a new setup, check the Terraform version and current backend documentation before choosing the locking configuration.

---

## 4. Import: bring an existing resource under Terraform management

Not every resource is created by Terraform. A resource may already exist because someone created it manually in AWS or because it was created by another process.

Terraform needs a resource block in configuration and a state association to manage that existing resource.

### Existing S3 bucket

Created this bucket manually in AWS:

```text
terraweek-import-test-snigdha
```

Added a resource block:

```hcl
resource "aws_s3_bucket" "imported" {
  bucket = "terraweek-import-test-snigdha"
}
```

Imported the existing bucket:

```bash
terraform import \
  aws_s3_bucket.imported \
  terraweek-import-test-snigdha
```

Then checked the state:

```bash
terraform state list
```

The bucket appeared in Terraform state. Later, its resource address was renamed to `aws_s3_bucket.logs_bucket` during the state surgery task.

### What does `terraform import` actually do?

It associates an existing real-world resource with a Terraform resource address in state.

It does **not**:

- Create a new AWS resource.
- Automatically write a complete resource block into your `.tf` files.
- Guarantee that the configuration already matches every setting on the real resource.

The resource block must be written separately. After importing, run `terraform plan` and review any proposed changes. Update the configuration when the intention is to preserve the existing resource settings.

### Import versus create

| Creating from scratch | Importing |
|---|---|
| Terraform creates the resource in the provider | The resource already exists |
| Terraform records it in state as part of creation | Import associates it with a resource address in state |
| Resource configuration is used to create it | Resource configuration must still be written and aligned |

**Interview answer:** I use import when infrastructure already exists outside Terraform and I want Terraform to manage it without creating a duplicate. I first define the resource block, import the real resource into the matching Terraform address, and then run a plan to check configuration alignment.

---

## 5. State surgery: `terraform state mv` and `terraform state rm`

These commands change Terraform's state tracking. They do not directly rename or delete the actual AWS resource.

### A. `terraform state mv`

Used to change a resource's address in state.

The exercise renamed the imported bucket from:

```text
aws_s3_bucket.imported
```

to:

```text
aws_s3_bucket.logs_bucket
```

Command:

```bash
terraform state mv \
  aws_s3_bucket.imported \
  aws_s3_bucket.logs_bucket
```

Then updated the resource block in the `.tf` configuration to use `logs_bucket` and ran:

```bash
terraform plan
```

### When would I use `state mv`?

For example, when renaming a resource block or moving a resource into a module, I may need to update its Terraform address. Moving the state address tells Terraform that the new address refers to the same existing object.

Without the corresponding state move, Terraform may interpret the old address as something to remove and the new address as something to create.

### B. `terraform state rm`

Used:

```bash
terraform state rm aws_s3_bucket.logs_bucket
```

This removed the bucket's address from Terraform state. It did not delete the S3 bucket from AWS.

The bucket continued to exist in AWS, but Terraform no longer tracked it at that address.

Then re-imported it:

```bash
terraform import \
  aws_s3_bucket.logs_bucket \
  terraweek-import-test-snigdha
```

### When would I use `state rm`?

When I intentionally want Terraform to stop managing a resource without destroying the real infrastructure.

For example, a resource may be handed over to another management process. Before removing it from state, I must understand what the next plan will do.

**Important:** If the resource block remains in the configuration after `state rm`, a normal plan can propose creating a new resource because Terraform sees a configured resource address that is no longer tracked. Removing it from state does not automatically remove the resource block.

### `mv` versus `rm`

| Command | Effect on state | Effect on AWS resource |
|---|---|---|
| `terraform state mv` | Changes the resource address | Does not rename or recreate the resource |
| `terraform state rm` | Removes the resource from state | Does not delete the resource |

---

## 6. State drift: detect and handle changes made outside Terraform

### What is drift?

Drift occurs when the real infrastructure changes outside Terraform and no longer matches the configuration Terraform is meant to manage.

Examples include:

- Someone changes an EC2 instance type in the AWS Console.
- Someone changes a resource tag manually.
- Someone modifies a security group outside Terraform.
- An instance is stopped outside Terraform.

### Hands-on EC2 drift exercise

Used this EC2 instance:

```text
i-02894f2e588906bd3
```

The original configuration expected:

| Attribute | Original configuration |
|---|---|
| Instance type | `t2.micro` |
| Name tag | `Terraweek-dev-server` |
| Public IP association | Enabled |

Manually changed the EC2 instance in AWS:

| Attribute | Actual value after manual change |
|---|---|
| Instance type | `t3.micro` |
| Name tag | `Manuallyupdated` |
| Instance state | Stopped |
| Public IP association | Disabled |
| Public IP | None |

### Step 1: Detect the drift

Ran:

```bash
terraform plan
```

Terraform refreshed information from AWS and reported that the EC2 instance had changed outside Terraform.

The plan proposed replacing the EC2 instance. One reason was:

```text
associate_public_ip_address = false -> true # forces replacement
```

The plan also showed the difference between the configured instance type and tag and the values found in AWS.

### Step 2: Preview state updates only

Ran:

```bash
terraform plan --refresh-only
```

This displayed the differences between the previously recorded state and the actual AWS values.

A refresh-only plan does not try to restore the original configuration. It previews the state updates Terraform can make to reflect what exists in AWS.

### Step 3: Save the refreshed values in state

Ran:

```bash
terraform apply -refresh-only
```

Confirmed by entering `yes`.

Result:

```text
Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
```

Terraform updated its state to reflect the observed AWS values. It did not modify or recreate the EC2 instance.

### Step 4: Run a normal plan again

Ran:

```bash
terraform plan
```

Terraform no longer reported the same external changes as newly detected drift. However, the plan still proposed replacing the EC2 instance.

Why?

Because the state now reflected the actual AWS values, but the `.tf` configuration still described the original desired values:

- Configuration: `t2.micro`
- Actual infrastructure and refreshed state: `t3.micro`
- Configuration: `Terraweek-dev-server`
- Actual infrastructure and refreshed state: `Manuallyupdated`
- Configuration: public IP association enabled
- Actual infrastructure and refreshed state: disabled

Terraform was now comparing the updated state with the unchanged configuration. It proposed bringing AWS back to the configured values. Because changing the public IP association requires replacement, the plan showed one resource to add and one to destroy.

**No normal apply was performed in this final step.** The replacement was reviewed but not executed.

### Two ways to handle drift

**Option A: Restore the configuration's desired values**

Keep the original `.tf` configuration and review the normal plan. If the proposed changes are acceptable, apply them to bring the infrastructure back in line with the configuration.

Be careful: some changes require resource replacement. Review the plan before applying.

**Option B: Accept the manual changes**

If the manual changes are intentional, update the `.tf` configuration to reflect the intended new values. Then run `terraform plan` again and review until the plan matches the intended outcome.

### Refresh-only versus normal plan/apply

| Command | What it does |
|---|---|
| `terraform plan` | Refreshes provider information and proposes changes needed to match configuration |
| `terraform plan --refresh-only` | Previews updates to state based on real infrastructure |
| `terraform apply -refresh-only` | Saves refreshed values to state without changing real infrastructure |
| `terraform apply` | Executes the normal plan and can modify or replace real infrastructure |

**Most important lesson:** Refresh-only does not fix the infrastructure or update `.tf` files. It updates Terraform's record of what exists. A normal plan can still propose changes when the configuration differs from reality.

---

## 7. How teams reduce drift in production

Teams can reduce unexpected drift by:

1. Restricting direct manual changes to managed infrastructure.
2. Making infrastructure changes through reviewed Terraform code.
3. Using CI/CD pipelines to run consistent plans and applies.
4. Reviewing Terraform plans before applying them.
5. Using remote state and locking for shared environments.
6. Monitoring infrastructure for changes made outside approved workflows.
7. Protecting state with appropriate permissions, encryption, and versioning.

The goal is to make Terraform configuration the controlled source of desired infrastructure and to make infrastructure changes traceable.

---

## 8. Interview revision: questions and clear answers

### Q1. What is Terraform state?

Terraform state is the record that maps Terraform resource addresses to real infrastructure objects and stores information Terraform needs to manage them.

### Q2. Why do we need a state file?

Terraform uses state to identify which real resources it manages, understand their recorded attributes, and calculate what needs to change.

### Q3. Why use a remote backend?

A remote backend provides a shared location for state so team members and automation can work with the same state instead of separate local copies.

### Q4. Why use S3 versioning?

It preserves previous versions of the state object, which can help recover from accidental overwrites or corruption.

### Q5. What is state locking?

State locking prevents multiple Terraform operations from modifying the same state simultaneously.

### Q6. What happens if two engineers run Terraform at the same time?

If the backend supports locking, one operation acquires the lock. The other operation must wait or fails to acquire it until the lock is released.

### Q7. What is Terraform import?

Import associates an existing resource with a Terraform resource address in state. The resource block must also exist in configuration.

### Q8. Does import create the resource?

No. The resource already exists. Import connects it to Terraform state.

### Q9. What is the difference between `state mv` and `state rm`?

`state mv` changes a resource's address in state. `state rm` removes the resource from state without deleting it from the provider.

### Q10. What is drift?

Drift is a difference between the actual infrastructure and the desired configuration, often caused by changes outside Terraform.

### Q11. What does `terraform apply -refresh-only` do?

It refreshes Terraform state to reflect the actual infrastructure without modifying that infrastructure.

### Q12. If I run refresh-only, will my `.tf` file change?

No. Refresh-only updates state, not the configuration files.

### Q13. Why can a normal plan still show changes after refresh-only?

Because the state now reflects actual infrastructure, but the `.tf` configuration may still describe different desired values. Terraform proposes changes to bring reality back to the configuration.

### Q14. What is `terraform force-unlock`?

It manually releases a state lock. It should only be used after confirming that no active Terraform operation is holding that lock.

### Q15. Does moving state to S3 recreate resources?

No. State migration moves Terraform's state data to the remote backend. It does not recreate the infrastructure.

---

## 9. Commands covered

```bash
# Inspect state
terraform show
terraform state list
terraform state show aws_instance.web
terraform state show aws_vpc.main

# Initialize and migrate backend
terraform init
terraform init -migrate-state

# Test locking
terraform apply
terraform plan

# Import an existing resource
terraform import aws_s3_bucket.imported terraweek-import-test-snigdha

# State surgery
terraform state mv aws_s3_bucket.imported aws_s3_bucket.logs_bucket
terraform state rm aws_s3_bucket.logs_bucket

# Re-import
terraform import aws_s3_bucket.logs_bucket terraweek-import-test-snigdha

# Detect and inspect drift
terraform plan
terraform plan --refresh-only
terraform apply -refresh-only

# Release a confirmed stale lock only when safe
terraform force-unlock <LOCK_ID>
```

---

## 10. Day 64 completion notes

Completed the hands-on exercises for:

- Inspecting Terraform state and resource attributes.
- Creating an S3 backend with versioning.
- Configuring DynamoDB-based state locking and testing concurrent access.
- Migrating existing local state to S3.
- Importing an existing S3 bucket.
- Renaming a resource address with `terraform state mv`.
- Removing and re-importing a resource with `terraform state rm` and `terraform import`.
- Manually changing EC2 settings and detecting drift.
- Applying a refresh-only operation to update Terraform state without changing AWS infrastructure.

**Drift exercise result:** The refresh-only operation completed with zero resources added, changed, or destroyed. The subsequent normal plan still proposed EC2 replacement because the configuration retained the original desired settings. That replacement was not applied.

## Final takeaway

Terraform state is essential for tracking infrastructure, but it is not the same as the configuration or the actual infrastructure.

- Configuration describes what should exist.
- State records what Terraform knows about managed resources.
- The provider reports what exists in the real environment.
- A plan compares the desired configuration with the refreshed infrastructure information.
- An apply executes the approved changes.

Understanding the difference between these three is the foundation of reliable Terraform state management.

---

**Day 64 -- Terraform State Management and Remote Backends**

#90DaysOfDevOps #TerraWeek #DevOpsKaJosh #TrainWithShubham
