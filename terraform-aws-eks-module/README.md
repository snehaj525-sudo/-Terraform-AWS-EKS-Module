# terraform-aws-eks-module

A reusable Terraform module that provisions a production-style AWS EKS setup:
a VPC with public/private subnets across two AZs, a NAT gateway, IAM roles
scoped per the EKS documentation, an EKS control plane, and a managed node
group — the same shape of IaC used to stand up Kubernetes infrastructure
for microservices.

## What this provisions

- **Networking**: VPC, 2 public + 2 private subnets (one pair per AZ),
  internet gateway, NAT gateway, route tables and associations
- **IAM**: a cluster role (`AmazonEKSClusterPolicy`) and a node role
  (`AmazonEKSWorkerNodePolicy`, `AmazonEKS_CNI_Policy`,
  `AmazonEC2ContainerRegistryReadOnly`)
- **EKS**: the control plane (`aws_eks_cluster`) and a managed node group
  (`aws_eks_node_group`) running in the private subnets
- **Remote state**: configured for an S3 backend with DynamoDB locking
  (see `main.tf` — replace the bucket/table names with your own before use)

## Why it's structured this way

- Worker nodes sit in **private subnets**; only the NAT gateway and load
  balancers touch the public subnets — standard practice for not exposing
  nodes directly to the internet.
- IAM roles are split (cluster vs. node) and scoped to only the policies
  each actually needs, rather than one broad role.
- All resources are tagged consistently via `var.tags`, including the
  `kubernetes.io/cluster/<name>` tags EKS and the AWS Load Balancer
  Controller expect on subnets.

## Usage

```bash
# 1. Set up remote state backend (S3 bucket + DynamoDB table) first,
#    or point main.tf's backend block at your own.

# 2. Copy and fill in variables
cp terraform.tfvars.example terraform.tfvars

# 3. Initialize and apply
terraform init
terraform plan -var-file=terraform.tfvars
terraform apply -var-file=terraform.tfvars

# 4. Configure kubectl (printed as an output after apply)
aws eks update-kubeconfig --region ap-south-1 --name demo-eks
```

## File layout

```
.
├── main.tf                     # provider, backend, networking, IAM, EKS, node group
├── variables.tf                 # all configurable inputs with sane defaults
├── outputs.tf                   # cluster endpoint, VPC/subnet IDs, kubectl command
├── terraform.tfvars.example     # template for real values (not committed)
└── .gitignore                   # keeps state files and real tfvars out of git
```

## Notes

- Values here (bucket names, CIDRs, instance types) are illustrative —
  replace `REPLACE_ME` placeholders and the example CIDRs with your own
  before running against a real AWS account.
- This module intentionally keeps everything in one `main.tf` for
  readability as a portfolio piece; in a larger codebase this would be
  split into `network.tf`, `iam.tf`, and `eks.tf`.
