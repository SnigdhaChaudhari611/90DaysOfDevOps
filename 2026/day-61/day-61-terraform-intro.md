# Day 61: Introduction to Terraform

## Overview

This day focused on Infrastructure as Code (IaC) and using Terraform to provision, inspect, modify, and destroy AWS resources.

Instead of creating infrastructure manually through the AWS Console, Terraform lets us define the desired infrastructure in configuration files and manage it through commands.

## 1. Infrastructure as Code (IaC)

Infrastructure as Code means managing infrastructure through code instead of creating and configuring resources manually.

### Why IaC matters in DevOps

- **Repeatability:** The same configuration can be used to create infrastructure consistently.
- **Automation:** Infrastructure changes can be included in automated workflows.
- **Version control:** Configuration files can be tracked in Git, making changes easier to review.
- **Consistency:** It reduces manual steps and configuration differences between environments.
- **Faster recovery:** Infrastructure can be recreated from code when needed.

### Manual AWS Console vs IaC

When infrastructure is created manually, each resource must be configured through the console. This can lead to inconsistent settings, missed steps, and difficulty repeating the same setup.

With IaC, the desired configuration is written in files. Terraform compares that configuration with the current infrastructure and shows what needs to be created, changed, or removed.

### Terraform and other tools

| Tool | Main purpose |
|---|---|
| Terraform | Provisions and manages infrastructure across multiple cloud providers and platforms. |
| AWS CloudFormation | Defines and manages AWS infrastructure using templates. |
| Ansible | Automates configuration management and application setup. |
| Pulumi | Defines infrastructure using general-purpose programming languages. |

Terraform is declarative: we describe the desired end state, and Terraform works out the actions needed to reach it. It is also cloud-agnostic, so it can manage resources from different providers through their providers.

## 2. Install Terraform and configure AWS CLI

Terraform and the AWS CLI are required to manage AWS infrastructure from the terminal.

Verify the installations and AWS identity:

```bash
terraform -version
aws sts get-caller-identity
```

`terraform -version` displays the installed Terraform version.

`aws sts get-caller-identity` confirms which AWS identity is being used by the configured credentials.

## 3. Create an S3 bucket using Terraform

I created a working directory and defined the AWS provider and S3 bucket in `main.tf`.

The configuration used the AWS provider in the `ap-southeast-1` region and created this bucket:

```hcl
resource "aws_s3_bucket" "example" {
  bucket = "snigdha-terraform-basics-2026"

  tags = {
    Name        = "My bucket"
    Environment = "Dev"
  }
}
```

The provider configuration specified:

```hcl
provider "aws" {
  region = "ap-southeast-1"
}
```

### Terraform workflow

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
```

| Command | Purpose |
|---|---|
| `terraform init` | Initializes the working directory and downloads the required provider. |
| `terraform fmt` | Formats Terraform configuration files. |
| `terraform validate` | Checks whether the configuration is valid. |
| `terraform plan` | Previews the infrastructure changes Terraform intends to make. |
| `terraform apply` | Applies the planned changes and creates or updates resources. |

The S3 bucket was created successfully. I verified it using:

```bash
aws s3 ls
```

### Issues encountered

**AWS region mismatch**

My initial provider configuration used `us-east-1`, but the AWS response indicated that the bucket operation needed to use `ap-southeast-1`. I updated the provider region and ran the Terraform workflow again.

**S3 bucket name already taken**

S3 bucket names must be globally unique. The initial name `my-tf-bucket` was already in use, so I changed it to:

`snigdha-terraform-basics-2026`

The bucket was then created successfully.

**AWS CLI command**

I initially tried `aws s3 list`. The correct command to list S3 buckets is:

```bash
aws s3 ls
```

### What is the `.terraform/` directory?

Terraform creates the `.terraform/` directory during initialization. It contains the downloaded provider plugins and other working data needed by Terraform.

It is generated locally and should not normally be committed to Git.

## 4. Create an EC2 instance using Terraform

After creating the S3 bucket, I added an EC2 instance resource to `main.tf`.

The instance configuration used:

- AMI: `ami-0e683f46a1f00df41`
- Instance type: `t2.micro`
- Region: `ap-southeast-1`
- Name tag: `TerraWeek-Day1`

The EC2 resource address in Terraform was:

```text
aws_instance.web-01
```

I ran:

```bash
terraform plan
terraform apply
```

Terraform detected that the S3 bucket was already managed in the state and planned to create the EC2 instance.

The EC2 instance was created successfully. Its instance ID was:

```text
i-0d7cddda298ab1a2e
```

I used the following command to inspect the resources and their attributes:

```bash
terraform show
```

### How did Terraform know that the S3 bucket already existed?

Terraform maintains a state file that records the resources it manages and their attributes.

When I added the EC2 resource and ran `terraform plan`, Terraform checked the existing state and refreshed the managed S3 bucket's information from AWS. It therefore planned to create the EC2 instance without creating the bucket again.

## 5. Inspect Terraform state

Terraform state keeps track of the infrastructure managed by a Terraform configuration. It maps resources in the configuration to real resources in the cloud provider and stores information about their attributes.

### List resources in the state

```bash
terraform state list
```

At this stage, the output showed:

```text
aws_instance.web-01
aws_s3_bucket.example
```

### Inspect individual resources

```bash
terraform state show aws_s3_bucket.example
terraform state show aws_instance.web-01
```

These commands display the attributes Terraform has recorded for each resource.

I also used:

```bash
terraform show
```

to inspect the current state in a readable format.

### What does the state file contain?

The `terraform.tfstate` file is a JSON file containing information such as:

- Terraform state version and metadata.
- Resource types and resource addresses.
- Resource IDs assigned by the provider.
- Attributes recorded for managed resources.
- Information Terraform uses to map configuration resources to real infrastructure.

### Why should we not edit the state file manually?

Terraform relies on the state file to understand which real resources it manages. Manually changing it can break resource mappings or make the state inconsistent with the actual infrastructure.

Use Terraform commands to inspect and manage state instead of editing the file directly.

### Why should we not commit the state file to Git?

State files can contain sensitive information and infrastructure details, including resource metadata and identifiers. They should be stored securely and access should be controlled.

For team environments, remote state storage with appropriate access controls and state locking is generally preferred.

### State file after destroying the resources

After the infrastructure was destroyed, I checked the working directory and inspected the state file:

```bash
ls
cat terraform.tfstate
```

The final state contained:

```json
{
  "version": 4,
  "terraform_version": "1.17.1",
  "serial": 10,
  "lineage": "6de83fe0-a19d-4429-9b24-b7bbd618b290",
  "outputs": {},
  "resources": [],
  "check_results": null
}
```

The empty `resources` array is expected because Terraform had destroyed the resources it was managing.

## 6. Modify and destroy infrastructure

I changed the EC2 instance's Name tag in `main.tf`:

```hcl
tags = {
  Name = "TerraWeek-Modified"
}
```

Then I ran:

```bash
terraform plan
```

Terraform showed that the tag could be updated without replacing the EC2 instance.

The plan showed:

```text
~ update in-place
```

It reported:

```text
0 to add, 1 to change, 0 to destroy
```

I applied the change:

```bash
terraform apply
```

The EC2 tag was updated successfully.

### Understanding Terraform plan symbols

| Symbol | Meaning |
|---|---|
| `+` | Create a resource. |
| `-` | Destroy a resource. |
| `~` | Update a resource in place, when supported by the provider and resource. |
| `-/+` | Destroy and then recreate a resource. |

An in-place update changes supported attributes without replacing the resource. Some changes require Terraform to destroy and recreate a resource instead. The plan shows which action Terraform intends to take.

### Destroy the infrastructure

After completing the modification, I removed the resources using:

```bash
terraform destroy
```

I confirmed the operation when Terraform prompted for approval.

Terraform successfully destroyed both resources:

- EC2 instance: `aws_instance.web-01`
- S3 bucket: `aws_s3_bucket.example`

The command reported:

```text
Destroy complete! Resources: 2 destroyed.
```

This completed the cleanup and left the Terraform state with no managed resources.

## Key takeaways

- Infrastructure can be provisioned and managed through code instead of manual console steps.
- `terraform init`, `plan`, `apply`, and `destroy` form the basic Terraform workflow.
- Terraform state tracks the relationship between configuration and real infrastructure.
- `terraform state list` and `terraform state show` help inspect managed resources.
- A plan should be reviewed before applying infrastructure changes.
- Some resource changes happen in place, while others require replacement.
- State files should be protected and should not be edited manually or committed to a public Git repository.
- Always clean up temporary cloud resources when the exercise is complete.

## Commands used

```bash
terraform -version
aws sts get-caller-identity

terraform init
terraform fmt
terraform validate
terraform plan
terraform apply

aws s3 ls

terraform show
terraform state list
terraform state show aws_s3_bucket.example
terraform state show aws_instance.web-01

terraform destroy

ls
cat terraform.tfstate
```
