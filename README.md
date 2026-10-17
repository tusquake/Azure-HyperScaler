# Azure Hyperscaler Learning Notes
<img width="1280" height="369" alt="Microsoft_Azure_Logo svg" src="https://github.com/user-attachments/assets/217828e5-e4cb-4d9b-9490-bdee831f670b" />


Interview-focused notes for learning **Microsoft Azure**, with **AWS** and **GCP** equivalents for every service, real-world project examples, analogies, and the shell commands used in real projects.

> Each topic is checked against Microsoft's current documentation. Service names change often (for example Azure AD is now Microsoft Entra ID), so every note uses the current name and mentions the old one.

---

## What each note contains

Every topic follows the same structure so you can revise quickly:

| Section | Purpose |
|---|---|
| Concepts | What it is and why it exists, in plain words |
| Analogy | One easy story to remember it by |
| Real-world project | How it is used in a real system (running example: **Tomato**, a food-delivery app) |
| AWS and GCP equivalents | Service-to-service mapping |
| CLI commands | `az` commands used in real projects, with `aws` / `gcloud` equivalents |
| Interview questions | Basic, intermediate and scenario questions with model answers |
| Revision sheet | One-page summary for the night before an interview |

Only topics that are asked in interviews or used in production are covered.

---

## Topics

| # | Topic | Status | Notes |
|---|---|---|---|
| 01 | Overview of Azure | Done | [notes/01-azure-overview.md](notes/01-azure-overview.md) |

More topics are added as I learn them, one folder entry per source PDF.

### Topic 01 at a glance

- Cloud computing basics: CapEx vs OpEx, IaaS / PaaS / SaaS / serverless, shared responsibility
- Azure history, naming changes and market position
- Regions, Availability Zones, region pairs, and the resource hierarchy (tenant, management group, subscription, resource group, resource)
- Azure Resource Manager (ARM) and infrastructure as code
- Service map by category with AWS and GCP equivalents
- Pricing model and cost levers
- Tomato food-delivery architecture on Azure
- Free account setup and cost safety habits
- Azure CLI hands-on: VM, Blob Storage, Azure SQL, App Service, RBAC, Key Vault
- 25 interview questions

---

## Repository structure

```
.
├── README.md
├── notes/            # One markdown note per topic
│   └── 01-azure-overview.md
├── cli/              # Runnable command scripts per topic
├── interview/        # Consolidated question bank across topics
└── resources/        # Source PDFs and slides
```

Folders are created as content is added.

---

## Getting started

### Prerequisites

- A Microsoft or GitHub account, a phone number, and a non-prepaid credit or debit card for the [Azure free account](https://azure.microsoft.com/free)
- Azure CLI (current release line 2.88.x)
- Optional for multi-cloud comparison: AWS CLI and Google Cloud CLI (`gcloud`)

### Install Azure CLI and sign in

```bash
# Linux (Debian/Ubuntu)
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# macOS
brew update && brew install azure-cli

# Windows
winget install -e --id Microsoft.AzureCLI

az version
az login
az account show -o table
```

### Create a lab environment

```bash
az group create -n rg-tomato-dev -l centralindia --tags env=dev app=tomato
# ... follow the topic note ...
az group delete -n rg-tomato-dev --yes --no-wait   # clean up when done
```

---

## Cost safety

Cloud labs can cost real money if left running.

- Create a **budget with alerts** in Cost Management on day one.
- Use **one resource group per lab** and delete it when finished.
- Use small SKUs and **deallocate** VMs after use (`az vm deallocate`).
- Turn on **MFA** for your account.
- Never commit secrets. Use Key Vault and keep `.env` files out of Git.

The free account gives $200 credit for 30 days, 12 months of free monthly amounts for popular services, and 55+ always-free services. Check the [free account FAQ](https://azure.microsoft.com/free/free-account-faq) for current terms.

---

## Certification path

AZ-900 (Fundamentals), then AZ-104 (Administrator), then AZ-204 (Developer) or AZ-400 (DevOps), then AZ-305 (Architect) or AZ-500 (Security).
---

## References

- [Azure documentation](https://learn.microsoft.com/azure/)
- [Azure CLI reference](https://learn.microsoft.com/cli/azure/)
- [Azure Architecture Center](https://learn.microsoft.com/azure/architecture/)
- [Azure Well-Architected Framework](https://learn.microsoft.com/azure/well-architected/)
- [AWS documentation](https://docs.aws.amazon.com/)
- [Google Cloud documentation](https://cloud.google.com/docs)
