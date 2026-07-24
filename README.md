# ☸️ EKS-Terraform-GitHub-Actions

### Production-Ready Amazon EKS Infrastructure, Fully Codified in Terraform

[![Terraform](https://img.shields.io/badge/Terraform-1.9.3-844FBA?logo=terraform&logoColor=white)](https://developer.hashicorp.com/terraform)
[![AWS](https://img.shields.io/badge/AWS-EKS-FF9900?logo=amazonaws&logoColor=white)](https://aws.amazon.com/eks/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.36-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![GitHub Actions](https://img.shields.io/badge/CI%2FCD-GitHub%20Actions-2088FF?logo=githubactions&logoColor=white)](https://github.com/features/actions)
[![Jenkins](https://img.shields.io/badge/CI%2FCD-Jenkins-D24939?logo=jenkins&logoColor=white)](https://www.jenkins.io/)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

> A modular Terraform codebase that stands up a real-world AWS networking layer and a fully working Amazon EKS cluster — private/public subnets across 3 AZs, on-demand + spot node groups, IRSA via OIDC — automated through **both** a GitHub Actions workflow and a Jenkins pipeline.

---

## 🎬 Architecture & Workflow Demo

![EKS Terraform Pipeline Demo](assets/Presentation1.gif)

*End-to-end walkthrough: pipeline trigger → Terraform init/validate/plan/apply → EKS cluster and node groups coming up on AWS.*

---

## 💡 Why This Project

Most "Terraform + EKS" repos on GitHub stop at `terraform apply` with hardcoded values and call it done. This one is built the way infrastructure actually gets run in a team:

- **Modular, reusable Terraform** — VPC, IAM, and EKS live in a shared `modules/` layer, consumed by an environment-specific root module. Adding a `staging` or `prod` environment is a new `.tfvars` file, not a rewrite.
- **Remote state with locking** — S3 backend + DynamoDB, so this isn't a "runs on my laptop" project.
- **Cost-aware compute** — separate On-Demand and Spot managed node groups, so workloads can be scheduled onto cheaper capacity where it's safe to do so.
- **Secure identity, not static keys** — an OIDC provider is wired into the cluster for IAM Roles for Service Accounts (IRSA), the standard way pods should assume AWS permissions.
- **Two working CI/CD paths** — a `workflow_dispatch`-triggered GitHub Actions pipeline *and* a parameterized Jenkins pipeline, both running `fmt → init → validate → plan/apply/destroy`. Shows comfort with the two most common CI/CD tools teams actually use.
- **Honest about tradeoffs** — see [Security Notes](#-security-notes) below. Nothing is swept under the rug.

## 🧱 What Gets Built

| Layer | Resources |
|---|---|
| **Networking** | Custom VPC, 3 public + 3 private subnets across AZs, IGW, NAT Gateway, route tables |
| **Compute** | EKS control plane + 2 managed node groups (On-Demand & Spot) |
| **Identity** | IAM roles for cluster/nodes, OIDC provider for IRSA |
| **Add-ons** | `vpc-cni`, `coredns`, `kube-proxy`, `aws-ebs-csi-driver` |
| **State** | S3 backend with DynamoDB state locking |
| **CI/CD** | GitHub Actions (`workflow_dispatch`) + Jenkins, both with `plan`/`apply`/`destroy` |
| **License** | Apache 2.0 |

## 🏗️ Architecture

```
                              ┌────────────────────────────┐
                              │            VPC              │
                              │        10.16.0.0/16         │
                              └────────────┬────────────────┘
                                           │
                 ┌─────────────────────────┼─────────────────────────┐
                 │                         │                         │
         ┌───────▼────────┐        ┌───────▼────────┐        ┌───────▼────────┐
         │ Public Subnet 1 │        │ Public Subnet 2 │        │ Public Subnet 3 │
         │  (us-east-1a)   │        │  (us-east-1b)   │        │  (us-east-1c)   │
         └───────┬─────────┘        └────────────────┘        └────────────────┘
                 │  NAT Gateway + IGW
                 │
   ┌─────────────┴────────────────────────────────────────────────────┐
   │                                                                    │
┌──▼──────────────────┐  ┌─────────────────────┐  ┌────────────────────▼─┐
│ Private Subnet 1     │  │ Private Subnet 2     │  │ Private Subnet 3      │
│  (us-east-1a)        │  │  (us-east-1b)        │  │  (us-east-1c)         │
│                      │  │                      │  │                       │
│    EKS Control Plane + Node Groups (On-Demand + Spot)                     │
└────────────────────────────────────────────────────────────────────────┘
```

## 📂 Repository Structure

```
.
├── .github/
│   └── workflows/
│       └── terraform.yml    # GitHub Actions: fmt → init → validate → plan/apply/destroy
├── Jenkinsfile              # Jenkins pipeline: init → validate → plan/apply/destroy
├── eks/
│   ├── backend.tf           # S3 + DynamoDB remote state backend
│   ├── main.tf              # Root module wiring — calls ../modules
│   ├── variables.tf         # Root-level variable declarations
│   └── dev.tfvars           # Environment-specific values (dev)
├── modules/
│   ├── vpc.tf               # VPC, subnets, IGW, NAT GW, route tables, SG
│   ├── eks.tf               # EKS cluster, node groups, add-ons, OIDC provider
│   ├── iam.tf               # Cluster/node IAM roles, policies, OIDC role
│   ├── gather.tf            # Data sources (TLS cert, IAM policy document)
│   └── variables.tf         # Module input variable declarations
├── assets/
│   └── Presentation1.gif    # Architecture / workflow demo (embedded above)
├── LICENSE                  # Apache 2.0
└── .gitignore
```

## 🛠️ Tech Stack

`Terraform` `AWS EKS` `AWS VPC/IAM/EC2` `Kubernetes` `GitHub Actions` `Jenkins` `HCL` `OIDC / IRSA`

## 🚀 Getting Started

### Prerequisites

- AWS account with permissions for VPC, EKS, IAM, and EC2
- [Terraform](https://developer.hashicorp.com/terraform/downloads) `~> 1.9.3` (CI) — root config pins `~> 1.15.8` locally
- `kubectl` and the AWS CLI
- An S3 bucket + DynamoDB table for remote state
- AWS credentials available as `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` (GitHub Actions secrets or Jenkins `aws-creds`)

### 1. Configure the backend

```hcl
# eks/backend.tf
backend "s3" {
  bucket         = "<your-tf-state-bucket>"
  region         = "us-east-1"
  key            = "eks/terraform.tfstate"
  dynamodb_table = "<your-lock-table>"
  encrypt        = true
}
```

### 2. Review environment variables

Adjust `eks/dev.tfvars` — VPC CIDR, subnet AZs, instance types, node group sizing, EKS version, add-ons.

### 3. Run it locally

```bash
cd eks
terraform init
terraform fmt -check -diff
terraform validate
terraform plan  -var-file=dev.tfvars
terraform apply -var-file=dev.tfvars

# when you're done
terraform destroy -var-file=dev.tfvars
```

### 4. Or trigger it via GitHub Actions

Go to **Actions → EKS-Creation-Using-Terraform → Run workflow**, and provide:

| Input | Example | Description |
|---|---|---|
| `tfvars_file` | `dev.tfvars` | Path to the `.tfvars` file to use |
| `action` | `plan` / `apply` / `destroy` | Terraform action to run |

The workflow runs `fmt -check → init → validate → plan/apply/destroy` inside the `eks/` directory, with a Terraform plugin cache to speed up repeat runs. Requires `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` set as repo secrets under the `production` environment.

### 5. Or trigger it via Jenkins

The `Jenkinsfile` exposes the same two parameters (`Environment`, `Terraform_Action`) through stages: **Preparing → Git Pulling → Init → Validate → Action**, under an `aws-creds` binding via `withAWS`.

## ⚙️ Key Configuration

| Variable | Example | Purpose |
|---|---|---|
| `vpc-cidr-block` | `10.16.0.0/16` | VPC address space |
| `cluster-version` | `1.36` | EKS Kubernetes version |
| `endpoint-private-access` / `endpoint-public-access` | `true` / `false` | EKS API endpoint exposure |
| `ondemand_instance_types` | `["t3a.medium"]` | On-demand node group instances |
| `spot_instance_types` | `["c5a.large", "m5a.large", ...]` | Spot node group instance pool |
| `desired/min/max_capacity_*` | `1` / `1` / `5` | Node group scaling bounds |
| `addons` | `vpc-cni`, `coredns`, `kube-proxy`, `aws-ebs-csi-driver` | EKS-managed add-ons |

## 🔒 Security Notes

- The EKS control-plane security group still allows inbound `443` from `0.0.0.0/0` — flagged in-code as a placeholder ("Allow 443 from Jump Server only"); restrict this to a bastion/VPN CIDR before real use.
- The sample OIDC IAM policy grants broad S3 actions for demonstration purposes — scope to least privilege beyond a sandbox environment.
- `.terraform/` and `*.tfstate` are gitignored — never commit state files or credentials.
- AWS credentials for both CI paths are pulled from secret stores (GitHub Actions secrets / Jenkins credential binding), not hardcoded — keep it that way.

## 🗺️ Roadmap

- [x] Add a GitHub Actions workflow alongside the Jenkins pipeline
- [x] Add a project LICENSE (Apache 2.0)
- [ ] Restrict the EKS control-plane security group to a specific CIDR
- [ ] Bootstrap ArgoCD for GitOps-style app delivery onto the cluster
- [ ] Scope the OIDC IAM policy per-workload instead of a shared broad policy
- [ ] Add `staging`/`prod` `.tfvars` alongside `dev.tfvars`

## 👤 Author

**Arpit Kumar Mishra**
[GitHub](https://github.com/Arpit03-gits) · [LinkedIn](https://www.linkedin.com/in/arpit-mishra-341203305)

Pre-Final year B.Tech IT student focused on Cloud/DevOps — building production-style infrastructure projects (AWS multi-tier architectures, CI/CD cost optimization, Kubernetes/GitOps) as part of an active internship search.

## 📄 License

Licensed under the [MIT](LICENSE).
