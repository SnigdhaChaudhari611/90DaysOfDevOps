# Day 66 -- Provision an EKS Cluster with Terraform Modules

## Objective

Provision an AWS VPC and an Amazon EKS cluster using reusable Terraform Registry modules, connect to the cluster with `kubectl`, deploy an Nginx workload, expose it through an AWS LoadBalancer Service, and destroy the infrastructure after verification.

The exercise demonstrates how Terraform can provision the cloud infrastructure while Kubernetes manages the workloads running on that infrastructure.

## Architecture

The deployment included:

- **VPC:** `terraweek-vpc`, using the CIDR range `10.0.0.0/16`
- **Availability Zones:** `ap-south-1a` and `ap-south-1b`
- **Public subnets:** `10.0.101.0/24` and `10.0.102.0/24`
- **Private subnets:** `10.0.1.0/24` and `10.0.2.0/24`
- **Internet Gateway:** Provides internet connectivity for resources in public subnets.
- **NAT Gateway:** A single NAT Gateway provides outbound internet access for resources in private subnets.
- **EKS:** `terraweek-eks`
- **Managed node group:** Two desired `t3.medium` worker nodes, with a minimum of one and maximum of three.
- **Kubernetes workload:** An Nginx Deployment with three replicas and a `LoadBalancer` Service.

> The EKS control plane is managed by AWS. The worker nodes are EC2 instances in the VPC's private subnets.

## Project structure

```text
terraform-eks/
├── providers.tf
├── vpc.tf
├── eks.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
└── k8s/
    └── nginx-deployment.yaml
```

## Terraform configuration

### 1. Providers -- `providers.tf`

The project used the AWS provider 5.x series and Kubernetes provider 2.x series.

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }

    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
}

provider "aws" {
  region = var.region
}
```

Terraform used local state for this exercise; no remote backend was configured. The Kubernetes provider was declared for potential Kubernetes resource management through Terraform, while the Nginx workload was deployed with `kubectl`.

### 2. Input variables -- `variables.tf`

```hcl
variable "region" {
  description = "AWS region where infrastructure will be created"
  type        = string
}

variable "cluster_name" {
  description = "Name of the EKS cluster"
  type        = string
  default     = "terraweek-eks"
}

variable "cluster_version" {
  description = "Kubernetes version for EKS"
  type        = string
  default     = "1.36"
}

variable "node_instance_type" {
  description = "EC2 instance type for EKS worker nodes"
  type        = string
  default     = "t3.medium"
}

variable "node_desired_count" {
  description = "Desired number of worker nodes"
  type        = number
  default     = 2
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}
```

### 3. Variable values -- `terraform.tfvars`

```hcl
region = "ap-south-1"
```

The exercise used Kubernetes version `1.36` rather than the task's original `1.31` value, and Amazon Linux 2023 for the managed nodes rather than the outdated Amazon Linux 2 AMI setting.

### 4. VPC module -- `vpc.tf`

The VPC was created using the `terraform-aws-modules/vpc/aws` Registry module.

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "terraweek-vpc"
  cidr = var.vpc_cidr

  azs = [
    "ap-south-1a",
    "ap-south-1b"
  ]

  public_subnets = [
    "10.0.101.0/24",
    "10.0.102.0/24"
  ]

  private_subnets = [
    "10.0.1.0/24",
    "10.0.2.0/24"
  ]

  enable_nat_gateway = true
  single_nat_gateway = true

  enable_dns_hostnames = true
  enable_dns_support   = true

  public_subnet_tags = {
    "kubernetes.io/role/elb" = "1"
  }

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = "1"
  }

  tags = {
    Environment = "dev"
    Project     = "TerraWeek"
    ManagedBy   = "Terraform"
  }
}
```

#### Why both public and private subnets?

Public subnets provide a route to the Internet Gateway and can host internet-facing resources. Private subnets are used for the EKS worker nodes so that the nodes do not need public IP addresses.

The NAT Gateway allows resources in private subnets to initiate outbound internet connections, for example to download container images. It does not make the private instances directly reachable from the internet.

A single NAT Gateway was used to reduce cost. This is a cost-versus-resilience trade-off: it is a single point of failure for outbound connectivity, and traffic from another Availability Zone may incur cross-AZ data transfer charges.

#### What do the subnet tags do?

- `kubernetes.io/role/elb = 1` identifies public subnets as candidates for internet-facing load balancers.
- `kubernetes.io/role/internal-elb = 1` identifies private subnets as candidates for internal load balancers.

These tags help AWS load-balancing integrations discover suitable subnets. The tags themselves do not create a load balancer.

### 5. EKS module -- `eks.tf`

The cluster was created using the `terraform-aws-modules/eks/aws` Registry module.

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = var.cluster_name
  cluster_version = var.cluster_version

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  cluster_endpoint_public_access = true

  enable_cluster_creator_admin_permissions = true

  eks_managed_node_groups = {
    terraweek_nodes = {
      ami_type       = "AL2023_x86_64_STANDARD"
      instance_types = [var.node_instance_type]

      min_size     = 1
      max_size     = 3
      desired_size = var.node_desired_count
    }
  }

  tags = {
    Environment = "dev"
    Project     = "TerraWeek"
    ManagedBy   = "Terraform"
  }
}
```

The EKS module receives the VPC ID and private subnet IDs from the VPC module. This is an example of one Terraform module's outputs being used as another module's inputs.

The managed node group uses EC2 instances to run Kubernetes workloads. AWS manages the managed node group lifecycle, while Kubernetes schedules Pods onto the available worker nodes.

The cluster endpoint was configured for public access to allow administration from the local machine. For a production environment, public endpoint access should be restricted to trusted IP ranges or replaced with an appropriate private access arrangement.

The cluster creator admin permission setting grants the IAM identity creating the cluster administrative access through EKS access management.

### 6. Outputs -- `outputs.tf`

```hcl
output "cluster_name" {
  description = "Name of the EKS cluster"
  value       = module.eks.cluster_name
}

output "cluster_endpoint" {
  description = "EKS Kubernetes API endpoint"
  value       = module.eks.cluster_endpoint
}

output "cluster_region" {
  description = "AWS region of the EKS cluster"
  value       = var.region
}

output "vpc_id" {
  description = "ID of the created VPC"
  value       = module.vpc.vpc_id
}
```

Outputs make useful values available after Terraform applies the configuration. They can be viewed with:

```bash
terraform output
```

## Provisioning the infrastructure

The configuration was formatted, initialized, validated and planned before applying:

```bash
terraform fmt
terraform init
terraform validate
terraform plan
terraform apply
```

The plan was reviewed before approval. Terraform then provisioned the VPC, networking resources, EKS control plane, IAM resources, security groups and managed node group.

**Total resources created:** `[Enter the exact number from your terraform apply output]`

Do not estimate this number. Use the final `Apply complete! Resources: X added, Y changed, Z destroyed.` line from the actual run.

### Apply screenshot

Add the actual screenshot showing the successful Terraform apply output:

```text
screenshots/terraform-apply.png
```

![Terraform apply completed](screenshots/terraform-apply.png)

## Connecting kubectl to EKS

After the cluster was created, the local kubeconfig was updated:

```bash
aws eks update-kubeconfig \
  --name terraweek-eks \
  --region ap-south-1
```

This added the EKS cluster context to the local kubeconfig file. `kubectl` uses that configuration and AWS authentication to communicate with the Kubernetes API server.

The cluster connection and worker nodes were checked with:

```bash
kubectl cluster-info
kubectl get nodes
kubectl get pods -A
```

The worker nodes were verified in the `Ready` state, and the cluster's system Pods were inspected in the `kube-system` namespace.

### Node verification screenshot

Add the actual screenshot showing the nodes:

```text
screenshots/eks-nodes.png
```

![EKS worker nodes](screenshots/eks-nodes.png)

## Deploying Nginx

A Kubernetes manifest was created at `k8s/nginx-deployment.yaml`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-terraweek
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

The manifest was applied:

```bash
kubectl apply -f k8s/nginx-deployment.yaml
```

The Deployment created three Nginx Pods. The Service selected those Pods using the `app: nginx` label and requested an AWS load balancer.

The rollout and resources were checked using:

```bash
kubectl rollout status deployment/nginx-terraweek
kubectl get deployments
kubectl get pods -o wide
kubectl get svc nginx-service
```

The external hostname was obtained from the Service and opened in a browser to verify the Nginx welcome page.

> The exact AWS load-balancer type depends on the EKS service load-balancer integration and configuration. A `LoadBalancer` Service does not, by itself, guarantee that an Application Load Balancer will be created.

### Nginx deployment screenshot

Add the actual screenshot showing the Deployment, Pods and Service:

```text
screenshots/nginx-resources.png
```

![Nginx resources on EKS](screenshots/nginx-resources.png)

### Nginx browser screenshot

Add the actual browser screenshot showing the Nginx welcome page:

```text
screenshots/nginx-welcome.png
```

![Nginx welcome page](screenshots/nginx-welcome.png)

## Destroying the infrastructure

The Kubernetes resources were deleted first:

```bash
kubectl delete -f k8s/nginx-deployment.yaml
```

The LoadBalancer Service was allowed to finish deleting its AWS load balancer before the VPC was destroyed. This order matters because an AWS load balancer attached to the VPC can prevent dependent networking resources from being removed cleanly.

The Terraform-managed infrastructure was then destroyed:

```bash
terraform destroy
```

After reviewing the destroy plan, the operation was confirmed. Terraform removed the resources it managed, including the EKS cluster, managed node group, IAM resources, security groups and VPC networking resources.

Cleanup was checked in the AWS Console and with AWS CLI commands, including:

```bash
aws eks list-clusters --region ap-south-1

aws ec2 describe-nat-gateways \
  --region ap-south-1 \
  --filter Name=state,Values=available,pending,deleting

aws elbv2 describe-load-balancers \
  --region ap-south-1

aws elb describe-load-balancers \
  --region ap-south-1
```

The AWS Console was also checked for leftover EC2 instances, NAT Gateways, Elastic IPs, load balancers, EBS volumes and network interfaces. Unrelated resources in the account should not be removed.

**Destroy result:** `[Record the exact Terraform destroy summary here]`

## Key learnings

- Terraform Registry modules make it possible to provision complex infrastructure without writing every AWS resource individually.
- Module outputs can be passed directly into other modules. For example, the VPC module provides the VPC ID and private subnet IDs used by the EKS module.
- EKS separates the AWS-managed Kubernetes control plane from the EC2 worker nodes.
- Public and private subnet placement, routing, NAT Gateway configuration and subnet tags are important parts of EKS networking.
- Terraform provisions the underlying infrastructure; Kubernetes manages workloads such as Deployments, Pods and Services.
- A Kubernetes `LoadBalancer` Service can trigger AWS load-balancer provisioning.
- Infrastructure cleanup must account for resources created indirectly by Kubernetes, such as AWS load balancers.
- Cost management is part of operating cloud infrastructure. EKS, NAT Gateways, EC2 instances and load balancers can incur charges while they exist.

## Reflection: EKS vs kind/minikube

| kind / minikube | Amazon EKS |
|---|---|
| Runs Kubernetes locally using containers or local VMs | Runs Kubernetes on AWS |
| Useful for local development and learning Kubernetes objects | Useful for learning and operating managed Kubernetes in AWS |
| No AWS VPC, NAT Gateway or managed node group is required | Requires cloud networking, IAM and worker-node infrastructure |
| Local cluster lifecycle is relatively quick | Cluster provisioning and deletion take longer |
| Does not represent all cloud networking and IAM concerns | Introduces real cloud networking, identity, access and cost-management concerns |

Day 50 introduced running Kubernetes locally with kind. This exercise extended that knowledge to a managed Kubernetes environment on AWS. The Kubernetes objects are familiar, but the surrounding infrastructure and operational responsibilities are much broader.

## Cleanup reminder

This exercise was intended to be temporary. Always verify that the EKS cluster, node-group instances, NAT Gateway, Elastic IPs and load balancer have been removed after the exercise. AWS billing can continue for resources that remain allocated even when they are no longer being used.

---

**Day 66 complete -- EKS infrastructure provisioned with Terraform modules, an Nginx workload deployed and the environment destroyed after verification.**

`#90DaysOfDevOps` `#TerraWeek` `#DevOpsKaJosh` `#TrainWithShubham`
