# Azure Handbook – Part 1: Overview of Azure

## 0. What was verified or corrected from the slides

| Slide said | Current reality (use this in interviews) |
| --- | --- |
| Launched Feb 1, 2010 | Correct for **general availability**. It was announced in 2008 as **Windows Azure** and renamed **Microsoft Azure in 2014**. Interviewers like this nuance. |
| 200+ products and services | Still Microsoft’s own wording (‘more than 200 products and cloud services’). |
| Free account needs a card | Yes. You also get **$200 credit for 30 days**, then must upgrade to pay-as-you-go to keep using services; popular services stay free for 12 months and 55+ services are always free (within monthly limits). |
| Market share AWS 28%, Azure 21%, GCP 14% | Fine as the ‘Big Three’ picture. Exact percentages change every quarter by analyst (Statista, Synergy, Gartner), so quote them as ‘roughly a quarter, a fifth and a seventh’. |
| Azure AD, Cognitive Services, Azure OpenAI | Names have moved: **Azure AD → Microsoft Entra ID**, **Cognitive Services → Azure AI Services**, and the AI platform is now branded **Microsoft Foundry** (formerly Azure AI Foundry). Know both old and new names; docs and job posts use both. |
| Azure CLI | Current release line is **2.88.x (July 2026)**, with a monthly release cadence. |

## 1. Cloud computing from zero

**Definition.** Cloud computing means renting computing resources (servers, storage, databases, networking, AI) over the internet from a provider’s data centres, and paying for what you consume instead of buying hardware.

**Microsoft Azure** is Microsoft’s public cloud platform. You use it to **build, deploy and manage** applications through a web portal (Azure Portal), a command line (Azure CLI / PowerShell), SDKs, or code (Bicep, Terraform).

### Why companies move to the cloud

| Benefit | Meaning | Real example |
| --- | --- | --- |
| CapEx → OpEx | No upfront hardware purchase; monthly bill | A startup avoids buying ₹50 lakh of servers |
| Elasticity | Scale out in peak, scale in when quiet | Food-delivery app at 8 PM dinner rush |
| Scalability | Grow bigger over months or years | User base 10k → 5M |
| High availability | Service survives failures | Two copies of the app in separate buildings |
| Global reach | Deploy near users in many regions | Low latency for customers in several countries |
| Agility | Environment in minutes, not weeks | Dev team spins up a test environment per pull request |
| Disaster recovery | Backups and failover in another region | Bank keeps a standby site |

**CapEx vs OpEx** is a classic fresher question: *Capital expenditure* is upfront spending on assets you own; *operational expenditure* is ongoing pay-per-use spending. Cloud shifts you to OpEx.

### Service models: IaaS, PaaS, SaaS

| Model | You manage | Provider manages | Azure | AWS | GCP |
| --- | --- | --- | --- | --- | --- |
| **IaaS** | OS, runtime, app, data | Hardware, virtualization, network | Virtual Machines | EC2 | Compute Engine |
| **PaaS** | App and data | Everything below, including OS and runtime | App Service, Azure SQL Database | Elastic Beanstalk, RDS | App Engine, Cloud SQL |
| **SaaS** | Just use it (and configure) | Everything | Microsoft 365, Dynamics 365 | Amazon WorkMail/Chime | Google Workspace |
| **Serverless / FaaS** | Only your function code | Servers and scaling, billed per execution | Azure Functions | Lambda | Cloud Run functions |

**Pizza analogy.** *On-premises* = make pizza at home (you buy everything). *IaaS* = you rent the kitchen, bring ingredients and cook. *PaaS* = takeaway, you just add drinks. *SaaS* = dine in at the restaurant, you only eat.

### Deployment models

- **Public cloud**: shared provider infrastructure (Azure, AWS, GCP).
- **Private cloud**: infrastructure used only by one organisation (on-prem or hosted).
- **Hybrid cloud**: on-prem plus public cloud connected together. Azure strength: **Azure Arc**, **Azure Stack HCI**, and ExpressRoute/VPN.
- **Multi-cloud**: using more than one provider (for example Azure + AWS). Azure Arc can manage non-Azure servers and Kubernetes too.

### Shared responsibility model (very frequently asked)

The provider secures **the cloud** (data centres, physical hosts, network backbone). You secure **what you put in the cloud** (identities, data, access rules, patching of VMs you run). The more managed the service (SaaS > PaaS > IaaS), the more the provider handles. Identity and data are **always** your responsibility.

## 2. History, naming and market position

| Year | Event |
| --- | --- |
| 2008 | Announced at PDC as **Windows Azure** (project codename ‘Red Dog’) |
| Feb 1, 2010 | General availability, starting with compute, storage and networking |
| 2012-13 | Virtual Machines (IaaS) added, so Linux and non-.NET workloads became first class |
| 2014 | Renamed **Microsoft Azure** |
| 2016 onward | Resource Manager (ARM) model becomes standard; Kubernetes, containers, DevOps |
| 2020s | Heavy AI push: Azure OpenAI, ML, Foundry |

**Position:** Azure is the #2 public cloud behind AWS, with GCP third. Azure’s edge is deep integration with the Microsoft ecosystem (Windows Server, SQL Server, Active Directory/Entra ID, Microsoft 365, GitHub, Visual Studio), which is why enterprises, banks and governments heavily adopt it.

**Customers from your slide** (to quote in interviews): Accenture, Audi, Carvana, Coca-Cola, CoStar, CSX, Husqvarna, Nestlé, UBS and others. The point to make: Azure is used across retail, auto, finance, logistics and non-profits.

## 3. Global infrastructure and the resource hierarchy

This is not on your slides, but it is the first thing an interviewer asks after ‘What is Azure?’ and you need it for every CLI command.

### Geography

- **Geography**: a market boundary such as India or the US, which keeps data residency and compliance in mind.
- **Region**: a set of data centres in one metro area (for example *Central India*, *South India*, *East US*). You pick a region for every resource.
- **Availability Zones (AZs)**: physically separate data-centre buildings inside a region (usually 3), each with independent power, cooling and network. Spreading a VM across zones protects against a building failure. Zone-redundant services give about 99.99% VM SLA.
- **Region pairs**: each region is paired with another in the same geography for disaster recovery and sequenced platform updates.
- **Edge locations / POPs**: used by CDN and Front Door to cache content close to users.

| Concept | Azure | AWS | GCP |
| --- | --- | --- | --- |
| Region | Region | Region | Region |
| Isolated building inside region | Availability Zone | Availability Zone | Zone |
| Paired DR region | Region pair | No formal pairing (you choose) | No formal pairing (you choose) |

### Resource hierarchy (memorise this)

```
Microsoft Entra tenant (identity boundary)
 └── Management Group  (optional, groups subscriptions, apply policy)
      └── Subscription  (billing + access boundary)
           └── Resource Group  (logical folder for related resources)
                └── Resource  (VM, storage account, SQL DB…)
```

| Level | Purpose | AWS equivalent | GCP equivalent |
| --- | --- | --- | --- |
| Tenant | Identity directory | AWS IAM Identity Center / Organization root | Cloud Identity / Organization |
| Management group | Governance across subscriptions | Organizations OU | Folder |
| Subscription | Billing and limits | **Account** | **Project** (closest) |
| Resource group | Lifecycle container; delete it to delete everything | CloudFormation stack / tags (no exact match) | Project (loosely) |
| Resource | The actual service instance | Resource | Resource |

**Key rules:** every resource lives in exactly one resource group; a resource group has a location only for its metadata; deleting a resource group deletes everything inside it; you can apply RBAC, Azure Policy, tags and locks at any level and they inherit downward.

### Azure Resource Manager (ARM)

ARM is the single management layer behind the portal, CLI, PowerShell, SDKs and templates. Every request goes through ARM, which authenticates it (Entra ID), authorises it (RBAC) and passes it to the resource provider (such as `Microsoft.Compute`). Declarative infrastructure as code uses **ARM templates / Bicep** (Azure-native) or **Terraform** (multi-cloud). AWS equivalent: CloudFormation / CDK. GCP: Infrastructure Manager / Terraform (Deployment Manager is being retired).

## 4. The Azure service map (what ‘200+ services’ really means)

Learn the **categories** and the **one or two flagship services** in each. That is what interviews test.

### Master equivalence table

| Category | Azure service | What it does | AWS | GCP |
| --- | --- | --- | --- | --- |
| **Compute** | Virtual Machines | IaaS servers | EC2 | Compute Engine |
|  | VM Scale Sets (VMSS) | Auto-scaling group of identical VMs | Auto Scaling Groups | Managed Instance Groups |
|  | App Service | PaaS for web apps and APIs | Elastic Beanstalk / App Runner | App Engine / Cloud Run |
|  | Azure Functions | Serverless code, event driven | Lambda | Cloud Run functions |
|  | AKS (Kubernetes) | Managed Kubernetes | EKS | GKE |
|  | Container Apps / Container Instances | Serverless containers | ECS Fargate / App Runner | Cloud Run |
|  | Azure Batch | Large parallel jobs | AWS Batch | Cloud Batch |
| **Storage** | Blob Storage | Object storage (images, backups, data lake) | S3 | Cloud Storage |
|  | Azure Files | Managed SMB/NFS file share | EFS / FSx | Filestore |
|  | Managed Disks | Block storage for VMs | EBS | Persistent Disk |
|  | Queue Storage | Simple message queue | SQS | Cloud Tasks / Pub/Sub |
|  | Table Storage | Cheap NoSQL key-value | DynamoDB | Bigtable / Firestore |
| **Databases** | Azure SQL Database | Managed SQL Server | RDS for SQL Server | Cloud SQL for SQL Server |
|  | Azure Database for MySQL / PostgreSQL (Flexible Server) | Managed open-source RDBMS | RDS / Aurora | Cloud SQL / AlloyDB |
|  | Cosmos DB | Global multi-model NoSQL | DynamoDB | Firestore / Spanner / Bigtable |
|  | Azure Cache for Redis | In-memory cache | ElastiCache | Memorystore |
| **Networking** | Virtual Network (VNet) | Private network, subnets | VPC | VPC |
|  | Load Balancer / Application Gateway | L4 / L7 traffic distribution | NLB / ALB | Cloud Load Balancing |
|  | Azure DNS | DNS hosting | Route 53 | Cloud DNS |
|  | Front Door / CDN | Global entry, caching, WAF | CloudFront | Cloud CDN / Global ALB |
|  | VPN Gateway / ExpressRoute | Connect on-prem | Site-to-Site VPN / Direct Connect | Cloud VPN / Interconnect |
| **Identity and security** | Microsoft Entra ID | Users, SSO, MFA | IAM + IAM Identity Center | Cloud IAM / Cloud Identity |
|  | Azure RBAC | Who can do what on resources | IAM policies | IAM roles |
|  | Key Vault | Secrets, keys, certificates | Secrets Manager + KMS | Secret Manager + Cloud KMS |
|  | Defender for Cloud | Security posture, threat detection | Security Hub / GuardDuty | Security Command Center |
| **Messaging / integration** | Service Bus | Enterprise queues and topics | SQS / SNS / Amazon MQ | Pub/Sub |
|  | Event Grid / Event Hubs | Event routing / big streaming ingest | EventBridge / Kinesis | Eventarc / Pub/Sub |
|  | Notification Hubs | Mobile push notifications | SNS mobile push | Firebase Cloud Messaging |
| **AI and ML** | Azure OpenAI / Microsoft Foundry | Hosted GPT-class models, agents | Bedrock | Vertex AI |
|  | Azure Machine Learning | Train and deploy ML models | SageMaker | Vertex AI |
|  | Azure AI Services | Vision, speech, language APIs | Rekognition, Transcribe, Comprehend | Vision AI, Speech-to-Text |
| **Analytics** | Synapse / Microsoft Fabric | Data warehouse and analytics | Redshift / Athena | BigQuery |
|  | Data Factory | ETL / data pipelines | Glue | Dataflow / Data Fusion |
|  | Databricks | Spark platform | EMR / Databricks | Dataproc / Databricks |
| **IoT** | IoT Hub | Device connectivity | AWS IoT Core | No direct service (IoT Core retired 2023) |
| **DevOps** | Azure DevOps / GitHub Actions | CI/CD | CodePipeline / CodeBuild | Cloud Build |
|  | ACR (Container Registry) | Docker image registry | ECR | Artifact Registry |
| **Management** | Azure Monitor / Log Analytics / App Insights | Metrics, logs, tracing | CloudWatch / X-Ray | Cloud Monitoring / Logging / Trace |
|  | Azure Policy | Enforce governance rules | AWS Config / SCP | Organization Policy |
|  | Cost Management + Billing | Budgets and cost analysis | Cost Explorer / Budgets | Cloud Billing |

**Diagram check:** your slide’s hub-and-spoke picture lists Compute, Storage, Networking, Databases, AI and ML, IoT. Add **Identity/Security, Monitoring, DevOps and Analytics** when you answer in an interview, because real systems always use them.

## 5. Pricing model: pay-as-you-go and beyond

Your slide says ‘pay only for what you use’. In interviews you also need the cost levers:

| Option | Idea | Typical saving | When to use |
| --- | --- | --- | --- |
| **Pay-as-you-go** | Per second / hour / GB / request, no commitment | Baseline | Dev, test, unpredictable load |
| **Reservations (Reserved Instances)** | Commit to 1 or 3 years of a specific VM/DB size | Up to \~70% | Steady 24x7 production |
| **Savings Plan for compute** | Commit to a hourly spend amount, flexible across VM families and regions | Up to \~65% | Steady spend, varied sizes |
| **Spot VMs** | Spare capacity, can be evicted with short notice | Up to \~90% | Batch, CI agents, fault-tolerant jobs |
| **Azure Hybrid Benefit** | Bring your own Windows/SQL Server licences | Large on Windows/SQL | Migrating existing licences |
| **Dev/Test pricing** | Discounted rates for non-prod subscriptions | Varies | Non-prod |

AWS equivalents: On-Demand, Reserved Instances / Savings Plans, Spot Instances, ‘Bring your own licence’. GCP: On-demand, Committed Use Discounts, Sustained Use Discounts (automatic), Spot VMs.

**What drives a bill:** compute hours, storage GB and tier, data **egress** (data leaving Azure; ingress is free), operations count, and managed-service SKU. A frequent real-world surprise: a VM that is merely *stopped* from inside the OS still bills; it must be **deallocated** (stopped from Azure) to stop compute charges, although disks still bill.

## 6. Real-world project: Tomato (food delivery) on Azure

Your PDF lists six needs. Here is how each maps to Azure, plus the extras a production system needs.

| Tomato need | Azure service | AWS | GCP |
| --- | --- | --- | --- |
| Website (React/Angular frontend) | **Static Web Apps** or **App Service**, fronted by **Front Door + CDN** | S3 + CloudFront / Amplify | Firebase Hosting / Cloud Storage + CDN |
| Backend APIs (orders, restaurants, payments) | **App Service** (simple) or **AKS / Container Apps** (microservices) | Beanstalk / ECS / EKS | Cloud Run / GKE |
| Customer and restaurant data | **Azure SQL Database** (orders, payments need ACID transactions) + **Cosmos DB** (menus, carts, catalogue; flexible and fast) | RDS + DynamoDB | Cloud SQL + Firestore |
| Food images | **Blob Storage** (+ CDN for delivery) | S3 + CloudFront | Cloud Storage + Cloud CDN |
| Notifications on order placed / delivered | **Service Bus or Event Grid** → **Azure Functions** → **Notification Hubs** (push) or **Communication Services** (SMS/email) | SQS/EventBridge → Lambda → SNS | Pub/Sub → Cloud Run functions → FCM |
| Thousands of users at peak | **Autoscale** on App Service / VMSS / AKS (HPA + cluster autoscaler), **Redis cache**, **Front Door** | Auto Scaling, ElastiCache | Autoscaler, Memorystore |
| AI chatbot | **Azure OpenAI / Microsoft Foundry** (+ **AI Search** for RAG over menus and FAQs) | Bedrock + Kendra / OpenSearch | Vertex AI + Vertex AI Search |
| Customer login | **Microsoft Entra External ID** (successor to Azure AD B2C, which is closed to new customers) | Cognito | Identity Platform |
| Secrets (DB passwords, API keys) | **Key Vault** + **Managed Identity** | Secrets Manager + IAM roles | Secret Manager + service accounts |
| Monitoring and alerts | **Azure Monitor, Application Insights** | CloudWatch | Cloud Monitoring |
| CI/CD | **GitHub Actions or Azure DevOps** → ACR → AKS/App Service | CodePipeline / ECR | Cloud Build / Artifact Registry |

### Flow of one order

1. Customer opens the site: DNS → **Front Door** (WAF, TLS, caching) → frontend; images load from **Blob via CDN**.
2. Customer logs in via **Entra External ID**; the API receives a JWT token.
3. App calls the **Orders API** (AKS/App Service), which writes the order to **Azure SQL** and publishes an `OrderPlaced` message to **Service Bus**.
4. A **Function** consumes the message and sends a push to the restaurant and customer via **Notification Hubs**.
5. On delivery, the rider app posts `OrderDelivered`; another Function notifies the customer.
6. Every service reads secrets from **Key Vault** using a managed identity, so no password sits in code.
7. At 8 PM, autoscale adds instances; at 3 AM it scales back in so you stop paying for idle capacity.
8. Customer asks ‘Where is my order?’ → chatbot calls the Orders API as a tool and answers.

**Why SQL *and* Cosmos?** (Common follow-up.) Payments and orders need relational integrity and transactions; menus and carts are read-heavy, schema-flexible documents that benefit from global low-latency reads. Use the right database per workload (*polyglot persistence*).

**Why a queue between order and notification?** Decoupling: if the notification system is slow or down, orders are still accepted and messages wait in the queue, so no order is lost.

## 7. The Azure Mall analogy

Your slide compares Azure to a shopping mall. Extend it so you can explain every concept in one story:

| Mall world | Azure world |
| --- | --- |
| The mall | Azure platform |
| Each shop (VM shop, Blob shop, SQL shop) | An individual service |
| Mall branch in a different city | **Region** |
| Separate wings with their own power and AC | **Availability Zones** |
| Your membership / billing account | **Subscription** |
| The floor or section you rent for your business | **Resource group** |
| Security guard checking ID and badge | **Entra ID + RBAC** |
| Safe in the manager’s office | **Key Vault** |
| Shop opening hours and extra staff at rush hour | **Autoscale** |
| Monthly rent based on shop size and usage | **Pay-as-you-go pricing** |
| Mall CCTV and incident log | **Azure Monitor** |

You don’t build the mall; you rent shops (services) only as long as you need them, and the mall operator handles the building, electricity and security. Same idea as the cloud shared-responsibility model.

## 8. Free account setup, step by step

**Prerequisites** (matches your slide)

1. A **Microsoft account or GitHub account** as your sign-in identity.
2. A **phone number** for verification.
3. A **credit card or debit card** (non-prepaid) for identity verification. You are not charged just for verifying.

**Steps**

1. Go to `azure.microsoft.com/free` and click **Try Azure for free**.
2. Sign in with your Microsoft or GitHub account.
3. Fill in country/region, name and phone number (the country is hard to change later and decides billing currency).
4. Verify by SMS or call, then add the card details.
5. Accept the agreement and finish. A default subscription called **Azure subscription 1** is created in your Microsoft Entra tenant.
6. Open `portal.azure.com` and look at **Subscriptions** and **Cost Management + Billing**.

**What you actually get (current terms)**

- **$200 credit valid for 30 days** for anything outside the free amounts.
- **12 months of free monthly amounts** of popular services (certain small VM sizes, Blob storage, Azure SQL Database, and others).
- **55+ services that are always free** within monthly limits.
- Free-account customers are not charged automatically. You have to **upgrade to pay-as-you-go within 30 days or when credit is used up**; otherwise services are disabled. Azure Hybrid Benefit cannot be combined with free credit, and only one free account per new customer.

**Safety habits (do these on day one)**

- Create a **budget with alerts** in Cost Management (for example at 50%, 80%, 100%).
- Put each lab in its **own resource group** and delete the group when you finish.
- Turn on **MFA** for your account.
- Use small SKUs (`Standard_B1s`, `B2s`) and **deallocate** VMs after use.
- Never leave a public IP, database or AKS cluster running overnight in a lab.

## 9. Azure CLI hands-on (what real projects use)

Azure has two shells: **Azure CLI** (`az`, cross-platform, the most common in pipelines) and **Azure PowerShell** (`Az` module). You can also use **Cloud Shell** in the portal with no installation. The current CLI line is 2.88.x.

### Install and sign in

```bash
# Linux (Debian/Ubuntu)
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
# macOS
brew update && brew install azure-cli
# Windows
winget install -e --id Microsoft.AzureCLI

az version                      # check installed version
az upgrade                      # update CLI and extensions
az login                        # opens browser sign-in
az login --use-device-code      # for servers / no browser
az account show -o table        # which subscription am I using?
az account list -o table        # all subscriptions
az account set --subscription 'Azure subscription 1'
az configure --defaults group=rg-tomato-dev location=centralindia
```

Output formats: `-o table`, `-o json`, `-o tsv`, `-o yamlc`. Filter with JMESPath: `--query '[].{name:name,state:powerState}'`.

### Discover things

```bash
az account list-locations -o table                                # regions
az vm list-skus -l centralindia --size Standard_B --output table  # VM sizes available
az vm image list -o table                                         # popular images
az find 'az vm create'                                            # AI-assisted examples
az vm create -h                                                   # help for any command
```

### Resource group, then the three ‘shops’ from your slides

```bash
# 1. Resource group (the folder for the project)
az group create -n rg-tomato-dev -l centralindia --tags env=dev app=tomato owner=you

# 2. Virtual Machine shop
az vm create -g rg-tomato-dev -n vm-tomato-web \
  --image Ubuntu2204 --size Standard_B1s \
  --admin-username azureuser --generate-ssh-keys \
  --public-ip-sku Standard
az vm open-port -g rg-tomato-dev -n vm-tomato-web --port 80
az vm list -g rg-tomato-dev -d -o table          # -d shows IP and power state
az vm show -g rg-tomato-dev -n vm-tomato-web --query publicIps -o tsv
ssh azureuser@<public-ip>
az vm deallocate -g rg-tomato-dev -n vm-tomato-web   # stops compute billing
az vm start -g rg-tomato-dev -n vm-tomato-web

# 3. Blob Storage shop (names: 3-24 chars, lowercase letters/numbers, globally unique)
az storage account create -g rg-tomato-dev -n sttomatodev$RANDOM \
  -l centralindia --sku Standard_LRS --kind StorageV2 \
  --allow-blob-public-access false --min-tls-version TLS1_2
SA=<storage-account-name>
az storage container create --account-name $SA -n food-images --auth-mode login
az storage blob upload --account-name $SA -c food-images \
  -f ./biryani.jpg -n biryani.jpg --auth-mode login
az storage blob list --account-name $SA -c food-images --auth-mode login -o table
az storage blob generate-sas --account-name $SA -c food-images -n biryani.jpg \
  --permissions r --expiry 2030-01-01T00:00Z --https-only --auth-mode login --as-user -o tsv

# 4. Azure SQL shop
az sql server create -g rg-tomato-dev -n sql-tomato-dev001 -l centralindia \
  --admin-user sqladmin --admin-password '<StrongPassword!>'
az sql db create -g rg-tomato-dev -s sql-tomato-dev001 -n tomatodb \
  --service-objective Basic
az sql server firewall-rule create -g rg-tomato-dev -s sql-tomato-dev001 \
  -n allow-my-ip --start-ip-address <my-ip> --end-ip-address <my-ip>

# 5. Web app (PaaS) in one command
az webapp up -n tomato-web-$RANDOM -g rg-tomato-dev -l centralindia --sku F1 --runtime 'NODE:20-lts'
```

### Access control, secrets, monitoring (used in every real project)

```bash
# RBAC: give a user read-only access to the resource group
az role assignment create --assignee user@company.com \
  --role Reader --scope $(az group show -n rg-tomato-dev --query id -o tsv)
az role assignment list --resource-group rg-tomato-dev -o table

# Key Vault
az keyvault create -g rg-tomato-dev -n kv-tomato-dev001 -l centralindia --enable-rbac-authorization true
az keyvault secret set --vault-name kv-tomato-dev001 -n DbPassword --value '<secret>'
az keyvault secret show --vault-name kv-tomato-dev001 -n DbPassword --query value -o tsv

# Managed identity for the VM (no passwords in code)
az vm identity assign -g rg-tomato-dev -n vm-tomato-web

# Autoscale rule (VM Scale Set / App Service plan)
az monitor autoscale create -g rg-tomato-dev --resource <vmss-or-plan-id> \
  --min-count 2 --max-count 10 --count 2 -n tomato-autoscale

# Policy / lock / tags
az lock create -n no-delete -g rg-tomato-prod --lock-type CanNotDelete
az tag update --resource-id <id> --operation merge --tags cost-center=food

# Cost control: list everything, then clean up
az resource list -g rg-tomato-dev -o table
az group delete -n rg-tomato-dev --yes --no-wait    # deletes ALL resources inside
```

### Infrastructure as code (what teams use instead of clicking)

```bash
az deployment group create -g rg-tomato-dev -f main.bicep -p env=dev
az deployment group what-if -g rg-tomato-dev -f main.bicep   # preview changes
# Terraform alternative: terraform init && terraform plan && terraform apply
```

### The same task on AWS and GCP

| Task | Azure CLI | AWS CLI | gcloud |
| --- | --- | --- | --- |
| Sign in | `az login` | `aws configure` / `aws sso login` | `gcloud auth login` |
| Who/where am I | `az account show` | `aws sts get-caller-identity` | `gcloud config list` |
| Create a container for resources | `az group create` | (no equivalent; use stack/tags) | `gcloud projects create` |
| Create VM | `az vm create` | `aws ec2 run-instances` | `gcloud compute instances create` |
| List VMs | `az vm list -d` | `aws ec2 describe-instances` | `gcloud compute instances list` |
| Stop VM (stop compute billing) | `az vm deallocate` | `aws ec2 stop-instances` | `gcloud compute instances stop` |
| Create object storage | `az storage account create` + `az storage container create` | `aws s3 mb s3://bucket` | `gcloud storage buckets create gs://bucket` |
| Upload object | `az storage blob upload` | `aws s3 cp file s3://bucket/` | `gcloud storage cp file gs://bucket/` |
| Create SQL DB | `az sql db create` | `aws rds create-db-instance` | `gcloud sql instances create` |
| Deploy web app | `az webapp up` | `eb deploy` / `aws apprunner` | `gcloud run deploy` |
| Delete everything in a project | `az group delete` | `aws cloudformation delete-stack` | `gcloud projects delete` |

## 10. Who uses Azure and career paths

**Who uses it and why**

| Sector | Why Azure | Example |
| --- | --- | --- |
| Startups | Credits, fast start, PaaS | Launch an MVP on App Service + Azure SQL |
| Enterprises | Microsoft licensing, Entra ID, hybrid | Migrate SAP and SQL Server estates |
| Banks | Compliance certifications, private networking, encryption, regional residency | Core banking analytics inside private VNets |
| Government | Sovereign / government clouds | Citizen portals |
| Healthcare | HIPAA-eligible services, data protection | Imaging storage and analytics |
| E-commerce | Autoscale for sales events, CDN, AI personalisation | Flash-sale traffic spikes |

**Roles from your slide and what each does day to day**

| Role | Daily work | Core skills |
| --- | --- | --- |
| Azure Cloud Engineer | Builds and automates infrastructure | VNets, VMs, Bicep/Terraform, `az` |
| Cloud Administrator | Runs and secures the estate | RBAC, Policy, backups, cost, patching |
| DevOps Engineer | CI/CD, containers, IaC | GitHub Actions/Azure DevOps, AKS, Docker |
| Cloud Architect | Designs solutions and trade-offs | Well-Architected Framework, networking, security |
| SRE | Reliability, SLOs, incident response | Monitoring, automation, Kubernetes |

**Certification path:** AZ-900 (fundamentals) → AZ-104 (Administrator) → AZ-204 (Developer) or AZ-400 (DevOps) → AZ-305 (Architect) / AZ-500 (Security). Add DP-900 (data) or AI-900 (AI) if relevant. Certifications get updated, so check the exam page before booking.

## 11. Interview questions and model answers

**Basic**

1. **What is Azure and how is it different from on-premises?** Azure is Microsoft’s public cloud with 200+ services over a global data-centre network. Instead of buying and maintaining hardware (CapEx), you rent resources on demand (OpEx), scale in minutes and pay for usage.
2. **Explain IaaS, PaaS and SaaS with Azure examples.** IaaS: Virtual Machines (you manage OS and up). PaaS: App Service / Azure SQL (you manage code and data). SaaS: Microsoft 365 (you just use it). Add the pizza analogy.
3. **What is the shared responsibility model?** Microsoft secures the physical infrastructure and platform; you secure identities, data, configuration and the OS on IaaS. Identity and data are always yours.
4. **Region vs Availability Zone vs Region pair?** Region = metro-area group of data centres; AZ = isolated building within a region (power, cooling, network); region pair = DR partner region in the same geography with sequenced updates.
5. **What is the Azure resource hierarchy?** Management group → Subscription → Resource group → Resource. RBAC and Policy inherit downward.
6. **What is a resource group? Can you move resources between groups?** A lifecycle and management container. Yes, most resources can be moved to another group or subscription (`az resource move`), but some types have restrictions; the resource keeps its original region.
7. **What is the pay-as-you-go model? How do you reduce cost?** Pay per use. Reduce with right-sizing, autoscale, deallocating idle VMs, reservations/savings plans, Spot, Hybrid Benefit, budgets, and tags for chargeback.
8. **What does the Azure free account include?** $200 credit for 30 days, 12 months of free monthly amounts of popular services, and 55+ always-free services; you must upgrade to pay-as-you-go to continue after 30 days.

**Intermediate**

9. **What is ARM?** The deployment and management layer behind portal, CLI and SDKs; authenticates via Entra ID, authorises via RBAC, and routes to resource providers. Enables declarative IaC (Bicep/ARM templates).
10. **ARM template/Bicep vs Terraform?** Bicep is Azure-native with day-zero support for new features and no state file; Terraform is multi-cloud with state management and a larger ecosystem. Many teams pick Terraform for multi-cloud and Bicep for Azure-only.
11. **Difference between stopping and deallocating a VM?** Stopping inside the OS keeps the host reserved and you keep paying for compute. Deallocating (`az vm deallocate`) releases the compute and stops compute charges (disks still bill; dynamic public IP may change).
12. **Blob vs Files vs Disk?** Blob = object storage for unstructured data via HTTP; Files = managed SMB/NFS share; Managed Disk = block storage attached to a VM.
13. **Azure SQL Database vs SQL on a VM vs Managed Instance?** Database = PaaS single DB, least management; VM = full control (IaaS), you patch and back up; Managed Instance = near 100% SQL Server compatibility as PaaS, best for lift-and-shift.
14. **When would you choose Cosmos DB over Azure SQL?** For globally distributed, low-latency, flexible-schema workloads with elastic throughput; choose SQL when you need relational integrity, joins and transactions.
15. **Service Bus vs Event Grid vs Event Hubs?** Service Bus: reliable enterprise messaging with queues/topics (commands, orders). Event Grid: reactive event routing (something happened). Event Hubs: high-volume streaming ingest (telemetry, logs).
16. **App Service vs AKS vs Functions?** App Service for straightforward web apps and APIs with low ops overhead; AKS for many containerised microservices needing orchestration control; Functions for short event-driven tasks and scale-to-zero.
17. **How do you give an app access to Key Vault without storing passwords?** Enable a **managed identity** on the app, grant it a Key Vault role (Key Vault Secrets User), and read secrets with the SDK’s default credentials.
18. **What is the difference between Entra ID roles and Azure RBAC roles?** Entra roles manage the directory (users, groups, apps); Azure RBAC roles manage access to Azure resources. They are separate systems.

**Scenario**

19. **Your app crashes at 8 PM every day under load. What do you do?** Check Azure Monitor / App Insights for CPU, memory, DB DTU/CPU and queue depth; enable autoscale with schedule and metric rules; add Redis caching; use Front Door/CDN to offload static content; load-test; add alerts.
20. **How do you make an app highly available?** Run at least two instances across Availability Zones, put them behind a load balancer or Front Door, use zone-redundant databases, and for DR replicate to a paired region with tested failover (set RTO and RPO).
21. **Your bill doubled this month. How do you investigate?** Cost Management → Cost analysis grouped by service, resource group and tag; look for forgotten VMs, premium disks, egress, or a scaled-up database; set budgets/alerts and Azure Policy to restrict expensive SKUs.
22. **How would you secure Tomato’s architecture?** Entra ID + MFA, least-privilege RBAC, private endpoints for SQL and storage, NSGs, WAF on Front Door, Key Vault with managed identities, encryption at rest and in transit, Defender for Cloud, and logging to Log Analytics.
23. **How do you migrate an on-prem app to Azure?** Assess with Azure Migrate, choose rehost (VMs), refactor (PaaS) or rebuild, set up networking (VPN/ExpressRoute) and identity sync, migrate data with Database Migration Service, test, cut over, and optimise.

**Multi-cloud**

24. **Equivalent of Azure Blob, Functions and AKS on AWS and GCP?** S3 / Lambda / EKS on AWS; Cloud Storage / Cloud Run functions / GKE on GCP.
25. **Azure subscription is closest to what in AWS and GCP?** AWS account; GCP project (the project is both the billing link and resource container, so it also overlaps with a resource group).

## 12. One-page revision sheet

- Azure = Microsoft public cloud, 200+ services, pay-as-you-go, GA Feb 1 2010 (announced 2008 as Windows Azure, renamed 2014).
- Big Three: AWS > Azure > GCP.
- Models: IaaS (VM) / PaaS (App Service, SQL DB) / SaaS (M365) / Serverless (Functions).
- Hierarchy: Tenant → Management group → Subscription → Resource group → Resource.
- Region → Availability Zones; region pairs for DR.
- Shops from your slides: **VM** (EC2, Compute Engine), **Blob** (S3, Cloud Storage), **Azure SQL** (RDS, Cloud SQL).
- Free account: card + phone + Microsoft/GitHub account; $200 for 30 days; upgrade to keep going.
- First commands: `az login`, `az account show`, `az group create`, `az vm create`, `az storage account create`, `az group delete`.
- Always: budget alerts, one resource group per lab, deallocate VMs, delete when done
