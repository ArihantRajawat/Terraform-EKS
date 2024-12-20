### Overview: Creating a VPC and EKS Architecture with Terraform

This project demonstrates a step-by-step process to provision a **VPC** and deploy an **Amazon EKS Cluster** using **Terraform**. It includes configurations for subnets, routing, gateways, and Kubernetes worker nodes, emphasizing modularity and scalability for staging or production environments.

#### Key Features:
1. **Infrastructure Provisioning**:
   - Fully customized VPC with public and private subnets.
   - Internet Gateway for public access and NAT Gateway for secure private communication.
   - Route tables for both private and public subnet traffic management.

2. **EKS Deployment**:
   - EKS Cluster with private subnets to enhance security.
   - Role-based access controls (IAM roles) for EKS components.
   - Node groups for Kubernetes worker nodes with autoscaling capabilities.

3. **Environment Configuration**:
   - Centralized variables to manage multiple environments (e.g., `staging`, `production`).
   - Reusable components for flexibility and ease of updates.

4. **Modular Design**:
   - Separate files for VPC, subnets, routing, NAT, and EKS cluster ensure clarity and maintainability.

#### Benefits:
- **Scalability**: Easily adapt configurations to different regions or zones.
- **Security**: Secure private resources with NAT and IAM role-based policies.
- **Automation**: Streamlined infrastructure deployment with Terraform.

### Usage:
1. Clone the repository containing this configuration.
2. Update the `locals` in `var-local.tf` to match your environment.
3. Run `terraform init`, `terraform plan`, and `terraform apply` to deploy the architecture.
4. Verify the setup in the AWS Management Console.

This repository provides a robust foundation for deploying production-ready Kubernetes clusters on AWS.
