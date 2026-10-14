## Terraform S3 State Locking Without DynamoDB: Building a VPC + EKS Cluster on AWS
If you have ever run terraform apply from your laptop and then wondered what happens when a teammate does the same thing, this post is for you.

In this walkthrough we will build, from scratch and with Terraform:

A multi-AZ VPC with public and private subnets
An Amazon EKS cluster with a managed node group
A remote backend on S3 with state locking, so a whole team can safely share one state file

What we are building
<img width="1997" height="1212" alt="architecture-diagram" src="https://github.com/user-attachments/assets/418fc3da-9c81-4f19-b15b-edb94ce17a14" />

The flow is simple:

1. The Terraform CLI reads the root configuration (`main.tf`).
2. State is stored in, and locked through, an **S3 bucket**.
3. The `vpc` module creates a `10.0.0.0/16` network spread over **three Availability Zones**, each with a public and a private subnet.
4. The `eks` module creates the EKS control plane and a managed node group that runs in the **private subnets**.

## Prerequisites

| Tool | Why you need it |
|------|-----------------|
| Terraform **1.10+** | The backend uses `use_lockfile`, native S3 locking, which was introduced in 1.10 |
| AWS CLI v2 | Authentication and `aws eks update-kubeconfig` |
| kubectl | Talking to the cluster once it exists |
| An AWS account | With permissions to create VPC, EKS, IAM and S3 resources |

Check your setup:

```bash
terraform version
aws --version
kubectl version --client
aws sts get-caller-identity
```

---

## Project structure

```text
aws-terraform-devops-project/
├── backend/               # bootstraps the S3 bucket used for remote state
├── modules/
│   ├── vpc/               # VPC, public + private subnets
│   └── eks/               # EKS cluster + managed node groups
├── main.tf                # providers, S3 backend, module calls
├── variable.tf            # all input variables with defaults
├── output.tf              # cluster endpoint, cluster name, VPC ID
├── .terraform.lock.hcl    # provider version lock file
└── README.md
```
The idea is a clean separation: the **root module** only wires things together, and each **child module** owns one piece of infrastructure.

---

## Understanding the root module

### `main.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket       = "aws-terraform-state-s3-bucket-nr"
    key          = "terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true # native S3 locking (.tflock file)
    encrypt      = true
  }
}

provider "aws" {
  region = var.region
}

module "vpc" {
  source               = "./modules/vpc"
  vpc_cidr             = var.vpc_cidr
  availability_zones   = var.availability_zones
  private_subnet_cidrs = var.private_subnet_cidrs
  public_subnet_cidrs  = var.public_subnet_cidrs
  cluster_name         = var.cluster_name
}

module "eks" {
  source          = "./modules/eks"
  cluster_name    = var.cluster_name
  cluster_version = var.cluster_version
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnet_ids
  node_groups     = var.node_groups
}
```

Two things worth noticing:

- The EKS module receives `module.vpc.private_subnet_ids`, so worker nodes are never placed in public subnets. Terraform also uses this reference to work out that the VPC must be created **before** the cluster.
- The backend block turns on `encrypt = true` and `use_lockfile = true`. More on that below.

### `variable.tf` (defaults)

| Variable | Default |
|----------|---------|
| `region` | `us-east-1` |
| `vpc_cidr` | `10.0.0.0/16` |
| `availability_zones` | `us-east-1a`, `us-east-1b`, `us-east-1c` |
| `private_subnet_cidrs` | `10.0.1.0/24`, `10.0.2.0/24`, `10.0.3.0/24` |
| `public_subnet_cidrs` | `10.0.4.0/24`, `10.0.5.0/24`, `10.0.6.0/24` |
| `cluster_name` | `my-eks-cluster` |
| `cluster_version` | `1.30` |
| `node_groups` | one group `general`: `t3.medium`, `ON_DEMAND`, min 1 / desired 2 / max 4 |

### `output.tf`

```hcl
output "cluster_endpoint" { value = module.eks.cluster_endpoint }
output "cluster_name"     { value = module.eks.cluster_name }
output "vpc_id"           { value = module.vpc.vpc_id }
```

---

## Why remote state and locking matter

Terraform's **state file** is its memory. It records which real resources belong to your configuration. If that file lives only on your laptop, a teammate who runs the same code has no idea what already exists, and Terraform will happily try to create a second copy of everything.

Storing state in **S3** fixes the "single source of truth" problem. **State locking** fixes the "two people applying at once" problem: while one `apply` is running, everyone else is blocked.

Historically, locking on S3 required a separate DynamoDB table. With `use_lockfile = true`, Terraform writes a lock file next to the state object in S3 itself, so you no longer need DynamoDB for this. That is one less resource to manage.

> **Heads up:** S3 bucket names are globally unique. Change `bucket` in `main.tf` to your own name before you run anything.

---

## Step-by-step: deploy it yourself

### Step 1: Clone the repo

```bash
git clone https://github.com/nithish1991/aws-terraform-devops-project.git
cd aws-terraform-devops-project
```

### Step 2: Authenticate with AWS

```bash
aws configure
#### input your security credentials #####
aws sts get-caller-identity
```


### Step 3: Create the remote backend bucket

The state bucket has to exist before the root module can use it, so it is created first from the `backend/` folder.

```bash
cd backend
terraform init
terraform plan
terraform apply
cd ..
```

### Step 4: Initialise the root module

```bash
terraform init
```

Look for these lines in the output:

```text
Successfully configured the backend "s3"!
Terraform has been successfully initialized!
```

### Step 5: Format and validate

```bash
terraform fmt -recursive
terraform validate
```

You should see `Success! The configuration is valid.`

### Step 6: Review the plan

```bash
terraform plan -out=tfplan
```

Read the plan carefully. You should see resources from both `module.vpc` and `module.eks` marked with a `+`.
### Step 7: Apply

```bash
terraform apply tfplan
```

EKS control planes take a while to come up, so expect this to run for **10 to 15 minutes**. When it finishes you will see `Apply complete!` followed by the outputs:

```text
cluster_endpoint = "https://XXXXXXXX.gr7.us-east-1.eks.amazonaws.com"
cluster_name     = "my-eks-cluster"
vpc_id           = "vpc-xxxxxxxxxxxxxxxxx"
```

### Step 8: Connect to the cluster

```bash
aws eks update-kubeconfig --region us-east-1 --name my-eks-cluster
kubectl get nodes
```

You should see your worker nodes in `Ready` state.

### Step 9: Verify the state lives in S3

```bash
aws s3 ls s3://<your-bucket-name>/
```

You will see `terraform.tfstate` in the bucket. While an operation is running you will also briefly see the lock file next to it.


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/syltcuds3www1epncx31.png)

Yes, we have successfully implemented production grade infrastructure provisioning via Terraform.

### Step 10: Clean up

EKS and EC2 nodes are billed while they run, so destroy the stack when you are done experimenting:

```bash
terraform destroy
```

## Wrap-up

You now have a repeatable way to create a multi-AZ network and an EKS cluster with Terraform, with state stored safely in S3 so your whole team can collaborate.

If this helped, please ⭐ **[the repo on GitHub](https://github.com/nithish1991/aws-terraform-devops-project)**, drop a ❤️ here, and share what you would add. Issues and pull requests are very welcome.

Happy provisioning! 🚀
