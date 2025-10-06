**Proposal: Standardized OpenTofu Infrastructure Automation Framework**

**Prepared By:** Vinay Kumar
**Purpose:** To define a unified and scalable structure for managing multi-client infrastructure using OpenTofu and GitHub Actions without dependency on Digger.

---

## 1. Objective

The goal of this proposal is to standardize the OpenTofu-based infrastructure automation process for multiple clients and environments. The new structure eliminates Digger dependency, simplifies deployment management, and ensures secure, scalable, and environment-isolated state management via S3 backends.

---

## 2. Current Challenges

- **Complex folder structure** under `opentofu/projects/xytech/client-name/environments/` makes onboarding slow.
- **Digger-based execution** introduces unnecessary coupling and lock conflicts.
- **Lack of standard documentation** around backend usage and environment segregation.
- **Inconsistent module tagging across clients and environments** — each module version is tagged differently per environment and client, leading to fragmented deployments, unpredictable behaviors, and loss of version control consistency.
- **Redundant workflows** per project and environment increase maintenance overhead.

---

## 3. Proposed Architecture

### Key Highlights

- One **common codebase** for Terraform configuration (`main.tf`, `providers.tf`, `versions.tf`).
- **Environment-specific variables and backend files** per client.
- **Dedicated statefile per environment** to ensure backend isolation and avoid state overlap.
- **Single GitHub Actions workflow** to handle all clients and environments dynamically.
- **Automatic plan** on Pull Request, and **manual apply** triggered via PR comment (`tofu apply`).

### Folder Structure

```
client1/
├── backend/
│ ├── dev.backend.tf
│ ├── stg.backend.tf
│ └── prd.backend.tf
│
├── envs/
│ ├── dev.tfvars
│ ├── stg.tfvars
│ └── prd.tfvars
│
├── main.tf
├── providers.tf
├── versions.tf
│
├── .github/
│ └── workflows/
│ ├── plan.yml
│ ├── apply.yml
│ └── validate.yml
│
└── README.md
```
---

## 4. Environment Backend Strategy

Each environment has its own backend configuration file stored under the client folder. Example for **Staging**:

```hcl
# backend/stg.backend.tf
bucket  = "org-infra-backend"
key     = "client1/stg/terraform.tfstate"
region  = "us-east-1"
encrypt = true
role_arn = "**************"
```


## 5. Common File Definitions

**providers.tf**

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 6.0.0"
    }
  }
}

provider "aws" {
  region = var.region
  assume_role {
    role_arn = var.role_arn
  }
}
```

**versions.tf**

```hcl
terraform {
  required_version = ">= 1.8.0"
}
```

**main.tf**

```hcl
module "network" {
  source = "git::https://github.com/org/main-modules.git//network/vpc?ref=main"
  name   = var.name
  cidr   = var.cidr
}
```

## 6. Architecture Diagram

```
Developer Commit (feature branch)
        ↓
Pull Request → GitHub Actions (plan.yml)
        ↓
Comment 'tofu apply' → Apply Workflow
        ↓
S3 Backend (State Isolation per env)
        ↓
AWS Infrastructure Deployment
```

---

## 5. Benefits

- ✅ Each environment has an isolated backend and statefile in S3.
- ✅ No dependency on Digger or custom orchestration tools.
- ✅ Common reusable workflows for all clients.
- ✅ Easy onboarding for new developers (single command per env).
- ✅ Scalable to support multi-client, multi-region, multi-environment setups.

---

## 9. Migration Plan

1. Create new repo structure as shown above.
2. Migrate existing Terraform modules into `main-modules` repo.
3. Migrate backend configurations per environment.
4. Configure GitHub secrets (AWS credentials, roles, Slack webhook if required).
5. Deploy test infra for one client (`client1`) and validate backend state isolation.

---

## 10. Summary

This proposal outlines a clean, extensible, and maintainable OpenTofu automation approach for multi-client environments. The framework supports a fully GitOps-driven model with minimal manual intervention and future scalability for advanced integrations like cost estimation, security scanning, or policy-as-code.

