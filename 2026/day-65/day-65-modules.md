# Day 65 -- Terraform Modules: Build Reusable Infrastructure

## Overview

Until now, most of the Terraform configuration was written directly in a
root `main.tf` file. That is manageable for small exercises, but becomes
difficult to maintain when infrastructure grows or needs to be reused
across environments.

Terraform modules solve this by grouping related resources into reusable
units. A module can accept input variables and expose outputs, much like
a function in programming:

-   **Inputs** configure what the module creates.
-   **Resources** define the infrastructure.
-   **Outputs** expose useful information to the calling configuration.

In this exercise, I built custom EC2 and Security Group modules, reused
the EC2 module to create two instances, and used a public Terraform
Registry module to provision the VPC.

------------------------------------------------------------------------

## 1. Understand the Module Structure

A Terraform module is a collection of Terraform configuration files in a
directory.

The project structure:

``` text
terraform-modules/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
└── modules/
    ├── ec2-instance/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── security-group/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

### Root module vs child module

**Root module**

The root module is the Terraform configuration in the directory where
Terraform commands are run. In this project, it contains the provider
configuration and calls the child modules.

**Child module**

A child module is a module called by another module. Here,
`ec2-instance` and `security-group` are local child modules called by
the root module.

The root module passes input values to child modules and can use their
outputs.

------------------------------------------------------------------------

## 2. Build a Custom EC2 Module

The EC2 module is responsible for creating an EC2 instance. Instead of
writing the resource twice, I defined it once and called the module
twice with different instance names.

### `modules/ec2-instance/variables.tf`

``` hcl
variable "ami_id" {
  description = "AMI ID for the EC2 instance"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "subnet_id" {
  description = "Subnet where the EC2 instance will be launched"
  type        = string
}

variable "security_group_ids" {
  description = "List of security group IDs"
  type        = list(string)
}

variable "instance_name" {
  description = "Name of the EC2 instance"
  type        = string
}

variable "tags" {
  description = "Additional tags for the EC2 instance"
  type        = map(string)
  default     = {}
}
```

### `modules/ec2-instance/main.tf`

``` hcl
resource "aws_instance" "this" {
  ami                    = var.ami_id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = var.security_group_ids

  tags = merge(
    var.tags,
    {
      Name = var.instance_name
    }
  )
}
```

`merge()` combines the additional tags with the `Name` tag. The `Name`
value is supplied separately for each module call.

### `modules/ec2-instance/outputs.tf`

``` hcl
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.this.id
}

output "public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = aws_instance.this.public_ip
}

output "private_ip" {
  description = "Private IP address of the EC2 instance"
  value       = aws_instance.this.private_ip
}
```

The module exposes the instance ID and IP addresses so the root module
can use them without needing to access the resource directly.

------------------------------------------------------------------------

## 3. Build a Custom Security Group Module

The Security Group module creates a security group in the VPC passed to
it. Its ingress rules are generated from a list of port numbers.

### `modules/security-group/variables.tf`

``` hcl
variable "vpc_id" {
  description = "VPC ID where the security group will be created"
  type        = string
}

variable "sg_name" {
  description = "Name of the security group"
  type        = string
}

variable "ingress_ports" {
  description = "List of ports allowed for inbound traffic"
  type        = list(number)
  default     = [22, 80]
}

variable "tags" {
  description = "Additional tags for the security group"
  type        = map(string)
  default     = {}
}
```

### `modules/security-group/main.tf`

``` hcl
resource "aws_security_group" "this" {
  name   = var.sg_name
  vpc_id = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_ports

    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = var.tags
}
```

### `modules/security-group/outputs.tf`

``` hcl
output "sg_id" {
  description = "ID of the security group"
  value       = aws_security_group.this.id
}
```

### Understanding the `dynamic` block

A dynamic block generates repeated nested configuration blocks.

Here:

-   `for_each` loops through the `ingress_ports` list.
-   `ingress.value` represents the current port.
-   `content` defines the ingress rule generated for that port.

For example, passing `[22, 80, 443]` generates three ingress rules.

A dynamic block does not create separate Terraform resources. It
generates repeated nested blocks inside the security group resource.

**Security note:** These rules allow inbound traffic from `0.0.0.0/0`.
This is suitable only for a controlled learning exercise. In a real
environment, SSH access on port 22 should be restricted to trusted IP
addresses or networks.

------------------------------------------------------------------------

## 4. Call the Custom Modules from the Root Module

The root module brings the infrastructure together.

It defines common tags and looks up an Amazon Linux AMI. It then creates
the network, calls the Security Group module once, and calls the EC2
module twice.

### Common tags and AMI lookup

``` hcl
locals {
  common_tags = {
    Project     = "TerraWeek"
    Environment = "Dev"
    ManagedBy   = "Terraform"
  }
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }

  filter {
    name   = "architecture"
    values = ["x86_64"]
  }
}
```

The AMI data source finds the latest matching Amazon Linux 2023 image in
the configured AWS region. This avoids hardcoding an AMI ID that may
differ between regions or become outdated.

### Network resources for the initial custom-module setup

For the initial version of the exercise, the VPC and subnet were defined
in the root module (or reused from the earlier VPC exercise). The
network included:

-   A VPC.
-   A public subnet.
-   An Internet Gateway.
-   A public route table with a default route to the Internet Gateway.
-   A route table association connecting the subnet to that route table.

The route is a nested `route` block inside `aws_route_table`, not a
separate Terraform resource.

### Security Group module call

``` hcl
module "web_sg" {
  source        = "./modules/security-group"
  vpc_id        = aws_vpc.main.id
  sg_name       = "terraweek-web-sg"
  ingress_ports = [22, 80, 443]
  tags          = local.common_tags
}
```

The root module passes the VPC ID, security group name, ingress ports
and common tags to the child module.

### Reuse the EC2 module for two instances

``` hcl
module "web_server" {
  source              = "./modules/ec2-instance"
  ami_id              = data.aws_ami.amazon_linux.id
  instance_type       = "t2.micro"
  subnet_id           = aws_subnet.public.id
  security_group_ids  = [module.web_sg.sg_id]
  instance_name       = "terraweek-web"
  tags                = local.common_tags
}

module "api_server" {
  source              = "./modules/ec2-instance"
  ami_id              = data.aws_ami.amazon_linux.id
  instance_type       = "t2.micro"
  subnet_id           = aws_subnet.public.id
  security_group_ids  = [module.web_sg.sg_id]
  instance_name       = "terraweek-api"
  tags                = local.common_tags
}
```

Both module calls use the same EC2 module source, but pass different
instance names.

They also use the same subnet and security group. Terraform creates two
separate EC2 instances because the module is called twice with two
different module labels: `web_server` and `api_server`.

### Root outputs

``` hcl
output "web_server_ip" {
  description = "Public IP of the web server"
  value       = module.web_server.public_ip
}

output "api_server_ip" {
  description = "Public IP of the API server"
  value       = module.api_server.public_ip
}
```

The root module accesses child-module outputs using:

``` text
module.<module_call_name>.<output_name>
```

For example, `module.web_server.public_ip` returns the public IP exposed
by the `web_server` module call.

------------------------------------------------------------------------

## 5. Replace the Hand-Written VPC with a Registry Module

Terraform also supports modules published in the public Terraform
Registry. Instead of maintaining all the VPC resources ourselves, we can
use a community-maintained module.

The VPC module used in this exercise:

``` hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "terraweek-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["ap-south-1a", "ap-south-1b"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.3.0/24", "10.0.4.0/24"]

  enable_nat_gateway   = false
  enable_dns_hostnames = true

  tags = local.common_tags
}
```

The provider region must match the Availability Zones used in the module
configuration. For these AZs, the AWS provider must use `ap-south-1`.

### Update the custom module calls

After replacing the hand-written VPC and subnet resources, the custom
module calls use outputs from the Registry module.

The Security Group module now receives the VPC ID from `module.vpc`:

``` hcl
module "web_sg" {
  source        = "./modules/security-group"
  vpc_id        = module.vpc.vpc_id
  sg_name       = "terraweek-web-sg"
  ingress_ports = [22, 80, 443]
  tags          = local.common_tags
}
```

Both EC2 module calls use the first public subnet created by the VPC
module:

``` hcl
subnet_id = module.vpc.public_subnets[0]
```

The rest of the EC2 module configuration remains the same.

### What does the Registry module create?

The VPC module manages the networking components based on its inputs and
defaults. This configuration creates a VPC with two public subnets and
two private subnets, along with the required networking resources and
route-table configuration.

NAT gateways are disabled using:

``` hcl
enable_nat_gateway = false
```

Therefore, the private subnets do not get outbound internet access
through a NAT gateway. Creating a private subnet does not automatically
give it internet connectivity.

### Where are Registry modules downloaded?

After running:

``` bash
terraform init
```

Terraform downloads the Registry module into:

``` text
.terraform/modules/
```

The `.terraform` directory is Terraform's local working directory. It
contains downloaded module code and other initialization data.

Do not edit the downloaded Registry module directly. Configure it
through its documented inputs in your root module.

### Hand-written VPC vs Registry VPC module

  -----------------------------------------------------------------------
  Hand-written VPC                    Registry VPC module
  ----------------------------------- -----------------------------------
  Individual AWS resources are        Networking is configured through a
  defined in the root configuration.  module call.

  You write and maintain the VPC,     The module manages the networking
  subnet, Internet Gateway, route     resources based on its inputs.
  table and association               
  configuration.                      

  Gives direct control over each      Provides reusable configuration
  resource definition.                with many supported options.

  More configuration to maintain in   Keeps the root configuration
  the root module.                    shorter and easier to read.
  -----------------------------------------------------------------------

**Resource-count comparison:** Record the actual counts from your
Terraform plans. The hand-written setup described above contains five
top-level networking resources: VPC, subnet, Internet Gateway, route
table and route table association. The default route is a nested block,
not a separate resource.

The Registry module's exact resource count depends on its version and
configuration. Use the resource additions shown in your own
`terraform plan` rather than assuming a fixed count.

  Configuration                                   Resource count
  ------------------------ -------------------------------------
  Hand-written VPC setup                                       5
  Registry VPC module        Add the actual count from your plan

------------------------------------------------------------------------

## 6. Module Versioning and Best Practices

### Version constraints

A Registry module should have an intentional version constraint.

Examples:

``` hcl
version = "5.1.0"
```

Selects exactly version `5.1.0`.

``` hcl
version = "~> 5.0"
```

Allows compatible versions from `5.0.0` up to, but not including,
`6.0.0`.

``` hcl
version = ">= 5.0, < 6.0"
```

Allows versions in the 5.x range.

Pinning a version makes module selection more predictable. A broad
constraint can allow Terraform to select a newer compatible release when
initializing or upgrading.

### Check for module upgrades

``` bash
terraform init -upgrade
```

This asks Terraform to check for newer provider and module versions that
satisfy the configured constraints.

An exact version constraint such as `5.1.0` will not be upgraded to
`5.2.0` unless the configuration is changed.

### Inspect Terraform state

``` bash
terraform state list
```

Module-managed resources appear with their module path in the resource
address. For example:

``` text
module.web_server.aws_instance.this
module.api_server.aws_instance.this
module.web_sg.aws_security_group.this
module.vpc....
```

The exact resource addresses under `module.vpc` depend on the Registry
module version and configuration.

The prefixes identify which module call owns each resource:

-   `module.web_server.` identifies resources in the web server module
    call.
-   `module.api_server.` identifies resources in the API server module
    call.
-   `module.web_sg.` identifies resources in the security group module
    call.
-   `module.vpc.` identifies resources managed by the VPC Registry
    module.

This is how Terraform represents module paths in its state.

### Destroy the lab infrastructure

After verifying the state and completing the exercise:

``` bash
terraform destroy
```

Review the destruction plan before confirming. This removes resources
managed by the current Terraform state, so make sure the state does not
include infrastructure you want to keep.

------------------------------------------------------------------------

## 7. Five Terraform Module Best Practices

### 1. Pin Registry module versions

Specify a version constraint for every Registry module. This makes
module selection more predictable and helps avoid unexpected changes
after a new release.

### 2. Keep modules focused

A module should have one clear responsibility. For example, keep EC2
creation separate from Security Group configuration instead of combining
unrelated infrastructure into one module.

### 3. Use input variables for configurable values

Expose values such as AMI IDs, instance types, subnet IDs and names
through variables. This makes a module reusable without changing its
internal resource code every time.

Use sensible defaults where appropriate. Not every constant needs to
become a variable.

### 4. Define useful outputs

Expose information that callers may need, such as resource IDs, IP
addresses or subnet IDs. Outputs allow the root module and other modules
to reference those values cleanly.

### 5. Add a README to every custom module

Document what the module creates, its required and optional inputs, its
outputs and a usage example. Good documentation makes a module easier to
understand, maintain and reuse.

------------------------------------------------------------------------

## 8. Verification and Cleanup

The main workflow for this exercise was:

``` bash
terraform fmt -recursive
terraform init
terraform validate
terraform plan
terraform apply
terraform state list
terraform destroy
```

-   `terraform fmt -recursive` formats Terraform files, including files
    inside child modules.
-   `terraform init` initializes the working directory and downloads
    required modules and providers.
-   `terraform validate` checks the configuration for syntax and
    internal consistency.
-   `terraform plan` previews the infrastructure changes.
-   `terraform apply` creates or updates the infrastructure.
-   `terraform state list` displays resources tracked in the current
    state.
-   `terraform destroy` removes the infrastructure managed by that
    state.

### AWS verification screenshot

Add the screenshot showing both EC2 instances in the AWS Console here.

Suggested location:

``` text
2026/day-65/images/day-65-ec2-instances.png
```

Then embed it in this document:

``` markdown
![Both EC2 instances running in AWS](images/day-65-ec2-instances.png)
```

The screenshot should show both instances and their different Name tags.
Only include a screenshot captured from your own AWS environment.

------------------------------------------------------------------------

## Key Takeaways

-   A Terraform module is a reusable collection of configuration files.
-   The root module calls child modules and passes their input values.
-   Module outputs allow callers to use information from resources
    inside a module.
-   One custom EC2 module can be called multiple times to create
    separate instances.
-   Dynamic blocks generate repeated nested configuration blocks from
    collections.
-   Registry modules let us reuse infrastructure code maintained and
    published by others.
-   Terraform records module paths in resource addresses in the state.
-   Version constraints help control which Registry module versions
    Terraform can select.

The biggest takeaway from this exercise: **write the infrastructure
logic once, then reuse it through module calls instead of duplicating
resource definitions.**

------------------------------------------------------------------------

## Submission

-   [ ] Save this file as `day-65-modules.md` in `2026/day-65/`.
-   [ ] Add the AWS EC2 screenshot under `2026/day-65/images/`.
-   [ ] Fill in the Registry VPC resource count from the actual
    Terraform plan.
-   [ ] Commit and push the changes to your fork.

Suggested commit message:

``` bash
git add 2026/day-65/
git commit -m "Complete Day 65 Terraform modules"
git push
```

------------------------------------------------------------------------

`#90DaysOfDevOps` `#TerraWeek` `#DevOpsKaJosh` `#TrainWithShubham`
