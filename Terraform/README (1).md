# Detailed Documentation for Creating a VPC and EKS Architecture using Terraform

## Prerequisites

1. **AWS CLI Installed and Configured**: Ensure the AWS CLI is installed and configured with proper credentials.
2. **Terraform Installed**: Install Terraform on your machine.
3. **IAM Permissions**: Ensure the user creating the infrastructure has permissions for managing VPC, EKS, IAM, and related AWS resources.

---

## Step 1: Define Local Variables (`var-local.tf`)

Local variables are used to centralize environment settings like `region`, `zones`, and `EKS` configurations.

```hcl
locals {
  env            = "staging"
  region         = "us-east-1"
  zone1          = "us-east-1a"
  zone2          = "us-east-1b"
  eks_name       = "demo"
  eks_version    = "1.31"
}
```

- **Purpose**: These variables allow consistent references throughout the Terraform files.
- **Key Variables**: Environment (`env`), AWS region (`region`), Availability Zones (`zone1`, `zone2`), EKS cluster name and version.

---

## Step 2: Create a VPC (`vpc.tf`)

Define the primary VPC for the architecture.

```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${local.env}-main"
  }
}
```

- **CIDR Block**: Specifies the IP range for the VPC.
- **DNS Settings**: Enable DNS support and hostnames for resources in the VPC.

---

## Step 3: Define Subnets (`subnet.tf`)

Create private and public subnets in two availability zones.

### Private Subnets

```hcl
resource "aws_subnet" "private_zone1" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.0.0/19"
  availability_zone = local.zone1

  tags = {
    Name                          = "${local.env}-private-${local.zone1}"
    "kubernetes.io/role/internal-elb" = "1"
    "kubernetes.io/cluster/${local.env}-${local.eks_name}" = "owned"
  }
}

resource "aws_subnet" "private_zone2" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.32.0/19"
  availability_zone = local.zone2

  tags = {
    Name                          = "${local.env}-private-${local.zone2}"
    "kubernetes.io/role/internal-elb" = "1"
    "kubernetes.io/cluster/${local.env}-${local.eks_name}" = "owned"
  }
}
```

### Public Subnets

```hcl
resource "aws_subnet" "public_zone1" {
  vpc_id                 = aws_vpc.main.id
  cidr_block             = "10.0.64.0/19"
  availability_zone      = local.zone1
  map_public_ip_on_launch = true

  tags = {
    Name                          = "${local.env}-public-${local.zone1}"
    "kubernetes.io/role/elb"     = "1"
    "kubernetes.io/cluster/${local.env}-${local.eks_name}" = "owned"
  }
}

resource "aws_subnet" "public_zone2" {
  vpc_id                 = aws_vpc.main.id
  cidr_block             = "10.0.96.0/19"
  availability_zone      = local.zone2
  map_public_ip_on_launch = true

  tags = {
    Name                          = "${local.env}-public-${local.zone2}"
    "kubernetes.io/role/elb"     = "1"
    "kubernetes.io/cluster/${local.env}-${local.eks_name}" = "owned"
  }
}
```

- **Private Subnets**: Used for internal resources.
- **Public Subnets**: Allow public-facing resources to communicate over the internet.

---

## Step 4: Set Up Routing (`routes.tf`)

Define routing for private and public subnets.

### Private Route Table

```hcl
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat.id
  }

  tags = {
    Name = "${local.env}-private"
  }
}
```

### Public Route Table

```hcl
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "${local.env}-public"
  }
}
```

### Route Table Associations

```hcl
resource "aws_route_table_association" "private_zone1" {
  subnet_id      = aws_subnet.private_zone1.id
  route_table_id = aws_route_table.private.id
}
resource "aws_route_table_association" "public_zone1" {
  subnet_id      = aws_subnet.public_zone1.id
  route_table_id = aws_route_table.public.id
}
```

- **Private Routing**: Private subnets route traffic via the NAT Gateway.
- **Public Routing**: Public subnets use the Internet Gateway for external communication.

---

## Step 5: Create a NAT Gateway (`nat.tf`)

Allocate an Elastic IP and create a NAT Gateway for private subnets.

```hcl
resource "aws_eip" "nat" {
  domain = "vpc"

  tags = {
    Name = "${local.env}-nat"
  }
}

resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public_zone1.id

  tags = {
    Name = "${local.env}-nat"
  }
}
```

---

## Step 6: Create an Internet Gateway (`igw.tf`)

```hcl
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${local.env}-igw"
  }
}
```

- **Purpose**: Allows public resources to access the internet.

---

## Step 7: Deploy the EKS Cluster (`eks_master.tf`)

### IAM Role for EKS Cluster

```hcl
resource "aws_iam_role" "eks" {
  name = "${local.env}-${local.eks_name}-eks-cluster"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Action = "sts:AssumeRole",
      Effect = "Allow",
      Principal = { Service = "eks.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "eks" {
  role       = aws_iam_role.eks.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}
```

### EKS Cluster Resource

```hcl
resource "aws_eks_cluster" "eks" {
  name     = "${local.env}-${local.eks_name}"
  role_arn = aws_iam_role.eks.arn
  version  = local.eks_version

  vpc_config {
    subnet_ids = [
      aws_subnet.private_zone1.id,
      aws_subnet.private_zone2.id
    ]
  }
}
```

- **Role Attachment**: Grants permissions for the EKS cluster.
- **Cluster Configuration**: Includes private subnet IDs for the worker nodes.

---

## Step 8: Add EKS Worker Nodes (`eks_nodes.tf`)

### IAM Role for Worker Nodes

```hcl
resource "aws_iam_role" "nodes" {
  name = "${local.env}-${local.eks_name}-eks-nodes"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Action = "sts:AssumeRole",
      Effect = "Allow",
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })
}

resource "aws_eks_node_group" "general" {
  cluster_name    = aws_eks_cluster.eks.name
  version         = local.eks_version
  node_group_name = "general"
  node_role_arn   = aws_iam_role.nodes.arn

  subnet_ids = [
    aws_subnet.private_zone1.id,
    aws_subnet.private_zone2.id
  ]

  scaling_config {
    desired_size = 1
    max_size     = 3
    min_size     = 1
  }
}
```

- **Worker Nodes**: Configure EC2 instances for the Kubernetes cluster.

---

## Conclusion

This Terraform configuration creates a complete VPC setup, routes, NAT Gateway, Internet Gateway, EKS cluster, and worker nodes. Each component is modularized for reusability and clarity.

