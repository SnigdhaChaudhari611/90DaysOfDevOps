# Day 67: Terraform Capstone – Multi-Environment AWS Infrastructure

## Overview

This capstone brings together Terraform fundamentals, reusable modules, workspaces, environment-specific variables, and remote state management.

The goal is to use one Terraform configuration to provision separate AWS infrastructure for three environments:

- **Dev**
- **Staging**
- **Production**

Each environment has its own VPC, public subnet, internet gateway, route table, security group, and EC2 instance. Terraform workspaces provide separate state for each environment, while an Amazon S3 backend stores the state remotely.

> **Progress note:** The Terraform configuration, modules, S3 backend, and workspaces were created. The first Dev apply encountered an EC2 Availability Zone compatibility error. The deployment, verification, destruction, workspace cleanup, and final screenshots must be updated with the actual results before treating this capstone as fully completed.

---

## 1. Project structure

```text
terraweek-capstone/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
├── locals.tf
├── dev.tfvars
├── staging.tfvars
├── prod.tfvars
├── .gitignore
└── modules/
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── security-group/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── ec2-instance/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

## 2. Architecture

The root Terraform configuration calls three reusable modules:

1. **VPC module** – creates the VPC, public subnet, internet gateway, public route table, and subnet association.
2. **Security group module** – creates the environment-specific security group and inbound/outbound rules.
3. **EC2 instance module** – creates an EC2 instance using the selected AMI, instance type, subnet, and security group.

The environment is selected using `terraform.workspace`. Each workspace receives its own variable file.

```text
                    Terraform configuration
                              |
                 Current Terraform workspace
                   /          |          \
                 dev        staging       prod
                  |            |            |
             dev.tfvars   staging.tfvars  prod.tfvars
                  |            |            |
             VPC module   VPC module    VPC module
             SG module    SG module     SG module
             EC2 module   EC2 module    EC2 module
                  |            |            |
              AWS Dev      AWS Staging   AWS Production

               Remote Terraform state in Amazon S3
```

## 3. Environment configuration

The environment-specific values used in the project are:

| Setting | Dev | Staging | Production |
|---|---|---|---|
| VPC CIDR | `10.0.0.0/16` | `10.1.0.0/16` | `10.2.0.0/16` |
| Public subnet CIDR | `10.0.1.0/24` | `10.1.1.0/24` | `10.2.1.0/24` |
| EC2 instance type | `t2.micro` | `t2.small` | `t3.small` |
| Inbound TCP ports | 22, 80 | 22, 80, 443 | 80, 443 |

The CIDR ranges are different so that the environments do not overlap.

### `dev.tfvars`

```hcl
vpc_cidr      = "10.0.0.0/16"
subnet_cidr   = "10.0.1.0/24"
instance_type = "t2.micro"
ingress_ports = [22, 80]
```

### `staging.tfvars`

```hcl
vpc_cidr      = "10.1.0.0/16"
subnet_cidr   = "10.1.1.0/24"
instance_type = "t2.small"
ingress_ports = [22, 80, 443]
```

### `prod.tfvars`

```hcl
vpc_cidr      = "10.2.0.0/16"
subnet_cidr   = "10.2.1.0/24"
instance_type = "t3.small"
ingress_ports = [80, 443]
```

These files contain infrastructure settings, not credentials. Do not put access keys, passwords, private keys, or other secrets in `.tfvars` files.

## 4. Reusable Terraform modules

### VPC module

The VPC module accepts the VPC CIDR, public subnet CIDR, environment, and project name.

It creates:

- An AWS VPC with DNS support and DNS hostnames enabled.
- A public subnet with public IP assignment enabled on launch.
- An internet gateway attached to the VPC.
- A public route table with a default route through the internet gateway.
- A route table association for the public subnet.

The module exports the VPC ID and subnet ID for use by the other modules.

### Security group module

The security group module accepts the VPC ID, inbound TCP ports, environment, and project name.

It creates:

- Inbound rules from the configured port list.
- An outbound rule allowing all traffic.

**Security reminder:** The initial module configuration used `0.0.0.0/0` for all inbound ports. This means SSH (port 22), when enabled, is open to the internet. Before using this configuration beyond a temporary lab, restrict SSH to a trusted public IP address or use a managed access method such as AWS Systems Manager Session Manager. Public HTTP/HTTPS access should also be limited to the intended service requirements.

### EC2 instance module

The EC2 module accepts:

- AMI ID
- Instance type
- Subnet ID
- Security group IDs
- Environment
- Project name

The root configuration discovers the latest matching Ubuntu 24.04 AMI from Canonical's AWS account and passes its ID to the module.

The module exports the EC2 instance ID and public IP address.

## 5. Workspace-based environment selection

Terraform workspaces allow the same configuration to maintain separate state files.

The workspaces created for this project are:

```bash
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod
```

List the available workspaces:

```bash
terraform workspace list
```

Select an environment:

```bash
terraform workspace select dev
```

Check the currently selected workspace:

```bash
terraform workspace show
```

The root configuration uses the selected workspace:

```hcl
locals {
  environment = terraform.workspace
  name_prefix = "${var.project_name}-${local.environment}"

  common_tags = {
    Project     = var.project_name
    Environment = local.environment
    ManagedBy   = "Terraform"
    Workspace   = terraform.workspace
  }
}
```

This allows resource names and environment tags to reflect the selected workspace.

> Terraform CLI workspaces separate state, but they are not a complete security boundary. For strongly isolated production environments, separate AWS accounts and access controls are generally preferable.

## 6. Remote state with Amazon S3

A remote backend was configured in `providers.tf`.

The backend configuration uses:

- An S3 bucket in `ap-south-1`.
- A project-specific state key.
- Server-side encryption.
- S3 native state locking with `use_lockfile = true`.

Example backend configuration (replace the bucket placeholder with the actual bucket name):

```hcl
terraform {
  backend "s3" {
    bucket       = "<YOUR_STATE_BUCKET>"
    key          = "terraweek-capstone/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}
```

The S3 bucket was configured with:

- Versioning enabled.
- Default server-side encryption using AES-256.
- S3 Block Public Access enabled.

Terraform was initialized/migrated using:

```bash
terraform init -migrate-state
```

After configuring the S3 backend, the workspaces were recreated and verified.

For a non-default workspace, Terraform stores state under the configured workspace key prefix. With the default S3 workspace prefix, named workspace state objects are stored under an `env:` path in the bucket.

The AWS identity used by Terraform needs appropriate permissions for the state object and its lock file. Keep the state bucket private; Terraform state can contain sensitive values.

## 7. Validation and deployment workflow

Format the Terraform files:

```bash
terraform fmt -recursive
```

Validate the configuration:

```bash
terraform validate
```

The configuration validation completed successfully before the first deployment attempt.

### Deploy Dev

```bash
terraform workspace select dev
terraform plan -var-file="dev.tfvars"
terraform apply -var-file="dev.tfvars"
terraform output
```

### Deploy Staging

```bash
terraform workspace select staging
terraform plan -var-file="staging.tfvars"
terraform apply -var-file="staging.tfvars"
terraform output
```

### Deploy Production

```bash
terraform workspace select prod
terraform plan -var-file="prod.tfvars"
terraform apply -var-file="prod.tfvars"
terraform output
```

Always review the plan before applying. Terraform asks for confirmation before making changes unless automatic approval is explicitly enabled.

## 8. Issue encountered during the first Dev deployment

The first Dev apply failed while creating the EC2 instance. AWS returned:

```text
Unsupported: Your requested instance type (t2.micro) is not supported
in your requested Availability Zone (ap-south-1c).
Please retry the request by not specifying an Availability Zone
or choosing ap-south-1a, ap-south-1b.
```

This is an AWS Availability Zone and instance-type compatibility issue. It does not indicate that the Terraform module syntax is invalid.

The proposed lab fix was to select a supported Availability Zone for the public subnet in `modules/vpc/main.tf`:

```hcl
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  availability_zone       = "ap-south-1a"
  cidr_block              = var.public_subnet_cidr
  map_public_ip_on_launch = true

  # Keep the existing tags here.
}
```

Changing a subnet's Availability Zone requires replacement. Review the plan carefully because Terraform may also need to replace resources that depend on the subnet.

After making this change:

```bash
terraform fmt -recursive
terraform workspace select dev
terraform plan -var-file="dev.tfvars"
```

Only apply after reviewing the plan and confirming that the proposed replacements are expected.

**Final resolution and successful deployment result: add the actual output here after the Dev apply succeeds.**

## 9. Outputs and verification

The root module exposes:

- Current environment
- VPC ID
- Public subnet ID
- Security group ID
- EC2 instance ID
- EC2 public IP

Display the outputs for the selected workspace:

```bash
terraform output
```

Verify resources in AWS as well. For example:

```bash
aws ec2 describe-instances \
  --region ap-south-1 \
  --filters "Name=tag:Project,Values=terraweek" \
  --query 'Reservations[].Instances[].[InstanceId,State.Name,InstanceType,Placement.AvailabilityZone,PublicIpAddress]' \
  --output table
```

Add the real verification output and screenshots after all three environments have been deployed.

## 10. Cleanup

AWS resources can incur charges while they are running. After verification, destroy each environment separately.

```bash
terraform workspace select dev
terraform destroy -var-file="dev.tfvars"
```

```bash
terraform workspace select staging
terraform destroy -var-file="staging.tfvars"
```

```bash
terraform workspace select prod
terraform destroy -var-file="prod.tfvars"
```

Review each destroy plan and confirm only when it contains the resources intended for that environment.

After confirming that all three environments have been destroyed, switch to the default workspace and remove the named workspaces:

```bash
terraform workspace select default
terraform workspace delete dev
terraform workspace delete staging
terraform workspace delete prod
terraform workspace list
```

Keep the S3 backend bucket if you want to retain the remote state history. Do not delete the bucket until you have checked whether any state files or other objects need to be retained.

## 11. Git and repository hygiene

The five Terraform learning folders were organized under one parent repository:

```text
terraform-learning/
├── terraform-basics/
├── terraform-aws-infra/
├── terraform-eks/
├── terraform-modules/
└── terraweek-capstone/
```

The root `.gitignore` excludes generated Terraform directories and state files while retaining provider lock files.

Important ignore patterns include:

```gitignore
**/.terraform/
*.tfstate
*.tfstate.*
*.pem
*.key
.env
.env.*
```

Do not commit:

- `.terraform/` directories or downloaded provider binaries.
- Terraform state files.
- AWS access keys or secret keys.
- Private keys or credential files.
- Other secrets.

Do commit `.terraform.lock.hcl` so provider selections and checksums are recorded.

If Terraform provider binaries were accidentally staged, remove them from Git's index while keeping local files:

```bash
git ls-files -z | grep -z '/\.terraform/' | xargs -0 -r git rm --cached --
```

Then stage the ignore rule and amend the unpublished commit if necessary:

```bash
git add .gitignore
git commit --amend --no-edit
```

Check the index before pushing:

```bash
git ls-files | grep '\.terraform/'
git status
```

The first command should return no tracked `.terraform` paths.

---

## 12. Terraform best practices learned

1. **Use reusable modules:** Keep VPC, security group, and EC2 logic separate so configurations can be reused.
2. **Use variables:** Avoid hardcoding environment-specific values in resource definitions.
3. **Use workspaces carefully:** Workspaces can maintain separate state for similar environments using one configuration.
4. **Use remote state:** Store shared state in a secure remote backend instead of relying only on a local state file.
5. **Enable state versioning:** S3 versioning can help recover earlier state object versions.
6. **Enable state locking:** Prevent multiple Terraform operations from modifying the same state at the same time.
7. **Use provider lock files:** Commit `.terraform.lock.hcl` to make provider selections reproducible.
8. **Tag resources:** Include project, environment, and management tags to make AWS resources easier to identify.
9. **Review plans:** Run `terraform plan` and inspect changes before applying or destroying resources.
10. **Protect secrets:** Never commit credentials, private keys, or sensitive state.
11. **Apply least privilege:** Restrict inbound access, especially SSH, and grant the Terraform identity only the permissions it needs.
12. **Clean up lab resources:** Destroy temporary infrastructure after testing to avoid unnecessary charges.

## 13. Day 67 key concepts

| Concept | Simple explanation |
|---|---|
| Terraform module | A group of Terraform files that manages a reusable part of infrastructure. |
| Root module | The main Terraform configuration that calls child modules. |
| Input variable | A value passed into a Terraform configuration or module. |
| Output | A value exposed after Terraform creates or reads resources. |
| Local value | A named expression that can be reused within a configuration. |
| Terraform workspace | A separate state instance for the same configuration. |
| `terraform.workspace` | Returns the name of the currently selected workspace. |
| `.tfvars` file | Supplies values for variables, such as CIDRs and instance types. |
| S3 backend | Stores Terraform state remotely in Amazon S3. |
| State locking | Helps prevent simultaneous operations from changing the same state. |
| `terraform plan` | Previews the changes Terraform intends to make. |
| `terraform apply` | Applies the planned infrastructure changes. |
| `terraform destroy` | Removes infrastructure managed by the selected state. |
| Provider lock file | Records the selected provider versions and checksums. |

---

## 14. Completion checklist

Update this checklist only after confirming each result.

- [x] Create the project directory and Terraform module structure.
- [x] Configure root Terraform files and environment variable files.
- [x] Create VPC, security group, and EC2 modules.
- [x] Configure Dev, Staging, and Production workspace names.
- [x] Configure the S3 remote backend.
- [x] Enable S3 versioning, encryption, and public access blocking.
- [x] Initialize Terraform with the S3 backend.
- [x] Validate the Terraform configuration.
- [ ] Resolve the Dev Availability Zone issue and successfully deploy Dev.
- [ ] Successfully deploy Staging.
- [ ] Successfully deploy Production.
- [ ] Verify all three environments and their separate state.
- [ ] Destroy all three environments.
- [ ] Delete the named workspaces after destruction.
- [ ] Add actual command outputs and screenshots.
- [ ] Push the final capstone documentation to GitHub.

## Final takeaway

This capstone demonstrates how Terraform can turn repeated infrastructure configurations into reusable modules and manage different environments using separate variable files and workspaces. It also introduces remote state management with S3, state locking, resource tagging, validation, deployment planning, and cleanup.

**Remember:** A successful `terraform validate` confirms configuration validity, not that AWS will accept every resource configuration. Always review plans and verify real deployments.
