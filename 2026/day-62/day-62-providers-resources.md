# Day 62 -- Providers, Resources and Dependencies

## Overview

For Day 62 of the 90DaysOfDevOps challenge, I used Terraform to build an AWS networking stack and explored how Terraform manages resource dependencies.

The infrastructure included a VPC, public subnet, internet gateway, route table, route table association, security group, EC2 instance, and an S3 bucket for application logs.

I also explored the Terraform dependency graph, used `depends_on` to define an explicit dependency, and tested the `create_before_destroy` lifecycle rule by changing the EC2 AMI and reviewing the execution plan.

---

## Task 1: Explore the AWS Provider

I created a project directory named `terraform-aws-infra` and configured the AWS provider in `providers.tf`.

### `providers.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

I ran:

```bash
terraform init
```

Terraform installed AWS provider version **5.100.0**.

### What does `~> 5.0` mean?

The `~>` operator is called the pessimistic version constraint. It allows compatible updates within the specified version range.

| Constraint | Meaning |
|---|---|
| `~> 5.0` | Allows versions `5.0.0` and later 5.x versions, but not 6.0.0 or higher |
| `>= 5.0` | Allows version 5.0 and any later version, including major version 6 |
| `= 5.0.0` | Allows only the exact version 5.0.0 |

### What is `.terraform.lock.hcl`?

The `.terraform.lock.hcl` file records the selected provider version and its checksums.

It helps ensure that Terraform uses the same provider version across repeated initializations and across team members' environments. This makes provider installation more consistent and predictable.

---

## Task 2: Build a VPC from Scratch

I created the AWS networking resources using Terraform.

The VPC uses the CIDR block `10.0.0.0/16`, and the public subnet uses `10.0.1.0/24`.

The subnet is associated with an internet gateway through a route table containing a default route (`0.0.0.0/0`).

The five networking resources created were:

1. VPC
2. Public subnet
3. Internet gateway
4. Public route table
5. Route table association

Terraform successfully created all five resources.

---

## Task 3: Understand Implicit Dependencies

Terraform automatically identifies dependencies when one resource references an attribute of another resource.

For example:

```hcl
vpc_id = aws_vpc.main.id
```

This tells Terraform that the resource needs the VPC ID before it can be created.

### Implicit dependencies in this configuration

| Resource | References | Dependency |
|---|---|---|
| `aws_subnet.public` | `aws_vpc.main.id` | VPC |
| `aws_internet_gateway.igw` | `aws_vpc.main.id` | VPC |
| `aws_route_table.public` | `aws_vpc.main.id`, `aws_internet_gateway.igw.id` | VPC and internet gateway |
| `aws_route_table_association.public` | `aws_subnet.public.id`, `aws_route_table.public.id` | Subnet and route table |
| `aws_security_group.web` | `aws_vpc.main.id` | VPC |
| `aws_instance.web` | `data.aws_ami.amazon_linux_2.id`, `aws_subnet.public.id`, `aws_security_group.web.id` | AMI data source, subnet and security group |

### How does Terraform know what to create first?

Terraform reads the references in the configuration and builds a dependency graph. It creates resources only after the resources they depend on are ready.

For example, the VPC must exist before Terraform can create a subnet inside it.

If I tried to create a subnet without a valid VPC ID, AWS would not be able to place that subnet in a VPC. Terraform's dependency graph prevents this ordering problem when the resources are connected through references.

---

## Task 4: Add a Security Group and EC2 Instance

I added a security group inside the VPC with these rules:

- SSH traffic on port `22`
- HTTP traffic on port `80`
- All outbound traffic

I then created a `t2.micro` EC2 instance in the public subnet. The instance uses an Amazon Linux 2 AMI selected through an AWS AMI data source.

The instance is associated with the security group and configured to receive a public IP address.

### AMI data source

Instead of hardcoding an AMI ID, I used a data source to find the latest matching Amazon Linux 2 image published by Amazon in the configured region.

```hcl
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

  filter {
    name   = "architecture"
    values = ["x86_64"]
  }
}
```

The EC2 instance uses the selected AMI through:

```hcl
ami = data.aws_ami.amazon_linux_2.id
```

---

## Task 5: Explicit Dependencies with `depends_on`

I created an S3 bucket for application logs and added an explicit dependency on the EC2 instance:

```hcl
resource "aws_s3_bucket" "app_logs" {
  bucket = "terraweek-app-logs-snigdha-2026"

  depends_on = [aws_instance.web]

  tags = {
    Name = "TerraWeek-App-Logs"
  }
}
```

There is no direct attribute reference between the S3 bucket and EC2 instance. I used `depends_on` to tell Terraform that the bucket must be created after the EC2 instance.

### When would I use `depends_on` in real projects?

1. **IAM permissions:** A service may need an IAM policy or role attachment to be created before the service starts. If Terraform cannot identify that ordering through a direct reference, `depends_on` can make the dependency explicit.
2. **Service initialization:** An application or service may need another resource to be fully configured before it is created or deployed. An explicit dependency can help enforce that order when no direct attribute reference expresses it.

I would prefer normal resource references wherever possible and use `depends_on` only when a real dependency is not already visible to Terraform.

### Dependency graph

I generated the Terraform dependency graph using:

```bash
terraform graph | dot -Tpng > graph.png
```

The graph is also available as `dependency-graph.png` in my project directory.

![Terraform dependency graph](dependency-graph.png)

The graph shows how Terraform connects resources and determines their creation and destruction order.

---

## Task 6: Lifecycle Rules and Destroy

I added the following lifecycle block to the EC2 instance:

```hcl
lifecycle {
  create_before_destroy = true
}
```

To test it, I changed the EC2 AMI ID from the AMI originally selected by the data source to a different AMI ID:

```hcl
ami = "ami-096ef200a877f36af"
```

I then ran:

```bash
terraform fmt
terraform validate
terraform plan
```

Terraform reported that `aws_instance.web` must be replaced because changing the AMI forces an EC2 instance replacement.

The plan showed:

```text
+/- create replacement and then destroy

Plan: 1 to add, 0 to change, 1 to destroy.
```

This demonstrated the planned `create_before_destroy` ordering. I reviewed the plan but did not apply the replacement. I proceeded to destroy the existing infrastructure.

### Three lifecycle arguments

| Lifecycle argument | What it does | When to use it |
|---|---|---|
| `create_before_destroy` | Creates the replacement resource before deleting the existing one | When downtime should be reduced during resource replacement |
| `prevent_destroy` | Prevents Terraform from destroying a resource while this rule remains in its configuration | For important resources such as production databases or critical storage |
| `ignore_changes` | Tells Terraform to ignore changes to selected resource attributes after creation | When another system or team manages particular attributes |

### Destroying the infrastructure

After reviewing the replacement plan, I ran:

```bash
terraform destroy
```

Terraform displayed the resources that would be destroyed and asked for confirmation. I entered `yes`.

Terraform destroyed the managed resources in dependency-aware reverse order. For example, it removed the route table association before removing the route table, and removed resources inside the VPC before deleting the VPC itself.

The final output confirmed:

```text
Destroy complete! Resources: 8 destroyed.
```

This confirmed that all eight managed AWS resources were destroyed.

---

## Complete `main.tf`

The following is the consolidated configuration used for the challenge. The lifecycle block belongs inside the EC2 resource.

```hcl
# 1. Create the VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "TerraWeek-VPC"
  }
}

# 2. Create a public subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true

  tags = {
    Name = "TerraWeek-Public-Subnet"
  }
}

# 3. Create an Internet Gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "TerraWeek-IGW"
  }
}

# 4. Create a route table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "TerraWeek-Public-RT"
  }
}

# 5. Associate the route table with the subnet
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# Find the latest Amazon Linux 2 AMI for this region
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

  filter {
    name   = "architecture"
    values = ["x86_64"]
  }
}

# 6. Create a security group
resource "aws_security_group" "web" {
  name        = "TerraWeek-SG"
  description = "Allow SSH and HTTP traffic"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "Allow SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Allow HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "TerraWeek-SG"
  }
}

# 7. Create an EC2 instance
resource "aws_instance" "web" {
  ami                         = data.aws_ami.amazon_linux_2.id
  instance_type               = "t2.micro"
  subnet_id                   = aws_subnet.public.id
  vpc_security_group_ids      = [aws_security_group.web.id]
  associate_public_ip_address = true

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name = "TerraWeek-Server"
  }
}

# 8. Create an S3 bucket for application logs
resource "aws_s3_bucket" "app_logs" {
  bucket = "terraweek-app-logs-snigdha-2026"

  depends_on = [aws_instance.web]

  tags = {
    Name = "TerraWeek-App-Logs"
  }
}
```

---

## Key Learnings

- The AWS provider gives Terraform the ability to manage AWS resources.
- The provider version constraint controls which provider versions Terraform can install.
- The dependency lock file records the selected provider version and checksums.
- Resource references create implicit dependencies.
- `depends_on` defines an explicit dependency when Terraform cannot infer the required ordering.
- `terraform graph` helps visualize how resources depend on one another.
- Lifecycle rules control how Terraform handles resource changes and replacement.
- Terraform destroys resources in reverse dependency order.

**One important security note:** The challenge allowed SSH from `0.0.0.0/0`. In a real environment, SSH access should normally be restricted to trusted IP addresses rather than open to the entire internet.

---

## Submission

- [x] Created the AWS provider configuration.
- [x] Created the VPC networking stack.
- [x] Added the security group and EC2 instance.
- [x] Added an S3 bucket with an explicit dependency.
- [x] Generated the Terraform dependency graph.
- [x] Tested the EC2 replacement plan with `create_before_destroy`.
- [x] Destroyed all 8 managed AWS resources.

**Note:** Add your actual Terraform apply and AWS Console screenshots to this folder before pushing. The terminal output in my notes confirms the successful plan and final destroy, but screenshots should be captured from the actual run rather than recreated.

---

`#90DaysOfDevOps` `#TerraWeek` `#DevOpsKaJosh` `#TrainWithShubham`
