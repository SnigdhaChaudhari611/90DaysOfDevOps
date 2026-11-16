# Day 63: Variables, Outputs, Data Sources and Expressions

## Overview

On Day 62, the Terraform configuration could create AWS infrastructure,
but several values were hardcoded. That made the configuration harder to
reuse across regions and environments.

In Day 63, I refactored the configuration to make it more dynamic and
reusable using:

-   Input variables and `.tfvars` files
-   Terraform outputs
-   AWS data sources
-   Locals and common tags
-   Built-in functions
-   Conditional expressions

The goal was to make the same Terraform configuration work with
different input values instead of editing the resource definitions every
time.

------------------------------------------------------------------------

## Task 1: Extract Variables

### Why use variables?

Input variables allow values to be passed into a Terraform configuration
instead of writing them directly inside resources.

I created a `variables.tf` file with the following variables:

``` hcl
variable "region" {
  description = "AWS region for the infrastructure"
  type        = string
  default     = "us-east-1"
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "subnet_cidr" {
  description = "CIDR block for the public subnet"
  type        = string
  default     = "10.0.1.0/24"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "project_name" {
  description = "Name of the project"
  type        = string
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"
}

variable "allowed_ports" {
  description = "Ports allowed through the security group"
  type        = list(number)
  default     = [22, 80, 443]
}

variable "extra_tags" {
  description = "Additional tags to apply to resources"
  type        = map(string)
  default     = {}
}
```

The AWS provider uses the region variable:

``` hcl
provider "aws" {
  region = var.region
}
```

Resource definitions use references such as `var.vpc_cidr`,
`var.subnet_cidr`, `var.instance_type`, and `var.project_name` instead
of repeating fixed values.

### Five commonly used Terraform variable types

  Type       Meaning                        Example
  ---------- ------------------------------ ---------------------------
  `string`   Text value                     `"us-east-1"`
  `number`   Numeric value                  `3`
  `bool`     `true` or `false`              `true`
  `list`     Ordered collection of values   `[22, 80, 443]`
  `map`      Key-value pairs                `{ Environment = "dev" }`

Terraform also supports more specific collection and structural types,
such as `set`, `tuple`, `object`, and combinations of these types.

### Why does Terraform ask for `project_name`?

The `project_name` variable has no default value:

``` hcl
variable "project_name" {
  type = string
}
```

If no value is supplied through a variable file, environment variable,
or command-line argument, Terraform prompts for it when a value is
needed.

This is useful when a value must be explicitly provided rather than
silently using a default.

------------------------------------------------------------------------

## Task 2: Variable Files and Precedence

Instead of entering values every time, I created separate variable files
for different environments.

### `terraform.tfvars`

This file is loaded automatically by Terraform:

``` hcl
project_name  = "Terraweek"
environment   = "dev"
instance_type = "t2.micro"
```

### `prod.tfvars`

This file contains the production-specific values:

``` hcl
project_name  = "terraweek"
environment   = "prod"
instance_type = "t3.small"
vpc_cidr      = "10.1.0.0/16"
subnet_cidr   = "10.1.1.0/24"
```

### Commands used

Use the automatically loaded default variable file:

``` bash
terraform plan
```

Use the production variable file explicitly:

``` bash
terraform plan -var-file="prod.tfvars"
```

Override a value using the CLI:

``` bash
terraform plan -var="instance_type=t2.nano"
```

Set an environment variable:

``` bash
export TF_VAR_environment="staging"
terraform plan
```

### Variable precedence: lowest to highest

Terraform's effective variable precedence, from lowest to highest, is:

1.  Variable default values
2.  Environment variables such as `TF_VAR_environment`
3.  `terraform.tfvars` and `terraform.tfvars.json`
4.  Automatically loaded `*.auto.tfvars` and `*.auto.tfvars.json` files
5.  Variable files supplied with `-var-file`
6.  Command-line values supplied with `-var`

**Important:** The order in the task hint places `TF_VAR_*` environment
variables above variable files. That is not Terraform's actual
precedence. A value in `terraform.tfvars` overrides the same variable
supplied through `TF_VAR_*`.

When multiple files of the same automatically loaded category define the
same variable, Terraform processes them in lexical filename order, with
later values taking precedence.

### What I observed

I used `terraform.tfvars` for the normal development values and created
`prod.tfvars` for production values.

For the production test, I deliberately ran:

``` bash
terraform plan -var="environment=prod"
```

I used this command instead of `-var-file="prod.tfvars"` because the
production file also changes the VPC and subnet CIDRs. I wanted to test
the production environment condition without changing the existing
network CIDRs.

------------------------------------------------------------------------

## Task 3: Add Outputs

Outputs expose useful information from resources after Terraform creates
or updates them.

I created an `outputs.tf` file:

``` hcl
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "subnet_id" {
  description = "ID of the public subnet"
  value       = aws_subnet.public.id
}

output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.web.id
}

output "instance_public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = aws_instance.web.public_ip
}

output "instance_public_dns" {
  description = "Public DNS name of the EC2 instance"
  value       = aws_instance.web.public_dns
}

output "security_group_id" {
  description = "ID of the security group"
  value       = aws_security_group.web.id
}
```

### Commands used

Apply the configuration:

``` bash
terraform apply
```

Show all outputs:

``` bash
terraform output
```

Show one specific output:

``` bash
terraform output instance_public_ip
```

Show outputs in JSON format:

``` bash
terraform output -json
```

### Output verification

After applying the production environment condition, Terraform
displayed:

``` text
Apply complete! Resources: 0 added, 7 changed, 0 destroyed.

Outputs:

instance_id          = "i-06e5ec2b91268a91b"
instance_public_dns  = ""
instance_public_ip   = "100.52.189.67"
security_group_id    = "sg-0657a22bba5f404bb"
subnet_id            = "subnet-048f10de4ab9671bf"
vpc_id               = "vpc-03da1f382ad7ef3a2"
```

The public IP output was also checked against the EC2 instance details
using the AWS CLI.

``` bash
terraform output instance_public_ip
```

**Observation:** The public IP was returned correctly. The public DNS
output was empty in this apply, so it should not be treated as a valid
DNS value. VPC DNS hostname settings and the instance's public DNS
details can be checked separately if a public DNS name is required.

### Why are outputs useful?

Outputs make it easier to retrieve resource IDs, IP addresses, DNS
names, and other important values without manually searching for them in
the AWS console.

They are also useful when passing information between Terraform modules
or using Terraform in automation scripts.

------------------------------------------------------------------------

## Task 4: Use Data Sources

### What is a data source?

A data source allows Terraform to read information about existing
resources or information available from a provider.

Instead of hardcoding an AMI ID, I used the AWS provider to find the
latest matching Amazon Linux 2 AMI.

### Amazon Linux 2 AMI data source

The configuration uses:

``` hcl
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }
}
```

The EC2 resource uses the AMI returned by the data source:

``` hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = var.instance_type

  # Other EC2 configuration...
}
```

The data source searches for the most recent Amazon-owned image matching
the filters. The AMI ID can therefore differ by region.

### Availability Zones data source

I also added a data source to retrieve available Availability Zones:

``` hcl
data "aws_availability_zones" "available" {
  state = "available"
}
```

The subnet uses the first available AZ:

``` hcl
availability_zone = data.aws_availability_zones.available.names[0]
```

This avoids hardcoding an Availability Zone name.

### Resource vs data source

  -----------------------------------------------------------------------
  Resource                            Data source
  ----------------------------------- -----------------------------------
  Manages infrastructure through      Reads information from a provider.
  Terraform.                          

  Can create, update, or delete       Does not create, update, or delete
  managed infrastructure.             the object it reads.

  Declared using a `resource` block.  Declared using a `data` block.

  Example: `aws_instance.web`         Example:
                                      `data.aws_ami.amazon_linux_2`
  -----------------------------------------------------------------------

**In simple terms:** A resource manages something. A data source looks
up information about something.

------------------------------------------------------------------------

## Task 5: Use Locals for Dynamic Values

### What are locals?

Locals are named values defined inside a Terraform configuration. They
help avoid repeating the same expressions and make configurations easier
to maintain.

I used locals to create a common name prefix and a reusable set of tags:

``` hcl
locals {
  name_prefix = "${var.project_name}-${var.environment}"

  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

For example, if:

``` hcl
project_name = "Terraweek"
environment  = "dev"
```

Then:

``` hcl
local.name_prefix
```

evaluates to:

``` text
Terraweek-dev
```

### Reuse locals in resource tags

I used `merge()` to combine common tags with resource-specific tags.

VPC:

``` hcl
tags = merge(local.common_tags, {
  Name = "${local.name_prefix}-vpc"
})
```

Subnet:

``` hcl
tags = merge(local.common_tags, {
  Name = "${local.name_prefix}-subnet"
})
```

EC2 instance:

``` hcl
tags = merge(local.common_tags, {
  Name = "${local.name_prefix}-server"
})
```

The same pattern can be used for other resources with their own
resource-specific names.

**Note:** I kept the security group's actual `name` attribute as
`Terraweek-SG` to avoid replacing the existing security group. AWS
security group names cannot be renamed in place. Terraform treats a
change to this immutable attribute as a replacement, so changing its
capitalization or name can trigger a destroy-and-create plan. The
security group was therefore kept stable while the other resources used
the common naming pattern.

### Why use `merge()`?

`merge()` combines two or more maps into one map.

For example:

``` hcl
merge(
  {
    Environment = "dev"
    ManagedBy   = "Terraform"
  },
  {
    Name = "Terraweek-dev-server"
  }
)
```

The result contains all three tags.

If the same key appears in more than one map, the value from the later
map takes precedence.

### Validation and apply

``` bash
terraform fmt
terraform validate
terraform plan
terraform apply
```

The plan showed:

``` text
Plan: 0 to add, 7 to change, 0 to destroy.
```

The apply completed:

``` text
Apply complete! Resources: 0 added, 7 changed, 0 destroyed.
```

The existing resources were updated in place.

**Important:** Terraform loads all `.tf` files in the same directory as
one module. A local name must be defined only once across those files.
Locals can be kept in `main.tf` or `locals.tf`, but defining the same
local names in both files causes duplicate-definition errors.

------------------------------------------------------------------------

## Task 6: Built-in Functions and Conditional Expressions

### Practise functions using Terraform Console

Terraform Console provides an interactive environment for testing
expressions and functions.

Start it with:

``` bash
terraform console
```

### 1. String functions

**`upper()`**

Converts a string to uppercase.

``` hcl
upper("terraweek")
```

Result:

``` text
"TERRAWEEK"
```

**`join()`**

Combines a list of strings using a separator.

``` hcl
join("-", ["terra", "week", "2026"])
```

Result:

``` text
"terra-week-2026"
```

**`format()`**

Creates a formatted string using placeholders.

``` hcl
format("arn:aws:s3:::%s", "my-bucket")
```

Result:

``` text
"arn:aws:s3:::my-bucket"
```

### 2. Collection functions

**`length()`**

Returns the number of elements in a collection.

``` hcl
length(["a", "b", "c"])
```

Result:

``` text
3
```

**`lookup()`**

Retrieves a value from a map using its key.

``` hcl
lookup({dev = "t2.micro", prod = "t3.small"}, "dev")
```

Result:

``` text
"t2.micro"
```

**`toset()`**

Converts a list into a set. Sets contain unique values, so duplicates
are removed.

``` hcl
toset(["a", "b", "a"])
```

Result:

``` text
toset([
  "a",
  "b",
])
```

### 3. Networking function

**`cidrsubnet()`**

Calculates a subnet CIDR block from a larger network CIDR.

``` hcl
cidrsubnet("10.0.0.0/16", 8, 1)
```

Result:

``` text
"10.0.1.0/24"
```

The arguments mean:

-   `10.0.0.0/16`: The parent network.
-   `8`: The number of additional prefix bits.
-   `1`: The subnet number to calculate.

Exit the console:

``` hcl
exit
```

### Five useful Terraform functions

  -----------------------------------------------------------------------
  Function                What it does            Where it is useful
  ----------------------- ----------------------- -----------------------
  `upper()`               Converts text to        Standardizing names or
                          uppercase.              labels.

  `join()`                Combines strings using  Building names and
                          a separator.            paths.

  `lookup()`              Gets a value from a map Selecting
                          using a key.            environment-specific
                                                  values.

  `merge()`               Combines multiple maps. Applying common and
                                                  resource-specific tags.

  `cidrsubnet()`          Calculates subnet CIDR  Planning VPC and subnet
                          blocks.                 address ranges.
  -----------------------------------------------------------------------

I also practised `format()`, `length()`, and `toset()` in Terraform
Console.

### Conditional expressions

A conditional expression allows Terraform to choose a value based on
whether a condition is true or false.

Syntax:

``` hcl
condition ? value_if_true : value_if_false
```

I used this expression for the EC2 instance type:

``` hcl
instance_type = var.environment == "prod" ? "t3.small" : "t2.micro"
```

It means:

-   If the environment is `prod`, use `t3.small`.
-   Otherwise, use `t2.micro`.

This makes the EC2 instance type environment-aware.

### Test the production condition

First, I checked the proposed changes:

``` bash
terraform plan -var="environment=prod"
```

The plan showed the EC2 instance type changing from `t2.micro` to
`t3.small`, along with environment-related tag updates.

I then applied the change:

``` bash
terraform apply -var="environment=prod"
```

Terraform reported:

``` text
Apply complete! Resources: 0 added, 7 changed, 0 destroyed.
```

I verified the instance type using the AWS CLI:

``` bash
aws ec2 describe-instances \
  --instance-ids "$(terraform output -raw instance_id)" \
  --query 'Reservations[0].Instances[0].InstanceType' \
  --output text
```

Result:

``` text
t3.small
```

The instance type was successfully updated, and the EC2 instance ID
remained the same.

**Observation:** Changing an EC2 instance type can require AWS to stop
and start the instance. If the instance uses an automatically assigned
public IPv4 address rather than an Elastic IP, that address can change
after a stop/start operation.

------------------------------------------------------------------------

## Difference Between Variable, Local, Output and Data Source

These four Terraform concepts serve different purposes:

  ----------------------------------------------------------------------------------
  Concept                 Purpose                 Example
  ----------------------- ----------------------- ----------------------------------
  Variable                Accepts input values    `var.region`
                          from the user or        
                          variable files.         

  Local                   Stores a reusable value `local.name_prefix`
                          or expression inside    
                          the configuration.      

  Output                  Exposes useful values   `output "instance_id"`
                          after Terraform         
                          operations.             

  Data source             Reads information from  `data.aws_ami.amazon_linux_2.id`
                          an existing object or   
                          provider.               
  ----------------------------------------------------------------------------------

A simple way to remember them:

-   **Variable:** What value should Terraform use?
-   **Local:** What value can I calculate and reuse?
-   **Output:** What information should Terraform show?
-   **Data source:** What information should Terraform look up?

------------------------------------------------------------------------

## Key Takeaways

-   Variables remove hardcoded inputs and make configurations reusable.
-   `.tfvars` files make it easier to manage development and production
    values.
-   Variable precedence determines which value Terraform uses when the
    same variable is defined in multiple places.
-   Outputs expose useful information such as resource IDs and public IP
    addresses.
-   Data sources retrieve information dynamically, such as the latest
    matching AMI and available Availability Zones.
-   Locals help reuse expressions and maintain consistent resource
    naming and tags.
-   Built-in functions make it easier to work with strings, maps, lists,
    and network ranges.
-   Conditional expressions allow configuration values to change based
    on the environment.
-   `terraform plan` should always be reviewed before applying
    infrastructure changes.

## Tasks Completed

-   [x] Extracted input variables into `variables.tf`.
-   [x] Created development and production variable files.
-   [x] Practised variable overrides and precedence.
-   [x] Added outputs for the VPC, subnet, EC2 instance, public IP,
    public DNS, and security group.
-   [x] Replaced the hardcoded AMI with an AWS AMI data source.
-   [x] Retrieved available Availability Zones dynamically.
-   [x] Used locals for naming and common tags.
-   [x] Practised Terraform built-in functions.
-   [x] Added an environment-based conditional expression.
-   [x] Applied the production condition and verified the EC2 instance
    type.

## Commands Reference

``` bash
terraform fmt
terraform validate
terraform plan
terraform plan -var-file="prod.tfvars"
terraform plan -var="instance_type=t2.nano"
terraform apply
terraform output
terraform output instance_public_ip
terraform output -json
terraform console
```

------------------------------------------------------------------------

## Submission

Save this file as:

``` text
2026/day-63/day-63-variables-outputs.md
```

Then commit and push it to your fork:

``` bash
git add 2026/day-63/day-63-variables-outputs.md
git commit -m "Add Day 63 Terraform variables and expressions"
git push
```

## Learn in Public

Made my Terraform configuration more dynamic and reusable today.

Worked with variables, environment-specific `.tfvars` files, outputs,
data sources, locals, built-in functions, and conditional expressions.

One useful takeaway: Terraform can select different infrastructure
values based on the environment, without needing separate resource
configurations.

#90DaysOfDevOps #TerraWeek #DevOpsKaJosh #TrainWithShubham
