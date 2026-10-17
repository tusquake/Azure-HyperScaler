# Azure Handbook – Part 2: Azure Virtual Machines

## 0. What was verified or corrected from the slides

| Slide said | Current reality (use this in interviews) |
| --- | --- |
| Azure VM is IaaS | Correct. You manage the OS, patching, runtime, app and data. Azure manages the physical host, hypervisor and network fabric. |
| NSG is region specific | Correct. An NSG can only be attached to resources (subnets, NICs) in the same region. |
| NSGs are stateful | Correct. Return traffic for an allowed flow is allowed automatically. |
| Optionally assign a static public IP | In practice **Standard SKU public IPs are always static**. Basic SKU public IPs were retired (30 Sept 2025), so the real choice is: attach a Standard public IP, or use no public IP. |
| Diagram shows NAT Gateway and Azure Bastion | This matches current guidance. For **new virtual networks (API versions after 31 March 2026) subnets are private by default**: a VM with no public IP, NAT Gateway, load balancer outbound rule or firewall has **no outbound internet access**. Existing VNets keep working. |
| SSH on 22 and RDP on 3389 straight from the internet | Fine for a lab. Production uses Azure Bastion, Just-in-Time access or VPN, and never opens 22/3389 to `0.0.0.0/0`. |
| Sizes: B, D, E, N | Correct starter set. Also know **F** (compute), **L** (storage), **M** (very large memory), **H** (HPC). Newer generations (v5, v6) and AMD/ARM variants exist, so always check availability with `az vm list-skus`. |
| Spot VMs: very low cost | Up to about 90% cheaper, evicted with a best-effort notice of up to 30 seconds, no SLA. Eviction policy is **Deallocate** (default) or **Delete**. |
| Pricing: PAYG, Reserved, Spot, Dedicated Host | Also know **Savings Plan for compute**, **Azure Hybrid Benefit** and Dev/Test pricing. |
| Temporary SSD in the diagram | Data on the temporary disk is lost on redeploy, deallocate or move to another host. Never store anything important there. |
| SMTP port 25 in the ports list | Azure blocks outbound port 25 for most subscription types (trial and pay-as-you-go). Use authenticated SMTP on 587 through a relay service. |

## 1. Virtual machines from zero

### 1.1 What is an Azure VM?

A **virtual machine** is a software-defined computer that runs on a physical server in a Microsoft data centre. A **hypervisor** (Azure uses a Hyper-V based one) slices one powerful physical host into many isolated VMs. Each VM gets its own virtual CPU, RAM, disks, network card and operating system, and behaves exactly like a normal server.

**Azure Virtual Machines = IaaS.** You choose the OS and size, Azure gives you the machine in about a minute, and you pay while it is allocated.

The slide lists what every machine needs: **OS, RAM, CPU, disk space, network/firewall, and a way to access it**. Those six items are exactly the choices you make when you create a VM, and they map to the resources in section 2.

### 1.2 Why do we need a VM?

Without cloud, hosting a website means buying a physical server, waiting weeks for delivery, paying for power, cooling and space, replacing failed hardware, and over-buying for peak traffic. With a VM you rent exactly the capacity you need, resize it later, and stop paying when you delete it.

**Typical real-world uses**

| Use case | Example |
| --- | --- |
| Lift-and-shift migration | Move an on-prem Windows/Linux application to Azure without rewriting it |
| Software that needs full OS control | Custom agents, kernel settings, licensed enterprise software |
| Self-hosted databases | SQL Server, PostgreSQL or MongoDB where a managed service is not allowed or not suitable |
| Legacy applications | Old .NET Framework or Java apps tied to a specific OS version |
| Dev and test machines | Disposable environments, per-developer workstations |
| CI/CD build agents | Self-hosted GitHub Actions or Azure DevOps agents, often on Spot VMs |
| Jump servers / admin boxes | Controlled entry point into a private network |
| GPU and HPC | Model training, rendering, simulations (N-series, H-series) |

### 1.3 Analogies

- **Renting a flat instead of buying land and building a house.** The building (data centre, power, water, security) belongs to the landlord (Azure). You choose a 1BHK or a 3BHK (VM size), furnish it (install software) and pay rent as long as you stay.
- **Image = furnished-flat template.** Ubuntu or Windows Server is the template the flat is built from.
- **Temporary disk = whiteboard.** It is wiped when you leave. **Managed disk = cupboard** whose contents stay.
- **NSG = security guard at the gate.** Checks who may enter on which door (port).
- **Azure Bastion = reception desk** that takes visitors to your flat through a controlled lift, so you never give out the flat’s address.
- **Spot VM = discounted sublet** the owner can take back with 30 seconds’ notice. **Dedicated Host = renting the whole floor.** **Reserved instance = annual lease** versus nightly hotel rate (pay-as-you-go).

### 1.4 VM versus other compute options

|  | VM | Container (AKS / Container Apps) | App Service | Functions |
| --- | --- | --- | --- | --- |
| Control | Full OS | App + container image | App code only | Function code only |
| Management effort | Highest (patching, scaling) | Medium | Low | Lowest |
| Startup | About a minute | Seconds | Minutes (always on) | Milliseconds to seconds |
| Best for | Legacy, custom OS, databases | Microservices | Web apps and APIs | Event-driven tasks |
| AWS | EC2 | ECS/EKS/Fargate | Elastic Beanstalk / App Runner | Lambda |
| GCP | Compute Engine | GKE / Cloud Run | App Engine / Cloud Run | Cloud Run functions |

**Rule of thumb for interviews:** start with the most managed option that works (PaaS or serverless). Choose a VM only when you need OS-level control, a specific licence, unsupported software, or you are lifting and shifting.

### 1.5 Shared responsibility on a VM

| Azure is responsible for | You are responsible for |
| --- | --- |
| Physical data centre, hosts, hypervisor | Guest OS patching and hardening |
| Network fabric and host isolation | NSG rules, open ports |
| Hardware replacement | Installed software, application security |
| Storage durability of managed disks | Backups, data protection, encryption choices |
| SLA for the platform | Identity, access keys, passwords |

## 2. Anatomy of an Azure VM

Creating one VM actually creates a **bundle of separate Azure resources**. The slide’s Windows VM diagram shows them all.

### 2.1 Resources that make up a VM

| Resource | Purpose | Notes |
| --- | --- | --- |
| **Virtual machine** | The compute: size, image reference, settings | Billed per second while allocated |
| **OS disk** | Managed disk holding the operating system | Persistent. Billed even when VM is stopped |
| **Data disks** (0 to many) | Extra managed disks for application data | Persistent, can be detached and re-attached |
| **Temporary disk** | Local SSD on the host (`/dev/sdb` Linux, `D:` Windows) | Not persistent. Used for swap, caches, scratch |
| **Network interface (NIC)** | Connects the VM to a subnet; holds private IP, NSG link | One VM can have several NICs |
| **Public IP address** | Optional internet-facing address | Standard SKU is static and zone-redundant |
| **Virtual network and subnet** | Private network the VM lives in | Created by default if you don’t supply one |
| **Network security group** | Firewall rules | Attached to NIC and/or subnet |
| **NAT Gateway** | Explicit outbound internet for private subnets | Needed for new private-by-default subnets |
| **Azure Bastion** | Browser-based or native-client RDP/SSH over TLS, no public IP on the VM | Lives in `AzureBastionSubnet` |
| **Diagnostics storage / Log Analytics** | Boot diagnostics, logs, metrics | Used for troubleshooting a VM that won’t boot |

**Critical cost fact:** deleting the VM with `az vm delete` does **not** automatically delete its disks, NIC, public IP or NSG unless you configured delete options. Orphaned disks and Standard public IPs keep billing. Deleting the **resource group** removes everything.

### 2.2 The creation choices from your slides

| Choice | Meaning | CLI flag | Real-world guidance |
| --- | --- | --- | --- |
| **VM size** | vCPUs + memory + disk and network limits | `--size Standard_B2s` | Start small, measure, resize. Use Azure Advisor for right-sizing |
| **Image** | OS template | `--image Ubuntu2204` | Prefer maintained marketplace images or your own golden image |
| **Disks** | OS and data disk type and size | `--os-disk-size-gb`, `--storage-sku` | Premium SSD for production, Standard SSD for dev |
| **Networking** | VNet, subnet, public IP, NSG | `--vnet-name --subnet --public-ip-sku --nsg` | Production: private IP only, access via Bastion/VPN |
| **Authentication** | SSH key (Linux) or password (Windows) | `--generate-ssh-keys`, `--admin-password` | SSH keys over passwords. Consider Entra ID login |
| **Managed identity** | Identity for the VM to call other Azure services without secrets | `--assign-identity` | Use instead of storing keys in code |
| **Custom data** | Script/cloud-init run at first boot | `--custom-data script.sh` | Bootstraps a server automatically |
| **Public IP** | Internet address, static for stable DNS/allow-lists | `--public-ip-address`, `--public-ip-sku Standard` | Only for internet-facing servers |
| **Availability option** | Zone, availability set, scale set | `--zone 1` | Spread across zones for production |

### 2.3 Images

An image is a disk template. You pick one when creating the VM.

- **Marketplace images:** Ubuntu, Red Hat, SUSE, Debian, Windows Server, SQL Server on Windows, and thousands of vendor images. An image is identified by a URN: `Publisher:Offer:Sku:Version` (for example `Canonical:ubuntu-24_04-lts:server:latest`). The CLI also has short aliases such as `Ubuntu2204`, `Ubuntu2404`, `Win2022Datacenter`.
- **Custom images / Azure Compute Gallery:** your own golden image with software and hardening pre-installed. Built manually (`az image create`) or with **Packer** / **Azure Image Builder**. Versioned and replicated across regions.
- **Generation 1 vs Generation 2:** Gen 2 uses UEFI, supports larger disks, Trusted Launch (secure boot, vTPM) and newer sizes. Use Gen 2 for new builds.
- **Trusted Launch:** default security type for supported Gen 2 images. Adds secure boot and virtual TPM to protect against boot-level malware.

```bash
az vm image list -o table                                       # popular aliases
az vm image list --publisher Canonical --all -o table           # everything from one publisher
az vm image list-skus -l centralindia -f 0001-com-ubuntu-server-jammy -p Canonical -o table
az vm image show --urn Canonical:ubuntu-24_04-lts:server:latest
```

Equivalents: AWS **AMI**; GCP **image / image family** (`ubuntu-2204-lts`).

### 2.4 VM sizes and families

Your slide gives the four most common families. The full practical picture:

| Family | Optimised for | Typical workload | AWS rough match | GCP rough match |
| --- | --- | --- | --- | --- |
| **B** (burstable) | Low baseline CPU, occasional bursts, cheapest | Small website, blog, dev/test, bastion, small DB | T3 / T4g | E2 shared-core / E2 |
| **D** (general purpose) | Balanced CPU:memory | Business apps, e-commerce, web/app tier, small DB | M6i / M7i | N2 / N4 / C3 |
| **E** (memory optimised) | High memory per vCPU | Large databases, caching, in-memory analytics | R6i / R7i | N2-highmem / M-series |
| **F** (compute optimised) | High CPU per memory | Batch, gaming servers, web servers under heavy load | C6i / C7i | C2 / C3 |
| **L** (storage optimised) | High local disk throughput | NoSQL, data warehousing | I3 / I4i | Z3 / local SSD sizes |
| **M** (very large memory) | Multi-TB memory | SAP HANA, huge in-memory DBs | X2 / High Memory | M-series |
| **N** (GPU) | NC = compute/AI, ND = deep learning training, NV = visualisation/VDI | AI, ML, rendering, video | P / G / Inf series | A2 / A3 / G2 |
| **H** (HPC) | Fast CPU and interconnect | Simulations, fluid dynamics | Hpc series | H3 |

**Reading a size name:** `Standard_D4s_v5`

| Part | Meaning |
| --- | --- |
| `D` | Family (general purpose) |
| `4` | 4 vCPUs |
| `s` | Supports Premium SSD storage |
| `v5` | Generation/version of the hardware |

Other letters you will meet: `a` = AMD CPU, `p` = ARM (Ampere) CPU, `d` = has local temp disk, `l` = low memory, `i` = isolated. A lower number in a name like `E8-4s_v5` means a constrained-vCPU size (same memory, fewer billed cores, useful for licensing).

**B-series credits (very common interview question).** B-series VMs run at a baseline CPU percentage and earn **credits** when idle. When busy they spend credits to burst above baseline. When credits run out the VM is throttled back to baseline. This is why B-series is cheap but unsuitable for a sustained 80% CPU workload. Monitor the `CPU Credits Remaining` metric.

**Choosing a size (decision flow):** start with B or D. Memory-heavy? E. CPU-heavy? F. GPU? N. Check quota first, since a new subscription has default vCPU limits per region and per family.

```bash
az vm list-sizes -l centralindia -o table                          # sizes in a region
az vm list-skus -l centralindia --size Standard_D --all -o table   # availability, restrictions, zones
az vm list-usage -l centralindia -o table                          # your quota usage
```

### 2.5 Disks

Azure VM disks are **managed disks** (you don’t manage the storage account).

| Disk type | Media | Use | AWS EBS | GCP PD |
| --- | --- | --- | --- | --- |
| **Standard HDD** | HDD | Backups, rarely accessed data | st1 / sc1 | pd-standard |
| **Standard SSD** | SSD | Web servers, dev/test | gp2 / gp3 | pd-balanced |
| **Premium SSD** | SSD | Production apps and databases | gp3 / io1 | pd-ssd |
| **Premium SSD v2** | SSD, size/IOPS/throughput set independently | Heavy production DBs | gp3 / io2 | Hyperdisk Balanced |
| **Ultra Disk** | Highest IOPS, lowest latency | Tier-1 databases, SAP | io2 Block Express | Hyperdisk Extreme |
| **Ephemeral OS disk** | Uses local VM cache/temp storage | Stateless nodes, scale sets (fast reimage) | Instance store root | Local SSD |

Key points interviewers ask about:

- **Three disk roles:** OS disk (one), data disks (many), temporary disk (not persistent).
- **Redundancy:** LRS (three copies in one datacentre) or ZRS (across zones) for managed disks.
- **Encryption:** server-side encryption with platform-managed keys is on by default. Options: customer-managed keys in Key Vault, **encryption at host**, Azure Disk Encryption (BitLocker/dm-crypt).
- **Snapshots:** point-in-time read-only copy of a disk, used for backup and cloning. **Azure Backup** automates this with policies.
- **Shared disks:** one disk attached to several VMs for clustering.
- **Disk size drives performance.** IOPS and throughput are tied to size tier on Premium SSD, so a bigger disk can be faster.
- **Caching:** `ReadOnly` for read-heavy data disks, `None` for write-heavy/log disks, `ReadWrite` for OS disks.

## 3. Networking for VMs and the Network Security Group

### 3.1 The pieces

- **Virtual network (VNet):** your private network in Azure, defined by an address space such as `10.0.0.0/16`. Isolated by default. Equivalent: AWS VPC, GCP VPC.
- **Subnet:** a slice of the VNet (for example `10.0.1.0/24`). VMs get their private IP from a subnet. **Azure reserves 5 addresses in every subnet** (network address, default gateway, two for Azure DNS, and broadcast), so a `/24` has 251 usable IPs.
- **NIC:** the virtual network card. The VM’s private IP lives on the NIC. NSGs can attach here.
- **Private IP:** dynamic (default, stable while the VM exists) or static. Used for traffic inside the VNet and over VPN/ExpressRoute.
- **Public IP:** internet-reachable address. Use the **Standard SKU** (static, zone-redundant, secure by default, closed until an NSG allows traffic). Basic SKU is retired.
- **DNS:** VMs in the same VNet resolve each other by name through Azure-provided DNS. Use Azure DNS or Private DNS zones for your own names.

### 3.2 Outbound internet access (changed recently, frequently asked)

A VM needs an **explicit outbound method** to reach the internet from a new VNet. Options:

| Method | When to use | AWS | GCP |
| --- | --- | --- | --- |
| Public IP on the VM | Single internet-facing VM, labs | Public/Elastic IP | External IP |
| **NAT Gateway** on the subnet | Private VMs that only need outbound (package updates, API calls); fixed outbound IP | NAT Gateway | Cloud NAT |
| Load Balancer outbound rules | VMs behind a Standard load balancer | NLB/ALB + NAT | Cloud NAT |
| Azure Firewall / NVA with a route table | Centralised inspection and filtering | Network Firewall | Cloud NGFW |

Old VNets and VMs using ‘default outbound access’ keep working, but this was never recommended for production, because the outbound IP is not under your control and traffic bypasses inspection.

```bash
az network public-ip create -g rg-vm-lab -n pip-nat --sku Standard --allocation-method Static
az network nat gateway create -g rg-vm-lab -n natgw-lab --public-ip-addresses pip-nat --idle-timeout 10
az network vnet subnet update -g rg-vm-lab --vnet-name vnet-lab -n snet-app --nat-gateway natgw-lab
```

### 3.3 Network Security Group (NSG)

**Definition (from your slide):** an NSG is a set of firewall rules that allow or deny inbound and outbound network traffic to Azure resources in a VNet.

**Important properties**

| Property | Detail |
| --- | --- |
| Region specific | Can only be associated with resources in the same region |
| Allow and Deny | Both rule types are supported |
| Attachment points | A **subnet** and/or a **NIC**. Best practice: attach at the subnet so every VM is covered |
| Stateful | If inbound is allowed, the reply is allowed automatically, no matching outbound rule needed |
| Priority based | Priority 100 to 4096, **lowest number is evaluated first**, first match wins |
| Reusable | One NSG can be attached to many subnets/NICs |

**A rule has:** name, priority, source (IP, CIDR, service tag, or application security group), source port range, destination, destination port range, protocol (TCP/UDP/ICMP/Any), direction, action.

**Evaluation when both subnet and NIC have NSGs:** inbound traffic passes the **subnet NSG first, then the NIC NSG**; outbound passes the **NIC NSG first, then the subnet NSG**. Traffic must be allowed by both.

**Default rules (always present, cannot be deleted, priority 65000+)**

| Direction | Priority | Name | Effect |
| --- | --- | --- | --- |
| Inbound | 65000 | AllowVnetInBound | Allow traffic from within the VNet (and peered/connected networks) |
| Inbound | 65001 | AllowAzureLoadBalancerInBound | Allow Azure load balancer health probes |
| Inbound | 65500 | DenyAllInBound | Deny everything else from the internet |
| Outbound | 65000 | AllowVnetOutBound | Allow traffic inside the VNet |
| Outbound | 65001 | AllowInternetOutBound | Allow outbound to the internet |
| Outbound | 65500 | DenyAllOutBound | Deny everything else |

This is why a fresh VM is unreachable from the internet until **you** add an inbound allow rule, as your slide says.

**Useful building blocks**

- **Service tags:** named groups of Microsoft-managed IP ranges, such as `Internet`, `VirtualNetwork`, `AzureLoadBalancer`, `Storage`, `Sql`, `AzureCloud`. Cleaner and safer than hard-coding IPs.
- **Application Security Groups (ASG):** label VMs by role (`asg-web`, `asg-db`) and write rules between roles instead of IPs.
- **Flow logs:** record allowed and denied traffic for auditing. NSG flow logs are being replaced by **virtual network flow logs**, so use the newer feature in new projects.

### 3.4 NSG, Azure Firewall and other firewalls

|  | NSG | Azure Firewall | WAF (Application Gateway / Front Door) |
| --- | --- | --- | --- |
| Layer | 3 and 4 (IP, port, protocol) | 3 to 7, FQDN and threat intelligence filtering | 7 (HTTP: SQL injection, XSS, bots) |
| Scope | Subnet/NIC | Central hub for the VNet(s) | Web apps |
| Cost | Free | Paid, per hour and per GB | Paid |
| Use | Basic segmentation | Central egress control, east-west inspection | Protect web traffic |

| Azure | AWS | GCP |
| --- | --- | --- |
| NSG (stateful, attached to subnet/NIC) | **Security Group** (stateful, attached to instance/ENI) and **Network ACL** (stateless, subnet level) | **VPC firewall rules** (stateful, targeted by network tag or service account) |

**Interview point:** AWS has two layers (security group stateful plus NACL stateless). Azure NSG is stateful only, and covers both subnet and NIC levels.

### 3.5 Ports you must know

| Port | Service | Notes |
| --- | --- | --- |
| 80 | HTTP | Unencrypted web |
| 443 | HTTPS | TLS web traffic. Always preferred |
| 22 | SSH and SFTP | Linux remote admin; SFTP runs over SSH |
| 21 | FTP | Unsecured, avoid; also needs data ports (20 or passive range) |
| 25 | SMTP | Server-to-server mail. Outbound blocked by default on most Azure subscriptions; use 587 |
| 53 | DNS | UDP and TCP |
| 3389 | RDP | Windows remote desktop. Never expose to the internet |
| 3306 | MySQL | Keep private |
| 5432 | PostgreSQL | Keep private |
| 1433 | SQL Server | Keep private |
| 6379 | Redis | Keep private |
| 27017 | MongoDB | Keep private |
| 445 | SMB | File sharing; major ransomware target, never public |
| 8080 / 8443 | Alternate HTTP/HTTPS | Common for app servers |

**Security rule:** only ports 80 and 443 should ever be reachable from the internet, and ideally behind a load balancer or Front Door. Management ports (22, 3389) go through Bastion, VPN or Just-in-Time access. Database ports are only open from the app subnet.

### 3.6 Example: three-tier NSG design for Tomato

```bash
# NSGs
az network nsg create -g rg-vm-lab -n nsg-web
az network nsg create -g rg-vm-lab -n nsg-app
az network nsg create -g rg-vm-lab -n nsg-data

# Web tier: allow HTTPS from internet
az network nsg rule create -g rg-vm-lab --nsg-name nsg-web -n allow-https \
  --priority 100 --direction Inbound --access Allow --protocol Tcp \
  --source-address-prefixes Internet --destination-port-ranges 443

# App tier: only from the web subnet
az network nsg rule create -g rg-vm-lab --nsg-name nsg-app -n allow-from-web \
  --priority 100 --direction Inbound --access Allow --protocol Tcp \
  --source-address-prefixes 10.0.1.0/24 --destination-port-ranges 8080

# Data tier: SQL Server only from the app subnet
az network nsg rule create -g rg-vm-lab --nsg-name nsg-data -n allow-sql-from-app \
  --priority 100 --direction Inbound --access Allow --protocol Tcp \
  --source-address-prefixes 10.0.2.0/24 --destination-port-ranges 1433

# Attach to subnets
az network vnet subnet update -g rg-vm-lab --vnet-name vnet-lab -n snet-web  --network-security-group nsg-web
az network vnet subnet update -g rg-vm-lab --vnet-name vnet-lab -n snet-app  --network-security-group nsg-app
az network vnet subnet update -g rg-vm-lab --vnet-name vnet-lab -n snet-data --network-security-group nsg-data

# Inspect
az network nsg rule list -g rg-vm-lab --nsg-name nsg-web -o table
az network nsg rule list -g rg-vm-lab --nsg-name nsg-web --include-default -o table   # include default rules
```

## 4. How to access a VM

Your slide covers access using cloud CLI, remotely from Windows and remotely from Mac. Full picture:

### 4.1 Linux VM: SSH

**SSH** (Secure Shell, TCP 22) gives an encrypted terminal on a remote machine. It uses a **key pair**: the **private key** stays on your computer, the **public key** is stored on the VM in `~/.ssh/authorized_keys`. Anyone holding the private key can log in, so protect it.

```bash
# Create your own key pair (recommended)
ssh-keygen -t ed25519 -C 'azure-lab' -f ~/.ssh/azure_lab

# Create VM with your public key
az vm create -g rg-vm-lab -n vm-web --image Ubuntu2204 --size Standard_B1s \
  --admin-username azureuser --ssh-key-values ~/.ssh/azure_lab.pub

# Connect
ssh -i ~/.ssh/azure_lab azureuser@<public-ip>

# Private key must not be readable by others
chmod 600 ~/.ssh/azure_lab
```

| Client | How |
| --- | --- |
| **Mac / Linux terminal** | `ssh -i key azureuser@IP` (built in). Fix permissions with `chmod 400` or `600` |
| **Windows 10/11** | OpenSSH is built in: run the same `ssh` command in PowerShell or Windows Terminal. Or use WSL, or PuTTY (convert key with PuTTYgen) |
| **Azure Cloud Shell** | Browser shell in the portal, already signed in, has `az`, `ssh`, `kubectl` |

### 4.2 Windows VM: RDP

**RDP** (Remote Desktop Protocol, TCP 3389) gives a graphical desktop. Needs the username and password set at creation and an NSG rule allowing 3389 from **your IP only**.

- **From Windows:** press Win+R, run `mstsc`, enter the public IP, sign in. Or download the `.rdp` file from the portal (**Connect** blade).
- **From Mac:** install **Windows App** (formerly Microsoft Remote Desktop) from the App Store, add the PC with the IP, username and password.
- **From Linux:** `xfreerdp` or Remmina.

### 4.3 Cloud-native access methods used in real projects

| Method | What it gives you | Commands / notes |
| --- | --- | --- |
| **Azure CLI `az ssh vm`** | SSH using Entra ID, no manual key handling | `az extension add --name ssh` then `az ssh vm -g rg -n vm` |
| **Azure Bastion** | Browser or native-client RDP/SSH over TLS 443, VM needs no public IP | `az network bastion ssh --name bas --resource-group rg --target-resource-id <vm-id> --auth-type ssh-key --username azureuser --ssh-key key` |
| **Run Command** | Run a script on the VM through the Azure control plane, no network path needed | `az vm run-command invoke -g rg -n vm --command-id RunShellScript --scripts 'uptime'` |
| **Serial Console** | Console access when networking or SSH is broken | Portal; needs boot diagnostics |
| **Just-in-Time (JIT) access** | Opens 22/3389 only for a short approved window | Defender for Cloud |
| **VPN / ExpressRoute** | Private IP access from office/home | Point-to-site or site-to-site |

Equivalents: AWS **SSM Session Manager** and **EC2 Instance Connect Endpoint**; GCP **IAP TCP forwarding** (`gcloud compute ssh --tunnel-through-iap`).

**Entra ID login for Linux VMs** (no per-user keys): install the `AADSSHLoginForLinux` extension, enable a system-assigned identity, and give users the role **Virtual Machine Administrator Login** or **Virtual Machine User Login**.

### 4.4 Troubleshooting connection errors

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `Connection timed out` | NSG blocks port, no public IP, wrong IP, VM stopped, or on-prem firewall | Check NSG rule and priority, public IP, VM power state. Use the portal tool **Connection troubleshoot** / **IP flow verify** |
| `Connection refused` | Network is fine but nothing is listening (sshd or Apache not running) | `systemctl status sshd` via Run Command or Serial Console |
| `Permission denied (publickey)` | Wrong key or wrong username | Use the admin username set at creation, correct `-i` file; reset with `az vm user update` |
| `WARNING: UNPROTECTED PRIVATE KEY FILE` | Key file permissions too open | `chmod 600 keyfile` |
| RDP: ‘cannot connect’ | Port 3389 not allowed, VM not running, Windows firewall | Add NSG rule from your IP; check power state |
| Website not loading but SSH works | Port 80 not allowed in NSG, or Apache down, or OS firewall (ufw) | Add NSG rule, `systemctl status apache2`, `curl localhost` on the VM |
| VM running but no internet from it | Private subnet without outbound method | Add public IP or NAT Gateway |

Useful reset commands:

```bash
az vm user update -g rg-vm-lab -n vm-web -u azureuser --ssh-key-value "$(cat ~/.ssh/azure_lab.pub)"
az vm user reset-ssh -g rg-vm-lab -n vm-web          # reset SSH config
az vm repair --help                                    # repair extension (needs: az extension add -n vm-repair)
```

## 5. Hands-on labs

### Lab 1: Linux VM that serves a static website (the slide’s workflow)

**Goal:** create a VM, run a script at first boot that installs Apache, open port 80, and reach the site from the browser.

**Step 1: the bootstrap script** (`install-apache.sh`, same as your slide)

```bash
#!/bin/bash
sudo apt update -y
sudo apt install -y apache2
sudo systemctl start apache2
sudo systemctl enable apache2
echo '<html><h1>Welcome to Apache Web Server on Azure VM!</h1></html>' | sudo tee /var/www/html/index.html
```

What each line does: `apt update` refreshes package lists; `apt install -y apache2` installs the web server without prompting; `systemctl start` runs it now; `systemctl enable` makes it start automatically after every reboot; `tee` writes the HTML into Apache’s default web root `/var/www/html`, using `sudo` because the folder is root-owned.

**Step 2: create everything with the CLI**

```bash
az group create -n rg-vm-lab -l centralindia --tags env=lab owner=you

az vm create -g rg-vm-lab -n vm-web \
  --image Ubuntu2204 --size Standard_B1s \
  --admin-username azureuser --generate-ssh-keys \
  --public-ip-sku Standard \
  --custom-data install-apache.sh

# Allow HTTP in the NSG (creates a rule on the VM's NSG)
az vm open-port -g rg-vm-lab -n vm-web --port 80 --priority 900

# Get the IP and test
IP=$(az vm show -g rg-vm-lab -n vm-web -d --query publicIps -o tsv)
curl http://$IP
```

Open `http://<public-ip>` in the browser. If it works you see the welcome heading.

**Step 3: verify and debug on the VM**

```bash
ssh azureuser@$IP
cloud-init status --wait                    # done / running / error
sudo tail -n 50 /var/log/cloud-init-output.log   # output of your script
systemctl status apache2
curl -I localhost                           # works locally?
sudo ss -tlnp | grep :80                    # is something listening on 80?
```

**How custom data works.** Ubuntu images include **cloud-init**. Custom data (max 64 KB, base64-encoded by Azure) is passed to cloud-init at **first boot only**. A script starting with `#!/bin/bash` runs as root once. It is not re-run on reboot. For repeatable configuration after creation use the **Custom Script Extension** or **Run Command**.

**Alternative: cloud-init YAML** (preferred in real projects because it is declarative)

```yaml
#cloud-config
package_update: true
packages:
  - apache2
write_files:
  - path: /var/www/html/index.html
    content: '<html><h1>Welcome to Apache Web Server on Azure VM!</h1></html>'
runcmd:
  - systemctl enable --now apache2
```

```bash
az vm create -g rg-vm-lab -n vm-web2 --image Ubuntu2204 --size Standard_B1s \
  --admin-username azureuser --generate-ssh-keys --custom-data cloud-init.yaml
```

**Run a script after creation (extensions)**

```bash
az vm extension set -g rg-vm-lab --vm-name vm-web \
  --publisher Microsoft.Azure.Extensions --name CustomScript \
  --settings '{"commandToExecute":"apt-get update && apt-get install -y nginx"}'

az vm run-command invoke -g rg-vm-lab -n vm-web \
  --command-id RunShellScript --scripts 'systemctl status apache2 --no-pager'
```

### Lab 2: Windows VM with IIS and RDP

```bash
az vm create -g rg-vm-lab -n vm-win --image Win2022Datacenter --size Standard_B2s \
  --admin-username azureadmin --admin-password '<Str0ng-Passw0rd!>' --public-ip-sku Standard

az vm open-port -g rg-vm-lab -n vm-win --port 3389 --priority 910   # lab only; restrict source in real life
az vm open-port -g rg-vm-lab -n vm-win --port 80   --priority 920

az vm run-command invoke -g rg-vm-lab -n vm-win --command-id RunPowerShellScript \
  --scripts 'Install-WindowsFeature -Name Web-Server -IncludeManagementTools'
```

Then connect with `mstsc /v:<ip>` (Windows) or Windows App (Mac), and browse to `http://<ip>`.

Tighten the RDP rule to your own IP:

```bash
MYIP=$(curl -s https://ifconfig.me)
az network nsg rule update -g rg-vm-lab --nsg-name vm-winNSG -n open-port-3389 --source-address-prefixes $MYIP/32
```

### Lab 3: Day-2 operations (what admins do daily)

**Power states and billing**

| State | Meaning | Compute billed? | Disks billed? | Public IP (Standard) billed? |
| --- | --- | --- | --- | --- |
| Running | Normal | Yes | Yes | Yes |
| Stopped (allocated) | Shut down inside the OS only | **Yes** | Yes | Yes |
| **Deallocated** | Stopped from Azure, host released | **No** | Yes | Yes |
| Deleted | Gone | No | Only if disks left behind | Only if IP left behind |

```bash
az vm list -g rg-vm-lab -d -o table                          # state, IPs
az vm get-instance-view -g rg-vm-lab -n vm-web --query instanceView.statuses[1].displayStatus -o tsv
az vm stop       -g rg-vm-lab -n vm-web                      # stopped, STILL BILLING compute
az vm deallocate -g rg-vm-lab -n vm-web                      # stops compute billing
az vm start      -g rg-vm-lab -n vm-web
az vm restart    -g rg-vm-lab -n vm-web
az vm redeploy   -g rg-vm-lab -n vm-web                      # move to a new host (fixes some host/network issues)

# Resize (VM restarts; if the size isn't available on this host cluster, deallocate first)
az vm list-vm-resize-options -g rg-vm-lab -n vm-web -o table
az vm resize -g rg-vm-lab -n vm-web --size Standard_B2s
```

**Add and mount a data disk**

```bash
az vm disk attach -g rg-vm-lab --vm-name vm-web --name disk-data1 --new --size-gb 64 --sku Premium_LRS

# Inside the VM
lsblk                                   # find new disk, e.g. sdc
sudo parted /dev/sdc --script mklabel gpt mkpart xfspart xfs 0% 100%
sudo mkfs.xfs /dev/sdc1
sudo mkdir /data && sudo mount /dev/sdc1 /data
sudo blkid /dev/sdc1                    # copy UUID
echo 'UUID=<uuid> /data xfs defaults,nofail 0 2' | sudo tee -a /etc/fstab   # survive reboot
```

**Snapshot, image and clone**

```bash
DISKID=$(az vm show -g rg-vm-lab -n vm-web --query storageProfile.osDisk.managedDisk.id -o tsv)
az snapshot create -g rg-vm-lab -n snap-web-001 --source $DISKID

# Golden image from a VM (Linux: run 'sudo waagent -deprovision+user' inside first)
az vm deallocate -g rg-vm-lab -n vm-web
az vm generalize -g rg-vm-lab -n vm-web
az image create -g rg-vm-lab -n img-web-v1 --source vm-web
az vm create -g rg-vm-lab -n vm-web-clone --image img-web-v1 --admin-username azureuser --generate-ssh-keys
```

**Backup**

```bash
az backup vault create -g rg-vm-lab -n rsv-lab -l centralindia
az backup protection enable-for-vm -g rg-vm-lab --vault-name rsv-lab --vm vm-web --policy-name DefaultPolicy
```

### Lab 4: Managed identity (no secrets in code)

```bash
# 1. Give the VM an identity
az vm identity assign -g rg-vm-lab -n vm-web
PID=$(az vm show -g rg-vm-lab -n vm-web --query identity.principalId -o tsv)

# 2. Grant it a role on a storage account
SAID=$(az storage account show -n <storage-account> -g rg-vm-lab --query id -o tsv)
az role assignment create --assignee-object-id $PID --assignee-principal-type ServicePrincipal \
  --role 'Storage Blob Data Reader' --scope $SAID

# 3. Inside the VM: sign in as the VM and use Azure
az login --identity
az storage blob list --account-name <storage-account> -c food-images --auth-mode login -o table
```

**System-assigned vs user-assigned:** system-assigned is created and deleted with the VM (one VM only); user-assigned is a separate resource you can attach to many VMs and keeps its role assignments if a VM is replaced. Equivalent: AWS **IAM instance profile / role**, GCP **service account attached to the VM**.

### Lab 5: Clean up

```bash
az vm delete -g rg-vm-lab -n vm-web --yes        # leaves NIC, disks, public IP, NSG behind!
az resource list -g rg-vm-lab -o table           # see what remains
az group delete -n rg-vm-lab --yes --no-wait     # removes everything in the group
```

## 6. VM pricing models

Your slide lists four. Real projects combine several, and interviewers expect the full list.

| Model | How it works | Saving vs pay-as-you-go | Best for | AWS | GCP |
| --- | --- | --- | --- | --- | --- |
| **Pay-as-you-go** | Per-second billing, no commitment | Baseline | Dev/test, unpredictable or short-lived load | On-Demand | On-demand |
| **Reserved VM Instances** | Commit to a specific VM size and region for **1 or 3 years** | Up to about 70% | Steady, always-on production (databases, core app servers) | Reserved Instances | Committed Use Discounts (resource-based) |
| **Savings Plan for compute** | Commit to a fixed **hourly spend** for 1 or 3 years, applies across sizes, families and regions | Up to about 65% | Steady spend but changing VM shapes or regions | Compute Savings Plans | Flexible CUDs |
| **Spot VMs** | Buy Azure’s spare capacity; Azure can evict you | Up to about 90% | Fault-tolerant, interruptible work | Spot Instances | Spot VMs |
| **Dedicated Host** | Whole physical server only for your organisation | Costs more, but may save licences | Compliance, isolation, bring-your-own per-core licences | Dedicated Hosts | Sole-tenant nodes |
| **Azure Hybrid Benefit** | Use existing Windows Server / SQL Server / RHEL / SUSE licences with Software Assurance | Large on licensed software | Migrations with existing licences | License Included vs BYOL | Bring your own licence |
| **Dev/Test pricing** | Discounted rates on non-production subscriptions | Varies | Non-prod environments | Dev/test discounts via programs | Not equivalent |

**What drives a VM bill:** compute (size and hours while allocated), OS licence (Windows and SQL cost extra, Linux is cheaper), disks (provisioned size and tier, billed even when VM is stopped), public IPs, data **egress**, backups, and monitoring ingestion.

### Spot VMs in detail

- Azure can **evict** a Spot VM when it needs capacity back, or when the price exceeds your maximum price. You get a **best-effort notice of up to 30 seconds**. There is **no SLA**.
- **Eviction policy:** **Deallocate** (default; the VM is stopped and its disks are kept, you pay for disks and restart it manually) or **Delete** (VM and its disks are removed, no further charge).
- You can set a **max price**; `-1` means ‘pay up to the normal pay-as-you-go price’ so you are evicted only for capacity.
- Applications poll the **Scheduled Events** endpoint on the VM to learn about the coming eviction and save state.
- Good fit: CI/CD agents, batch jobs, image/video processing, rendering, stateless workers behind a queue, dev/test. Bad fit: single databases, anything needing guaranteed uptime.

```bash
az vm create -g rg-vm-lab -n vm-spot --image Ubuntu2204 --size Standard_D2s_v5 \
  --priority Spot --max-price -1 --eviction-policy Delete \
  --admin-username azureuser --generate-ssh-keys

# Inside the VM: check for upcoming eviction
curl -s -H Metadata:true 'http://169.254.169.254/metadata/scheduledevents?api-version=2020-07-01'
```

### Dedicated Host in detail

An Azure Dedicated Host is a physical server whose hardware is **not shared with any other customer**. You pay for the whole host regardless of how many VMs you place on it. Use it for regulatory isolation, to control maintenance timing, or to use per-core licences more efficiently.

```bash
az vm host group create -g rg-vm-lab -n hg-prod --platform-fault-domain-count 1 --zone 1
az vm host create -g rg-vm-lab --host-group hg-prod -n host1 --sku <host-sku> --platform-fault-domain 0
az vm create -g rg-vm-lab -n vm-dedicated --image Ubuntu2204 --size <size-supported-by-host> \
  --host host1 --admin-username azureuser --generate-ssh-keys
```

(Pick the host SKU from the portal or the Azure Dedicated Host documentation, since supported types change by region.)

### Cost-saving checklist (frequent scenario question)

1. **Right-size:** use Azure Advisor and Monitor metrics, downsize idle VMs.
2. **Deallocate** dev/test VMs outside working hours (auto-shutdown, or a schedule through Automation / Logic Apps / Functions).
3. **Commit** steady workloads with reservations or a savings plan.
4. Move interruptible workloads to **Spot**.
5. Use **Hybrid Benefit** for existing licences.
6. Prefer **Standard SSD** for non-prod, **delete orphaned disks and IPs**.
7. Use **autoscale** to match capacity to demand.
8. Add **tags and budgets** so every cost has an owner.

```bash
az vm auto-shutdown -g rg-vm-lab -n vm-web --time 1900 --email you@company.com   # shut down at 19:00 UTC daily
az disk list --query "[?managedBy==null].{name:name,rg:resourceGroup,sizeGb:diskSizeGb}" -o table   # unattached disks
az network public-ip list --query "[?ipConfiguration==null].{name:name,rg:resourceGroup}" -o table   # unused public IPs
```

## 7. High availability, scaling and day-2 operations

### 7.1 Availability options and SLAs

| Option | What it protects against | Typical SLA | AWS | GCP |
| --- | --- | --- | --- | --- |
| **Single VM** (Premium SSD / Ultra / Premium SSD v2 disks) | Nothing beyond platform basics | 99.9% | Single instance | Single instance |
| **Availability Set** (2 or more VMs) | Rack and maintenance failures within one data centre using **fault domains** (separate racks) and **update domains** (rolling maintenance) | 99.95% | Spread placement group | Spread policy |
| **Availability Zones** (2 or more VMs in different zones) | Whole data-centre failure | 99.99% | Multi-AZ | Multi-zone |
| **Cross-region DR** (Site Recovery / replicated deployment) | Whole region failure | Depends on your design | Elastic Disaster Recovery | Multi-region design |

New designs use **Availability Zones** or **Virtual Machine Scale Sets in Flexible orchestration mode** spread across zones. Availability sets are still supported but are the older approach. You cannot put a VM into an availability set or zone after creation; choose at creation time.

```bash
az vm create -g rg-vm-lab -n vm-web-z1 --image Ubuntu2204 --size Standard_B2s --zone 1 \
  --admin-username azureuser --generate-ssh-keys
az vm availability-set create -g rg-vm-lab -n avset-web --platform-fault-domain-count 2 --platform-update-domain-count 5
```

### 7.2 Scale sets (VMSS) and load balancing

A **VM Scale Set** manages a group of identical VMs that scale out and in automatically from a rule or schedule. Put them behind a **load balancer** (layer 4) or **Application Gateway / Front Door** (layer 7, WAF).

```bash
az vmss create -g rg-vm-lab -n vmss-web --image Ubuntu2204 --vm-sku Standard_B2s \
  --instance-count 2 --zones 1 2 3 --orchestration-mode Flexible \
  --admin-username azureuser --generate-ssh-keys --custom-data install-apache.sh \
  --lb lb-web --upgrade-policy-mode Automatic

az monitor autoscale create -g rg-vm-lab --resource vmss-web \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name autoscale-web --min-count 2 --max-count 10 --count 2

az monitor autoscale rule create -g rg-vm-lab --autoscale-name autoscale-web \
  --condition 'Percentage CPU > 70 avg 5m' --scale out 2
az monitor autoscale rule create -g rg-vm-lab --autoscale-name autoscale-web \
  --condition 'Percentage CPU < 30 avg 10m' --scale in 1
```

**Scale up vs scale out:** scale **up** (vertical) = bigger VM size, needs a restart, has an upper limit. Scale **out** (horizontal) = more VMs, no downtime, needs a stateless or shared-state design. Production prefers scale out.

### 7.3 Operations a VM owner must do

| Area | What | Azure tool | AWS | GCP |
| --- | --- | --- | --- | --- |
| Patching | OS updates on schedule | **Azure Update Manager** | Systems Manager Patch Manager | VM Manager OS patch management |
| Backup | Daily policy, retention, restore tests | **Azure Backup** (Recovery Services vault) | AWS Backup | Backup and DR Service |
| Disaster recovery | Replicate VM to another region | **Azure Site Recovery** | Elastic Disaster Recovery | Backup and DR / multi-region design |
| Monitoring | CPU, memory, disk, logs, alerts | **Azure Monitor**, VM Insights, Azure Monitor Agent, Log Analytics | CloudWatch + agent | Cloud Monitoring + Ops Agent |
| Boot problems | Screenshot and serial log | **Boot diagnostics**, Serial Console | EC2 console output | Serial port output |
| Security posture | Vulnerabilities, JIT, endpoint protection | **Defender for Cloud / Defender for Servers** | Inspector, GuardDuty | Security Command Center |
| Config management | Fleet commands, desired state | Run Command, Automanage, Ansible | SSM Run Command | OS Config |
| Cost | Right-sizing advice | **Azure Advisor**, Cost Management | Compute Optimizer | Recommender |

**Planned maintenance:** Azure patches hosts regularly. Most updates are memory-preserving (a pause of a few seconds). Rarely a reboot is needed; availability zones, sets and scale sets stagger this so not all your VMs go down together. **Redeploy** moves a VM to a different physical host if a host-level problem is suspected.

**Security baseline for any VM:** no public IP unless needed; NSG default-deny with narrow allow rules; access through Bastion or JIT; SSH keys or Entra ID login, no shared passwords; managed identity instead of keys in code; OS patching automated; disk encryption (platform-managed keys at minimum, customer-managed keys for regulated workloads); Defender for Servers; logs shipped to Log Analytics; Azure Policy to enforce all of this.

## 8. Real-world project examples

### 8.1 Tomato: lift-and-shift of the legacy order-management system

Tomato’s original order system is a Windows Server application with a SQL Server database running in an office server room. The goal is to move it to Azure with minimal code changes first, then modernise to PaaS later.

**Network plan**

| Subnet | CIDR | Contains | NSG policy |
| --- | --- | --- | --- |
| `snet-web` | `10.0.1.0/24` | Web VMs (VM scale set across zones) | Inbound 443 from Application Gateway only |
| `snet-app` | `10.0.2.0/24` | Application VMs | Inbound 8080 from `snet-web` only |
| `snet-data` | `10.0.3.0/24` | SQL Server VMs | Inbound 1433 from `snet-app` only; no internet |
| `AzureBastionSubnet` | `10.0.254.0/26` | Azure Bastion (name and minimum size are mandatory) | Managed |
| `GatewaySubnet` | `10.0.255.0/27` | VPN Gateway to the office | Managed |

**Build steps (what you would actually do)**

1. Create subscription layout, resource groups (`rg-tomato-net`, `rg-tomato-web`, `rg-tomato-data`), tags and budgets.
2. Create the VNet, subnets and NSGs (section 3.6), plus a **NAT Gateway** on `snet-app` and `snet-data` for updates.
3. Build a **golden image** (Windows Server + IIS + app prerequisites + hardening) with Packer or Azure Image Builder and publish it in an **Azure Compute Gallery**.
4. Deploy the web tier as a **VM scale set across 3 zones** behind **Application Gateway with WAF**, with CPU-based autoscale for the 8 PM dinner peak.
5. Deploy the app tier VMs in two zones behind an **internal load balancer**.
6. Deploy **SQL Server VMs** (E-series, Premium SSD v2 or Ultra for data and log disks, separate disks for data, log and tempdb) in two zones with an **Always On availability group**, using **Hybrid Benefit** for the licence. (Later migration target: Azure SQL Managed Instance.)
7. Admin access only through **Azure Bastion**; DBAs use Entra ID login with MFA.
8. **Managed identities** let VMs read secrets from **Key Vault** and images from **Blob Storage** without stored keys.
9. **Azure Backup** daily with 30-day retention; **Site Recovery** replicates app VMs to the paired region; quarterly failover test.
10. **Azure Monitor** alerts on CPU, disk latency, failed logins and low free disk; **Update Manager** patches on a weekend window, one zone at a time.
11. **Reserved instances** for the always-on SQL and app VMs (1 year first, 3 years once stable); web tier uses pay-as-you-go plus autoscale.

**Why VMs and not PaaS here?** The app depends on Windows services and local file paths that cannot run on App Service without code changes. VMs make the migration fast, and the team modernises piece by piece afterwards.

### 8.2 Self-hosted CI/CD runners on Spot VMs

Tomato’s mobile and web builds are slow and expensive on hosted runners. The DevOps team runs **GitHub Actions self-hosted runners** on a **Spot VM scale set** (autoscaled from the job queue). If a Spot VM is evicted the job is retried on another runner. Runners use a managed identity to push images to **Azure Container Registry**. Savings are often 60 to 90% against on-demand pricing.

### 8.3 Food-image processing batch on Spot VMs

Restaurants upload large photos to **Blob Storage**. A message per photo goes to a **Storage Queue / Service Bus**. A pool of **Spot VMs** pulls messages, resizes and compresses images, writes thumbnails back to Blob, and deletes the message. If a VM is evicted, the message reappears after its visibility timeout and another worker picks it up, so the design tolerates eviction by being **idempotent and queue-driven**.

### 8.4 Dev boxes and a jump host

Developers get B-series Linux VMs with **auto-shutdown at night**, created from a template so each is identical. A small hardened **jump VM** (or Bastion alone) in the management subnet is the only way to reach private resources.

## 9. AWS and GCP equivalents for everything in this topic

### 9.1 Concepts

| Concept | Azure | AWS | GCP |
| --- | --- | --- | --- |
| Virtual machine | Virtual Machine | EC2 instance | Compute Engine instance |
| VM size | Size / SKU (`Standard_D4s_v5`) | Instance type (`m6i.xlarge`) | Machine type (`n2-standard-4`) |
| OS template | Image (Marketplace / Compute Gallery) | AMI | Image / image family |
| Persistent disk | Managed Disk | EBS volume | Persistent Disk / Hyperdisk |
| Temp local disk | Temporary disk | Instance store | Local SSD |
| Firewall | NSG | Security Group (+ NACL) | VPC firewall rules |
| Reusable role label for rules | Application Security Group | Security group reference | Network tags / service accounts |
| Private network | VNet | VPC | VPC |
| Static public IP | Standard Public IP | Elastic IP | Static external IP |
| Outbound NAT | NAT Gateway | NAT Gateway | Cloud NAT |
| Secure admin access | Azure Bastion / JIT | SSM Session Manager / EC2 Instance Connect Endpoint | IAP TCP forwarding / OS Login |
| Boot script | Custom data (cloud-init) / Custom Script Extension | User data | Startup script (metadata) |
| VM identity | Managed identity | IAM instance profile / role | Attached service account |
| Auto-scaling group | VM Scale Set | Auto Scaling Group | Managed Instance Group |
| Spread across data-centre buildings | Availability Zones | Availability Zones | Zones |
| Low-priced interruptible VM | Spot VM (about 30 s notice) | Spot Instance (2 min notice) | Spot VM (about 30 s notice) |
| Exclusive hardware | Dedicated Host | Dedicated Host | Sole-tenant node |
| Commitment discount | Reservation / Savings Plan | Reserved Instance / Savings Plan | Committed Use Discount |
| Run script remotely | Run Command | SSM Run Command | OS Config / `gcloud compute ssh --command` |
| Golden image pipeline | Azure Image Builder / Packer | EC2 Image Builder / Packer | Packer / custom image |

### 9.2 The same lab on all three clouds

| Task | Azure CLI | AWS CLI | gcloud |
| --- | --- | --- | --- |
| Create VM with boot script | `az vm create -g rg -n vm --image Ubuntu2204 --size Standard_B1s --custom-data install-apache.sh --generate-ssh-keys` | `aws ec2 run-instances --image-id <ami> --instance-type t3.micro --key-name <key> --security-group-ids <sg> --user-data file://install-apache.sh` | `gcloud compute instances create vm --zone=asia-south1-a --machine-type=e2-micro --image-family=ubuntu-2204-lts --image-project=ubuntu-os-cloud --tags=http-server --metadata-from-file=startup-script=install-apache.sh` |
| Open port 80 | `az vm open-port -g rg -n vm --port 80` | `aws ec2 authorize-security-group-ingress --group-id <sg> --protocol tcp --port 80 --cidr 0.0.0.0/0` | `gcloud compute firewall-rules create allow-http --allow=tcp:80 --target-tags=http-server` |
| List VMs | `az vm list -d -o table` | `aws ec2 describe-instances` | `gcloud compute instances list` |
| SSH in | `ssh azureuser@<ip>` or `az ssh vm` | `ssh -i key.pem ubuntu@<ip>` or `aws ssm start-session --target <id>` | `gcloud compute ssh vm --zone=<zone>` |
| Stop and stop compute billing | `az vm deallocate` | `aws ec2 stop-instances` | `gcloud compute instances stop` |
| Start | `az vm start` | `aws ec2 start-instances` | `gcloud compute instances start` |
| Resize | `az vm resize --size <size>` | stop, `aws ec2 modify-instance-attribute --instance-type`, start | stop, `gcloud compute instances set-machine-type`, start |
| Create and attach disk | `az vm disk attach --new --size-gb 64` | `aws ec2 create-volume` then `attach-volume` | `gcloud compute disks create` then `instances attach-disk` |
| Snapshot | `az snapshot create` | `aws ec2 create-snapshot` | `gcloud compute disks snapshot` |
| Image from VM | `az image create` | `aws ec2 create-image` | `gcloud compute images create` |
| Spot VM | `az vm create --priority Spot --eviction-policy Delete` | `aws ec2 run-instances --instance-market-options MarketType=spot` | `gcloud compute instances create --provisioning-model=SPOT` |
| Delete | `az vm delete` (disks remain) | `aws ec2 terminate-instances` | `gcloud compute instances delete` |

**Important nuance:** on Azure, stopping from inside the OS keeps billing; you must **deallocate**. On AWS and GCP a normal ‘stop’ already releases compute billing (disks still billed). Interviewers love this cross-cloud trap.

## 10. Interview questions and model answers

### Basic

1. **What is an Azure Virtual Machine and when would you use one?** An IaaS compute resource that gives you a full virtual server with your choice of OS. Use it for lift-and-shift, legacy or OS-dependent software, custom networking or agents, self-managed databases, and dev/test. If the app can run on App Service, containers or Functions, prefer those because they reduce operations work.
2. **What do you need to decide when creating a VM?** Size, image (OS), disks, networking (VNet, subnet, public IP, NSG), authentication (SSH key or password), optional managed identity and custom data, plus region and availability option.
3. **What is a VM size and how do you choose one?** A SKU that defines vCPUs, memory, disk and network limits. Start with B (small/dev) or D (general), E for memory-heavy, F for CPU-heavy, N for GPU. Measure real usage and right-size using Azure Advisor.
4. **Explain B-series.** Burstable VMs with a baseline CPU level that accumulate credits while idle and spend them when busy. Cheap, but throttled to baseline when credits run out, so unsuitable for sustained high CPU.
5. **What is an NSG and where can it be attached?** A stateful rule-based firewall that filters traffic by source, destination, port and protocol. Attached to a subnet and/or a NIC. Region specific.
6. **What are the default NSG rules?** Inbound: allow VNet traffic, allow Azure load balancer probes, deny everything else. Outbound: allow VNet traffic, allow internet, deny the rest. All have priorities of 65000 and above and cannot be removed.
7. **What does stateful mean?** If the inbound request is allowed, the response is automatically allowed without a matching outbound rule.
8. **How does NSG rule priority work?** Numbers 100 to 4096; lowest number is evaluated first; the first matching rule decides and evaluation stops.
9. **How do you connect to a Linux VM? A Windows VM?** Linux: SSH on port 22 with a key pair. Windows: RDP on port 3389. Production: through Azure Bastion, JIT access or VPN, not an open public port.
10. **What is cloud-init / custom data?** A first-boot automation mechanism. The script or YAML you pass is run once when the VM first starts, used to install packages and write config. It does not re-run on reboot.
11. **What ports should you know?** 80 HTTP, 443 HTTPS, 22 SSH/SFTP, 21 FTP, 25 SMTP, 53 DNS, 3389 RDP, 3306 MySQL, 5432 PostgreSQL, 1433 SQL Server.

### Intermediate

12. **Difference between stopping and deallocating a VM?** Stopping from inside the OS leaves the VM allocated on the host and compute billing continues. Deallocating (`az vm deallocate` or Stop in the portal) releases the host, stops compute charges, and may change a dynamic IP. Disks are billed either way.
13. **What happens to disks, NIC and public IP when you delete a VM?** By default they remain unless delete options were set, and they keep billing. Delete the resource group or remove them explicitly.
14. **What is the temporary disk and what must never be stored on it?** A local SSD on the host used for swap and cache. Its contents are lost on redeploy, deallocate or host move. Never store application data on it.
15. **Compare disk types.** Standard HDD (cheap, infrequent access), Standard SSD (web/dev), Premium SSD (production), Premium SSD v2 (tunable IOPS and throughput), Ultra Disk (highest performance, lowest latency, tier-1 databases).
16. **How do you make a VM highly available?** Run two or more VMs across Availability Zones (99.99% SLA) behind a load balancer; use zone-redundant disks and zone-aware databases; back up with Azure Backup; replicate to another region with Site Recovery for DR.
17. **Availability Set vs Availability Zone vs Scale Set.** Availability sets spread VMs across racks and maintenance windows in one data centre. Zones spread across separate buildings. Scale sets add automatic scaling and are the modern way to run groups of identical VMs, with flexible orchestration across zones.
18. **What is a managed identity?** An identity Azure creates for a resource so it can authenticate to Entra-protected services without stored credentials. System-assigned (tied to the VM) or user-assigned (shared, independent lifecycle).
19. **Custom Script Extension vs custom data vs Run Command.** Custom data runs once at first boot. The Custom Script Extension can run after creation and be re-applied. Run Command runs ad-hoc scripts through the control plane, even if the network path is broken.
20. **Reserved Instance vs Savings Plan vs Spot.** Reservation: fixed size and region commitment, deepest discount on steady workloads. Savings Plan: hourly spend commitment, flexible across sizes and regions. Spot: spare capacity at the deepest discount but evictable with about 30 seconds notice and no SLA.
21. **What is a Dedicated Host and when would you use it?** A whole physical server for your organisation. For compliance isolation, control over maintenance, or per-core licence optimisation.
22. **Spot eviction policies?** Deallocate (default; disks kept, restart later) or Delete (VM and disks removed). Design workloads to be idempotent and checkpoint on the Scheduled Events notice.
23. **How can you resize a VM and what is the catch?** `az vm resize`. It reboots the VM. If the new size is not available on the current hardware cluster you must deallocate first. Quota and regional availability also matter. Moving between families can fail if the disk type or generation is unsupported.
24. **Can you change the region or availability zone of a VM?** Not directly. Snapshot or image the disks and recreate in the target region or zone, use Azure Resource Mover, or Site Recovery for a managed move.
25. **NSG on subnet vs NIC?** Both are evaluated. Inbound hits the subnet NSG first, then the NIC NSG. Outbound hits the NIC NSG first, then the subnet. Best practice is subnet-level rules for consistency and NIC-level only for exceptions.
26. **NSG vs Azure Firewall vs WAF?** NSG: free L3/L4 filtering. Azure Firewall: managed, central L3 to L7 filtering with FQDN rules and threat intelligence. WAF: L7 protection for HTTP traffic.
27. **Compare with AWS and GCP.** Azure NSG is stateful and attaches to subnet or NIC. AWS uses stateful Security Groups plus stateless NACLs. GCP uses VPC firewall rules targeted by tags or service accounts. Azure Bastion is AWS SSM Session Manager and GCP IAP tunnelling. A stopped AWS or GCP instance stops compute billing automatically, but an Azure VM must be deallocated.

### Scenario and troubleshooting

28. **You created a VM and installed Apache but the website does not load. Steps?** (1) `curl localhost` on the VM to confirm Apache works. (2) `systemctl status apache2` and `ss -tlnp | grep 80`. (3) Check the NSG has an inbound allow on port 80 with a priority lower (higher precedence) than any deny. (4) Confirm the VM has a public IP and you use the right IP. (5) Check OS firewall (`ufw status`). (6) Use IP Flow Verify / Connection Troubleshoot in Network Watcher.
29. **SSH says connection timed out.** Network path problem: NSG missing port 22 allow, no public IP, VM deallocated, wrong IP, or a corporate firewall. **Connection refused** means the path is fine but sshd is not listening. **Permission denied (publickey)** means wrong key or username.
30. **A VM in a new private subnet cannot reach the internet.** Newer VNets are private by default, so there is no default outbound access. Attach a NAT Gateway to the subnet, or give the VM a public IP, or route through Azure Firewall / use load balancer outbound rules.
31. **The monthly bill doubled. What do you check?** Cost Management grouped by resource and tag; look for forgotten or oversized VMs, stopped-but-not-deallocated VMs, unattached premium disks, unused Standard public IPs, large egress, Windows and SQL licence costs, and Spot vs pay-as-you-go mix. Then right-size, schedule shutdowns, apply reservations and set budgets.
32. **How do you securely give developers access to production VMs?** No public IP; access through Azure Bastion or JIT; Entra ID login with MFA and RBAC roles (Virtual Machine User Login); session recording/logging; NSG restricted; secrets from Key Vault via managed identity.
33. **How do you patch 200 VMs without downtime?** Azure Update Manager with maintenance configurations and schedules, patch one zone or scale-set batch at a time behind the load balancer, test in staging first, use pre/post scripts, and take snapshots or backups beforehand.
34. **Design a highly available three-tier app on VMs.** Web tier in a zone-spread VM scale set behind Application Gateway with WAF; app tier behind an internal load balancer; database on zone-redundant managed service or clustered SQL VMs; NSGs per tier; Bastion for admin; NAT Gateway for outbound; Backup plus Site Recovery; Monitor alerts; autoscale on the web tier.
35. **A Spot VM disappeared. Why and how do you prevent data loss?** It was evicted for capacity or price. Use Deallocate to keep disks, checkpoint work on the Scheduled Events notice, keep job state in a queue or database, and run only interruptible workloads there.
36. **You need identical VMs created quickly and consistently. How?** Build a golden image with Packer or Azure Image Builder, publish to Azure Compute Gallery with versions, and create VMs or scale sets from it, plus cloud-init for per-instance config; define everything in Bicep or Terraform.
37. **How would you migrate an on-prem VM to Azure?** Assess with Azure Migrate, replicate with Azure Migrate or Site Recovery, run a test failover, plan network and IP changes, cut over, then optimise size and consider modernising to PaaS.

## 11. Common mistakes cheat sheet

| Mistake | Consequence | Correct habit |
| --- | --- | --- |
| Opening 22 / 3389 to `0.0.0.0/0` | Brute-force attacks within minutes | Bastion, JIT or restrict to your IP |
| Using a password for Linux admin | Weak authentication | SSH keys or Entra ID login |
| Stopping inside the OS | Still paying for compute | `az vm deallocate` |
| Deleting only the VM | Orphaned disks and public IPs keep billing | Delete resource group or set delete options |
| Storing data on the temp disk | Data loss on deallocate/redeploy | Use data disks |
| Using B-series for sustained heavy CPU | Throttling after credits are used | Move to D-series |
| Forgetting an NSG rule priority | A deny with a lower number overrides your allow | Plan priority bands (for example 100s allow, 4000s deny) |
| Trusting `Connection refused` means a firewall issue | Wasted time | Refused = service not running, timeout = network |
| Putting secrets in custom data | Custom data is readable inside the VM and via metadata | Key Vault + managed identity |
| One big VM for everything | Single point of failure | Scale out across zones |
| Not testing restore | Backups that fail when needed | Scheduled restore drills |
| Running without a quota check | Allocation failure at create or scale-out | `az vm list-usage`, request quota early |

## 12. One-page revision sheet

- Azure VM = **IaaS**: you manage OS, patches, software and data; Azure manages hardware and hypervisor.
- A VM is a **bundle**: VM, OS disk, data disks, temp disk, NIC, public IP, NSG, VNet/subnet (plus Bastion, NAT Gateway).
- Creation choices: **size, image, disks, networking, authentication, managed identity, custom data, public IP**.
- Sizes: **B** burstable, **D** general, **E** memory, **F** compute, **L** storage, **M** huge memory, **N** GPU. Name `Standard_D4s_v5` = family D, 4 vCPU, premium-storage capable, version 5.
- Disks: Standard HDD, Standard SSD, Premium SSD, Premium SSD v2, Ultra. Temp disk is not persistent.
- **NSG:** region specific, stateful, allow and deny, subnet or NIC, priority 100 to 4096 (lowest first), defaults at 65000+.
- **Ports:** 80, 443, 22, 3389, 21, 25, 53, 3306, 5432, 1433.
- Access: SSH key (Linux), RDP (Windows), Bastion, Run Command, `az ssh vm`, Cloud Shell.
- New VNets are **private by default**: add a public IP, NAT Gateway or firewall for outbound.
- Pricing: pay-as-you-go, **Reserved** (1/3 yr), **Savings Plan**, **Spot** (30 s notice, no SLA), **Dedicated Host**, **Hybrid Benefit**.
- **Deallocate** to stop compute billing; disks always bill.
- HA: zones (99.99%) beat availability sets (99.95%) beat single VM (99.9%). Use scale sets plus load balancer plus autoscale.
- Day 2: Backup, Site Recovery, Update Manager, Monitor, Defender.
- First-boot setup: **custom data / cloud-init** (once); later changes: Custom Script Extension or Run Command.
- Multi-cloud: VM = EC2 = Compute Engine; NSG = Security Group = VPC firewall rule; Bastion = SSM = IAP; managed identity = instance profile = service account.
