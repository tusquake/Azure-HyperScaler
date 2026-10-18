# Azure Handbook – Part 3: Virtual Networks (VNet)

## 0. What was verified or corrected from the slides

| Slide said | Current reality (use this in interviews) |
| --- | --- |
| VNet peering ‘supports transitive peering’ | **Incorrect.** Peering is **non-transitive**: if A↔B and B↔C are peered, A cannot reach C. Transit is built with a hub (Azure Firewall/NVA + route tables), gateway transit, Virtual WAN, or Azure Virtual Network Manager. See section 10. |
| Region: ‘India South Central’ | This region now exists: **Hyderabad**, generally available since **6 August 2026**, with **3 Availability Zones**. India now has four Azure regions (Central India/Pune, South India/Chennai, West India/Mumbai, India South Central/Hyderabad). Not every service is available in a new region on day one, and sources differ on its region pair, so check the Azure region list and `az account list-locations` before you design. |
| ‘Within the VPC, you can create subnets’ and AWS-style ‘Instances’ icons | ‘VPC’ and ‘instances’ are **AWS** words. In Azure it is **VNet** and **virtual machines**. Easy to mix up in an interview. |
| public-subnet and private-subnet | Azure has **no formal public/private subnet type** and no Internet Gateway. A subnet behaves ‘public’ only because its VMs have public IPs and the NSG allows traffic. Since 31 March 2026, **new VNets create subnets as private by default** (no default outbound internet). |
| First VM in `10.0.1.0/24` gets `10.0.1.4` | Correct, and a favourite question: Azure reserves `.0` to `.3` and the last address, so the first usable address is `.4`. |
| ‘All resources in the same VNet can communicate, even in different subnets’ | Correct. It comes from the default NSG rule `AllowVnetInBound`. You restrict it with NSGs. |
| Route table type ‘Virtual Network’ | In the CLI the next hop type is called `VnetLocal`. The others are `VirtualAppliance`, `VirtualNetworkGateway`, `Internet`, `None`. |
| ‘VNet 100 IPs, 50 + 50’ | Fine for intuition. Real CIDR blocks are powers of two (`/24` = 256), and Azure reserves 5 per subnet. |
| NSG definition | Correct. Full NSG detail is in Part 2, section 3. Here it is applied to a VNet design. |

## 1. Networking fundamentals you must know first

A **network** is a group of devices connected so they can communicate and share data. Your slide uses a home Wi-Fi router connecting a laptop, phone and TV, and an office network with a switch and router.

| Term | Meaning | Home or office analogy |
| --- | --- | --- |
| **IP address** | Unique address of a device on a network (IPv4: four numbers such as `10.0.1.4`, 32 bits) | House number |
| **Private IP** | Address valid only inside a private network (ranges below), not routable on the internet | Flat number inside a society |
| **Public IP** | Internet-routable address | The society’s street address |
| **Subnet** | A smaller block of a network | One building (block) of the society |
| **Switch** | Connects devices inside a network | Lift/corridor |
| **Router / gateway** | Forwards traffic between networks | Main gate |
| **DNS** | Converts names (`tomato.com`) to IPs | Phone book |
| **DHCP** | Hands out IP addresses automatically (in Azure, the platform does this for you) | Reception allotting flat numbers |
| **NAT** | Lets many private addresses share a public IP for outbound traffic | One office phone line for everyone calling out |
| **Firewall** | Allows or blocks traffic by rules | Security guard |
| **Port** | Door number on a device for a specific service (80, 443, 22) | Room number |
| **TCP / UDP** | Reliable connection-based / fast connectionless transport | Registered post / postcard |
| **Protocol** | Agreed rules for communication (HTTP, SSH, DNS) | Language |

**Private IPv4 ranges (RFC 1918)**

| Range | CIDR | Addresses |
| --- | --- | --- |
| `10.0.0.0` to `10.255.255.255` | `10.0.0.0/8` | 16,777,216 |
| `172.16.0.0` to `172.31.255.255` | `172.16.0.0/12` | 1,048,576 |
| `192.168.0.0` to `192.168.255.255` | `192.168.0.0/16` | 65,536 |

Use these for VNets. Azure does not allow some ranges in a VNet address space: multicast (`224.0.0.0/4`), broadcast (`255.255.255.255/32`), loopback (`127.0.0.0/8`), link-local (`169.254.0.0/16`) and the Azure DNS/metadata address `168.63.129.16/32`.

## 2. What is an Azure Virtual Network and why do we need it?

### 2.1 Definition

A **Virtual Network (VNet)** is your own **private, isolated, software-defined network** in Azure. Resources such as VMs connect to it and communicate securely with each other, with the internet, and with other networks (other VNets, your on-premises network). Equivalent: AWS **VPC**, GCP **VPC network**.

### 2.2 The problem it solves (the my-webapp story from your slides)

1. You have a web app with a **frontend VM** (web server, `10.0.1.4`) and a **backend VM** (app server, `10.0.2.4`).
2. **Users on the internet** must reach the frontend, and only the frontend.
3. The frontend must talk to the backend, privately and fast.
4. Nobody else (for example an admin on the internet) should reach the backend directly, which your slide shows with the red cross.

Without a private network, every VM would sit on shared infrastructure with public exposure. A VNet gives you an **isolated boundary** in which you decide who talks to whom.

### 2.3 What a VNet gives you

| Capability | Meaning |
| --- | --- |
| **Isolation** | Traffic is private to your VNet; other customers cannot see it |
| **Segmentation** | Split into subnets (web, app, data) with different rules |
| **Private communication** | Resources talk over private IPs, no internet needed |
| **Controlled internet access** | You choose inbound (public IP + NSG) and outbound (NAT Gateway, firewall) |
| **Hybrid connectivity** | Connect to on-premises through VPN Gateway or ExpressRoute |
| **VNet-to-VNet** | Peering, VPN, or Virtual WAN |
| **Traffic control** | NSGs, route tables, Azure Firewall |
| **Service access** | Private Endpoints and Service Endpoints to PaaS services such as Storage and SQL |
| **Name resolution** | Built-in Azure DNS or your own DNS / Private DNS zones |

### 2.4 Key properties (interview facts)

- A VNet is **scoped to one region** and one subscription. It spans **all Availability Zones** in that region. It cannot span regions, so you use peering or gateways between regions.
- **Creating a VNet is free.** You pay for things attached to it: public IPs, NAT Gateway, VPN/ExpressRoute gateways, Azure Firewall, Bastion, peering data transfer, Private Endpoints.
- By default **everything inside a VNet can talk to everything else**, even across subnets (default NSG rule `AllowVnetInBound`). **Nothing from the internet can get in** unless you add a public IP and an allowing rule.
- You can add or change address spaces later, but not carelessly. Subnets in use can’t be resized, and a VNet can’t be moved to another region.
- No broadcast or multicast inside a VNet. Layer 2 is virtual and handled by the platform.
- Name resolution works automatically through Azure DNS. Reserved addresses `168.63.129.16` (DNS, DHCP, health probes) is the platform virtual IP.

### 2.5 What can live inside a VNet

Virtual machines and scale sets, AKS nodes, load balancers, Application Gateway, Azure Firewall, NAT Gateway, Bastion, VPN and ExpressRoute gateways, Private Endpoints, App Service (through VNet integration), Azure SQL Managed Instance, Container Apps environments.

```bash
# Quick first look
az network vnet create -g rg-net-lab -n vnet-lab -l centralindia \
  --address-prefixes 10.0.0.0/16 \
  --subnet-name snet-frontend --subnet-prefixes 10.0.1.0/24
az network vnet list -o table
az network vnet show -g rg-net-lab -n vnet-lab --query '{space:addressSpace.addressPrefixes, subnets:subnets[].{name:name,cidr:addressPrefix}}' -o json
```

## 3. VNet address space and CIDR

### 3.1 Address space

When you create a VNet you choose an **address space**: the block of private IPs available to the whole VNet, such as `10.0.0.0/16` (65,536 addresses). A VNet can have **several** address spaces. Subnets are carved out of it.

### 3.2 CIDR notation

**CIDR** (Classless Inter-Domain Routing) writes a range as `network-address/prefix-length`. An IPv4 address is **32 bits**. The prefix says how many leading bits are the **network** part; the remaining bits are for **hosts**.

```
10.0.0.0/16
  first 16 bits  -> network (fixed)
  last  16 bits  -> hosts   (free)   => 2^(32-16) = 65,536 addresses

10.0.1.0/24 in binary:
  00001010.00000000.00000001.00000000
  |------ 24 network bits -----|-8 host bits-|
  range: 10.0.1.0 to 10.0.1.255  => 2^(32-24) = 256 addresses
```

**Formula:** addresses = 2^(32 − prefix). **Each step smaller in prefix doubles the size.** A **longer prefix number means a smaller network** (`/28` is much smaller than `/16`).

| CIDR | Subnet mask | Total addresses | Usable in an Azure subnet (total − 5) | Typical use |
| --- | --- | --- | --- | --- |
| `/16` | 255.255.0.0 | 65,536 | 65,531 | Whole VNet |
| `/20` | 255.255.240.0 | 4,096 | 4,091 | Large AKS subnet |
| `/22` | 255.255.252.0 | 1,024 | 1,019 | Big app tier |
| `/23` | 255.255.254.0 | 512 | 507 | Medium tier |
| `/24` | 255.255.255.0 | 256 | 251 | Standard subnet |
| `/25` | 255.255.255.128 | 128 | 123 | Small tier |
| `/26` | 255.255.255.192 | 64 | 59 | Azure Bastion / Firewall subnet minimum |
| `/27` | 255.255.255.224 | 32 | 27 | Gateway subnet (recommended minimum) |
| `/28` | 255.255.255.240 | 16 | 11 | Tiny tier |
| `/29` | 255.255.255.248 | 8 | 3 | Smallest allowed subnet |

Azure subnets can be as small as **/29** and as large as **/2**.

### 3.3 Practice (interviewers ask these)

| Question | Answer |
| --- | --- |
| Usable addresses in an Azure `/27`? | 32 − 5 = **27** |
| How many `/24` subnets fit in a `/16`? | 2^(24−16) = **256** |
| Split `10.0.0.0/16` into 4 equal subnets | `/18`: `10.0.0.0/18`, `10.0.64.0/18`, `10.0.128.0/18`, `10.0.192.0/18` |
| Do `10.0.0.0/16` and `10.0.5.0/24` overlap? | **Yes**, the `/24` is inside the `/16` |
| Do `10.1.0.0/16` and `10.2.0.0/16` overlap? | **No** (safe to peer) |
| Need 500 hosts in one subnet. Which prefix? | Need at least 505 addresses, so **/23** (507 usable) |
| Range of `172.16.0.0/12`? | `172.16.0.0` to `172.31.255.255` |

```bash
# Verify with Python (installed on most machines)
python3 - <<'EOF'
import ipaddress
n = ipaddress.ip_network('10.0.0.0/16')
print(n.num_addresses)                              # 65536
print(list(n.subnets(new_prefix=18)))               # four /18 blocks
print(n.overlaps(ipaddress.ip_network('10.0.5.0/24')))   # True
EOF
```

### 3.4 Planning address space like a professional

1. **Never overlap** with your on-premises network or any VNet you may ever peer or connect. Overlap blocks peering and VPN, and fixing it later means rebuilding. Keep a central **IP address plan** (spreadsheet or IPAM).
2. Give each environment its own block: for example prod `10.1.0.0/16`, test `10.2.0.0/16`, dev `10.3.0.0/16`, hub `10.0.0.0/16`.
3. **Don’t make it too small.** Growth is easy to run out of, especially with AKS (pods can consume huge numbers of IPs).
4. **Don’t waste it either.** Giving every VNet a `/8` blocks everyone else. A `/16` per VNet is a common compromise; smaller for tiny workloads.
5. Leave **unallocated space** inside the VNet for future subnets.
6. Use RFC 1918 ranges. Many companies reserve `10.0.0.0/8` slices per team or region.

## 4. Subnets and IP addressing

### 4.1 What is a subnet?

A **subnet** is a smaller, segmented part of the VNet’s address space that groups and isolates resources. Your slide splits a 100-address VNet into public (50) and private (50); in the demo, `azure-demo-vnet 10.0.0.0/16` holds `10.0.0.0/24` and `10.0.1.0/24`. Azure subnets are **regional**: one subnet spans all Availability Zones (unlike AWS, where a subnet lives in one AZ).

**Why create subnets**

- **Security boundary:** apply a different NSG and route table to each tier.
- **Organisation:** web, app, data, management, gateways.
- **Special services:** Bastion, Firewall, VPN Gateway need dedicated subnets.
- **Scaling and cost control:** separate address ranges per team or workload.

**What happens when you create a subnet (from your slide):** you give a CIDR block that falls **inside** the VNet’s address space and **does not overlap** another subnet. That block becomes the pool of private IPs for resources in the subnet.

### 4.2 The five reserved addresses

In every subnet, Azure reserves five addresses. Example for `10.0.1.0/24`:

| Address | Reserved for |
| --- | --- |
| `10.0.1.0` | Network address |
| `10.0.1.1` | Default gateway |
| `10.0.1.2` and `10.0.1.3` | Azure DNS mapping |
| `10.0.1.255` | Broadcast address |

So the **first usable address is `10.0.1.4`** (your slide’s frontend VM) and the last is `10.0.1.254`. Usable = 2^(32−prefix) − 5.

### 4.3 Private IP assignment

- **Dynamic (default):** Azure DHCP gives the next free address. It stays the same while the VM exists, but is not guaranteed forever.
- **Static:** you reserve a specific private IP on the NIC (use for DNS servers, domain controllers, NVAs, anything others hard-code).
- Never configure a static IP *inside* the OS. Set it on the NIC in Azure.

### 4.4 Special subnets (names and sizes matter)

| Purpose | Required name | Minimum size |
| --- | --- | --- |
| VPN / ExpressRoute gateway | `GatewaySubnet` | `/27` recommended |
| Azure Bastion (dedicated SKUs) | `AzureBastionSubnet` | `/26` or larger |
| Azure Firewall | `AzureFirewallSubnet` | `/26` |
| Azure Firewall management (forced tunneling) | `AzureFirewallManagementSubnet` | `/26` |
| Application Gateway v2 | Any dedicated subnet | `/24` recommended |
| PaaS delegation (App Service integration, Container Apps, SQL MI) | Any name; subnet **delegated** to the service | Depends on service |

### 4.5 Subnet design for the slide’s application

| Subnet | CIDR | Contains | Rules |
| --- | --- | --- | --- |
| `snet-frontend` | `10.0.1.0/24` | Web VMs | Allow 80 and 443 from internet; allow SSH only from Bastion |
| `snet-backend` | `10.0.2.0/24` | App VMs | Allow 8080 only from `10.0.1.0/24`; no internet inbound |
| `snet-data` | `10.0.3.0/24` | Databases | Allow DB port only from `snet-backend` |
| `AzureBastionSubnet` | `10.0.254.0/26` | Bastion | Managed |

### 4.6 ‘Public subnet’ and ‘private subnet’ in Azure, AWS and GCP

|  | Azure | AWS | GCP |
| --- | --- | --- | --- |
| What makes it ‘public’ | VMs have a public IP and NSG allows inbound. No special subnet type | Route table has a route to an **Internet Gateway** | VM has an external IP and a firewall rule allows traffic |
| What makes it ‘private’ | No public IPs; for new VNets (after 31 March 2026) outbound also needs NAT Gateway/firewall/LB rules | No route to IGW; outbound via NAT Gateway | No external IP; outbound via Cloud NAT |
| Subnet scope | Regional (all zones) | **One AZ** | Regional |
| Network scope | Regional | Regional | **Global VPC** with regional subnets |

```bash
# Create a second subnet; make it a private subnet (no default outbound access)
az network vnet subnet create -g rg-net-lab --vnet-name vnet-lab -n snet-backend \
  --address-prefixes 10.0.2.0/24 --default-outbound false

az network vnet subnet list -g rg-net-lab --vnet-name vnet-lab -o table
az network vnet subnet show -g rg-net-lab --vnet-name vnet-lab -n snet-backend \
  --query '{cidr:addressPrefix, nsg:networkSecurityGroup.id, routeTable:routeTable.id, natGw:natGateway.id}'
```

(Confirm flag names for your CLI version with `az network vnet subnet create -h`, since networking options evolve.)

### 4.7 Reaching Azure PaaS services privately

|  | Service Endpoint | Private Endpoint |
| --- | --- | --- |
| How | Extends your subnet’s identity to the service over the Azure backbone | Gives the service a **private IP in your subnet** through Private Link |
| Service public endpoint | Still exists (you restrict it to your subnet) | Can be **disabled completely** |
| DNS | No change | Needs a **Private DNS zone** (for example `privatelink.blob.core.windows.net`) |
| Cost | Free | Hourly plus per GB |
| Reach from on-premises | No | Yes (via VPN/ExpressRoute) |
| AWS equivalent | Gateway endpoint (S3, DynamoDB) | Interface endpoint (PrivateLink) |
| GCP equivalent | Private Google Access | Private Service Connect |

Banks and regulated companies standardise on **Private Endpoints** for Storage, SQL, Key Vault and others.

## 5. Deploying a VM inside a VNet

In Part 2 `az vm create` quietly created a VNet for you. In real projects you **create the network first, then place VMs into chosen subnets**.

```bash
RG=rg-net-lab
LOC=centralindia
az group create -n $RG -l $LOC

# Network first
az network vnet create -g $RG -n vnet-lab --address-prefixes 10.0.0.0/16 \
  --subnet-name snet-frontend --subnet-prefixes 10.0.1.0/24
az network vnet subnet create -g $RG --vnet-name vnet-lab -n snet-backend --address-prefixes 10.0.2.0/24

# VM into an existing subnet, with a fixed private IP and no public IP
az vm create -g $RG -n vm-backend --image Ubuntu2204 --size Standard_B1s \
  --vnet-name vnet-lab --subnet snet-backend \
  --private-ip-address 10.0.2.4 --public-ip-address '' \
  --nsg '' --admin-username azureuser --generate-ssh-keys
```

- `--vnet-name` and `--subnet` place the VM in your subnet. `--public-ip-address ''` creates **no** public IP. `--nsg ''` skips the per-NIC NSG so you control security with a subnet-level NSG.
- The first VM in a subnet usually receives `.4` (as in your slide: frontend `10.0.1.4`, backend `10.0.2.4`).
- **Same VNet, different subnets: VMs reach each other automatically** over private IPs; no routes or peering are needed.
- Test with `ping 10.0.2.4` (Linux) or `curl http://10.0.2.4:8080`. Windows blocks ICMP in its own firewall by default, so ping can fail even when the Azure network is fine.
- `az vm create` into a VNet with no public IP and no outbound method means the VM **cannot download packages** (see NAT Gateway, section 8).

```bash
az vm list -g $RG -d --query '[].{name:name, privateIp:privateIps, publicIp:publicIps}' -o table
az network vnet subnet list-available-delegations -l $LOC -o table   # services a subnet can be delegated to
az network vnet list-endpoint-policies -g $RG 2>/dev/null           # (rarely used)
```

## 6. NIC: connecting a VM to the network

### 6.1 What it is

A **network interface (NIC)** is the virtual network card that connects a VM to a subnet and carries its IP configuration. Your slide shows `MyVM-nic` with private IP `10.0.1.4` and an optional public IP.

**A VM has no IP address by itself. The NIC does.** Every VM has at least one NIC, and the NIC is a **separate Azure resource**.

### 6.2 What a NIC holds

| Item | Detail |
| --- | --- |
| **IP configuration(s)** | One primary plus optional secondary private IPs, each can link to a public IP |
| **Subnet** | Which subnet it lives in. A NIC belongs to exactly one VNet |
| **NSG association** | Optional NIC-level firewall (alongside subnet NSG) |
| **MAC address** | Assigned by Azure, stable for the NIC |
| **Accelerated networking** | SR-IOV for lower latency and higher throughput on supported sizes (recommended on) |
| **IP forwarding** | Lets the NIC forward traffic not addressed to itself. Needed for **NVAs** (firewalls, routers) |
| **DNS servers** | Inherited from VNet unless overridden |
| **Application Security Groups** | Role-based grouping used in NSG rules |

### 6.3 Facts for interviews

- The **maximum number of NICs depends on VM size**. Bigger sizes support more.
- **Multiple NICs** are used for NVAs (inside/outside), separating management from data traffic, or connecting to two subnets.
- A NIC can be **detached and attached** to another VM in the same region (the VM must be deallocated). Because the IP is on the NIC, the IP follows.
- Deleting a VM does not delete its NIC by default (a cost and clutter source).
- Bandwidth is determined by **VM size**, not by the NIC.
- Equivalent: AWS **ENI** (Elastic Network Interface); GCP **network interface** attached to an instance (multi-NIC supported, each in a different VPC).

```bash
az network nic create -g $RG -n nic-app01 --vnet-name vnet-lab --subnet snet-backend \
  --private-ip-address 10.0.2.10 --accelerated-networking true
az network nic list -g $RG -o table
az network nic show -g $RG -n nic-app01 --query 'ipConfigurations[].{ip:privateIPAddress,alloc:privateIPAllocationMethod,subnet:subnet.id}'

# Add a secondary IP, enable IP forwarding (for a firewall/NVA)
az network nic ip-config create -g $RG --nic-name nic-app01 -n ipconfig2 --private-ip-address 10.0.2.11
az network nic update -g $RG -n nic-app01 --ip-forwarding true

# Diagnose what the NIC is actually subject to
az network nic list-effective-nsg -g $RG -n nic-app01
az network nic show-effective-route-table -g $RG -n nic-app01 -o table
```

## 7. NSG: controlling traffic inside the VNet

The slide’s definition: an **NSG is a set of network security rules that control inbound and outbound traffic for Azure resources.** Full theory (priority, defaults, stateful, service tags, ASGs, subnet vs NIC, ports) is in **Part 2, section 3**. Here is how it is applied to this VNet.

**Frontend NSG (from your architecture slide)**

| Priority | Name | Direction | Source | Port | Action |
| --- | --- | --- | --- | --- | --- |
| 100 | Allow-HTTP | Inbound | Internet | 80 | Allow |
| 110 | Allow-HTTPS | Inbound | Internet | 443 | Allow |
| 4096 | Deny-All-Inbound | Inbound | Any | Any | Deny |

The explicit deny is already covered by default rule 65500, but many teams add a visible low-priority deny (for logging and clarity). **Backend NSG:** allow 8080 only from the frontend subnet `10.0.1.0/24` (your slide uses the single VM `10.0.1.4`; using the subnet or an ASG survives scale-out). Everything else from the internet is denied by default.

```bash
az network nsg create -g $RG -n nsg-frontend
az network nsg rule create -g $RG --nsg-name nsg-frontend -n Allow-HTTP --priority 100 \
  --direction Inbound --access Allow --protocol Tcp --source-address-prefixes Internet --destination-port-ranges 80
az network nsg rule create -g $RG --nsg-name nsg-frontend -n Allow-HTTPS --priority 110 \
  --direction Inbound --access Allow --protocol Tcp --source-address-prefixes Internet --destination-port-ranges 443
az network nsg rule create -g $RG --nsg-name nsg-frontend -n Deny-All-Inbound --priority 4096 \
  --direction Inbound --access Deny --protocol '*' --source-address-prefixes '*' --destination-port-ranges '*'

az network nsg create -g $RG -n nsg-backend
az network nsg rule create -g $RG --nsg-name nsg-backend -n Allow-8080-From-Frontend --priority 100 \
  --direction Inbound --access Allow --protocol Tcp --source-address-prefixes 10.0.1.0/24 --destination-port-ranges 8080

az network vnet subnet update -g $RG --vnet-name vnet-lab -n snet-frontend --network-security-group nsg-frontend
az network vnet subnet update -g $RG --vnet-name vnet-lab -n snet-backend  --network-security-group nsg-backend
```

## 8. NAT Gateway: outbound internet for private resources

### 8.1 What NAT is

**NAT (Network Address Translation)** lets resources with **private IPs** reach the internet using a **public IP**. The NAT device rewrites the source address on the way out and maps the replies back. **Azure NAT Gateway** is the managed service for this.

- It is **outbound only**. The internet cannot initiate connections to the VMs through it.
- It attaches at the **subnet** level; every VM in that subnet uses it automatically.
- It uses **one or more Standard public IPs** (or a public IP prefix). All VMs share those IPs, so you get a **stable, known outbound IP** you can give to partners for allow-listing.
- Provides **SNAT port capacity**: each public IP provides about 64,000 SNAT ports, and up to 16 IPs can be attached. Many parallel outbound connections to the same destination need enough ports, otherwise connections fail (**SNAT port exhaustion**).
- Configurable **idle timeout** (4 to 120 minutes).
- **Takes precedence** for outbound traffic from its subnet over other outbound methods.
- Billing: hourly for the gateway plus per GB processed, plus the public IP.
- Newer NAT Gateway SKUs add zone-redundancy. Check the current documentation for which SKU to use in your region.

### 8.2 Why it matters now

New VNets are private by default. A backend VM with no public IP can no longer `apt update`, pull Docker images, call external APIs or reach licensing servers unless **you give it an outbound path**.

| Outbound method | Inbound possible? | Stable outbound IP | Best for |
| --- | --- | --- | --- |
| Public IP on the VM | Yes (if NSG allows) | Yes, per VM | One internet-facing server |
| **NAT Gateway** | No | **Yes, shared** | Private tiers needing updates/APIs |
| Load balancer outbound rules | Via LB rules | Yes | VMs already behind a Standard LB |
| Azure Firewall / NVA + UDR | Controlled | Yes | Central inspection, FQDN filtering |
| Default outbound access (legacy) | No | **No** (Azure-owned, can change) | Avoid; retired for new VNets |

AWS: **NAT Gateway** (in a public subnet, with a route from private subnets). GCP: **Cloud NAT** (attached to a Cloud Router).

```bash
az network public-ip create -g $RG -n pip-nat --sku Standard --allocation-method Static
az network nat gateway create -g $RG -n natgw-lab --public-ip-addresses pip-nat --idle-timeout 10
az network vnet subnet update -g $RG --vnet-name vnet-lab -n snet-backend --nat-gateway natgw-lab

# Prove it: from the backend VM (no public IP) the outside world sees the NAT IP
ssh azureuser@10.0.2.4          # via Bastion or from the frontend VM
curl -s https://ifconfig.me ; echo
az network public-ip show -g $RG -n pip-nat --query ipAddress -o tsv   # same address
```

**Order of operations (a real gotcha):** attach the NAT Gateway to the subnet **before** creating a VM whose custom data installs packages. Otherwise first-boot `apt install` fails because there is no internet path.

## 9. Route tables

### 9.1 What routing does

A **route table** defines **where traffic from a subnet goes next**. Each route has an **address prefix** and a **next hop**. Azure creates working routes for you automatically (**system routes**). You override them with **user-defined routes (UDRs)** in a route table associated with a subnet.

### 9.2 System routes (exist in every subnet)

| Prefix | Next hop | Meaning |
| --- | --- | --- |
| VNet address space (for example `10.0.0.0/16`) | `VnetLocal` | Traffic inside the VNet |
| `0.0.0.0/0` | `Internet` | Everything else goes to the internet |
| `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` (not part of your VNet), `100.64.0.0/10` | `None` | Dropped, to avoid leaking private ranges |

Other routes are added automatically: **VNet peering** routes for the peer’s address space, **VPN/ExpressRoute** routes learned through BGP, and Service Endpoint routes.

### 9.3 Next hop types

| Next hop type | Use |
| --- | --- |
| `VnetLocal` | Stay inside the VNet |
| `Internet` | Go to the public internet |
| `VirtualNetworkGateway` | Send to a VPN/ExpressRoute gateway (on-premises) |
| `VirtualAppliance` | Send to an NVA or Azure Firewall at a private IP |
| `None` | Drop the traffic (blackhole) |

### 9.4 How Azure picks a route

1. **Longest prefix match** wins (`10.0.3.0/24` beats `10.0.0.0/16` beats `0.0.0.0/0`).
2. If prefixes are equal: **UDR beats BGP route beats system route**.

### 9.5 Your slide’s example table

| Destination | Next hop type | Next hop |
| --- | --- | --- |
| `10.0.3.0/24` | Virtual appliance | `10.0.3.4` |
| `0.0.0.0/0` | Virtual appliance | `10.0.3.4` |
| `10.1.0.0/16` | Virtual network (`VnetLocal`) | n/a |

The `0.0.0.0/0 → virtual appliance` route is **forced tunneling / forced inspection**: all internet-bound traffic from the backend subnet goes to **Azure Firewall** (or an NVA) at `10.0.3.4` first. This is the standard enterprise pattern. For the NVA to forward traffic, enable **IP forwarding** on its NIC and in its operating system.

**Common uses of UDRs:** route internet traffic via Azure Firewall; send spoke-to-spoke traffic through the hub; force on-premises traffic through an NVA; blackhole a prefix (`None`); override BGP routes.

```bash
az network route-table create -g $RG -n rt-backend --disable-bgp-route-propagation true
az network route-table route create -g $RG --route-table-name rt-backend -n to-internet-via-fw \
  --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address 10.0.3.4
az network route-table route create -g $RG --route-table-name rt-backend -n block-bad-range \
  --address-prefix 203.0.113.0/24 --next-hop-type None
az network vnet subnet update -g $RG --vnet-name vnet-lab -n snet-backend --route-table rt-backend

# Troubleshooting
az network nic show-effective-route-table -g $RG -n <nic-name> -o table
az network watcher show-next-hop -g $RG --vm vm-backend --source-ip 10.0.2.4 --dest-ip 8.8.8.8
```

Notes: associating a route table with a subnet that has a **NAT Gateway** for the same destination can conflict, since the UDR to a firewall changes the outbound path. Also watch for **asymmetric routing** (request goes via the firewall, reply returns directly), which breaks stateful firewalls.

|  | Azure | AWS | GCP |
| --- | --- | --- | --- |
| Route object | Route table (associated to subnets) | Route table (associated to subnets) | Routes (VPC-wide, matched by network tags) |
| Internet path | System route `0.0.0.0/0 → Internet` | Route to **Internet Gateway** | Default route to the default internet gateway |
| Send via firewall | UDR → `VirtualAppliance` | Route to firewall endpoint / ENI | Route with next hop an instance or ILB |
| Hub transit | Peering + UDR / Virtual WAN | **Transit Gateway** | Network Connectivity Center |

## 10. VNet peering

### 10.1 What it is

**VNet peering** connects two Azure VNets so resources in each talk to each other **privately, using private IPs, over the Microsoft backbone network**. Traffic never touches the public internet, needs **no VPN gateway or extra device**, and has the latency and bandwidth of traffic inside one VNet. Equivalent: AWS **VPC peering**, GCP **VPC Network Peering**.

| Type | Meaning |
| --- | --- |
| **Regional peering** | Both VNets in the same region |
| **Global peering** | VNets in different regions (your slide: East US to West Europe) |

Peering also works **across subscriptions and across Microsoft Entra tenants** (with the right permissions on both sides).

### 10.2 Rules and behaviour (exam and interview favourites)

- **Address spaces must not overlap.** This is the number one reason peering fails.
- Peering is **non-transitive**. A↔B and B↔C does **not** give A↔C. (Your slide says ‘supports transitive peering’, which is wrong.)
- It must be created **in both directions** (two links). Until both exist the state is `Initiated`; when both exist it becomes `Connected`.
- NSGs still apply. The default `AllowVnetInBound` rule treats **peered VNets as part of the ‘VirtualNetwork’ tag**, so traffic is allowed unless you restrict it.
- Peered VNets are **not** merged: each keeps its own subnets, NSGs and route tables.
- Data transfer over peering is **charged per GB** in both directions (lower inside a region, higher across regions).
- You cannot peer VNets whose address spaces overlap, and you must sync the peering when you change an address space.
- Name resolution does not automatically work across peered VNets. Link a **Private DNS zone** to both, or use your own DNS.

### 10.3 Peering settings

| Setting | Purpose |
| --- | --- |
| **Allow virtual network access** | Let resources in the two VNets communicate (on by default) |
| **Allow forwarded traffic** | Accept traffic that did not originate in the peer VNet (needed when an NVA/firewall in one VNet forwards traffic) |
| **Allow gateway transit** | Set on the **hub**: lets peers use this VNet’s VPN/ExpressRoute gateway |
| **Use remote gateways** | Set on the **spoke**: use the hub’s gateway to reach on-premises (only if the hub allows gateway transit) |

### 10.4 Peering vs VPN Gateway (VNet-to-VNet)

|  | VNet peering | VPN Gateway |
| --- | --- | --- |
| Path | Microsoft backbone | Encrypted tunnel (still over the backbone for Azure-to-Azure) |
| Latency / bandwidth | Lowest / highest | Limited by gateway SKU |
| Gateway needed | **No** | Yes, in each VNet |
| Transitive | No (needs hub design) | Can be configured with BGP |
| Cost | Per GB transfer | Gateway hourly + transfer |
| Use | Almost all Azure-to-Azure connectivity | Encryption requirement, on-premises, overlapping IP workarounds |

### 10.5 Hub-and-spoke and transit patterns

A **hub** VNet holds shared services (Azure Firewall, VPN/ExpressRoute gateway, Bastion, DNS, monitoring). **Spoke** VNets (prod, test, per team) each peer to the hub.

| Need | Solution |
| --- | --- |
| Spokes reach on-premises through one gateway | **Gateway transit** (hub: allow gateway transit; spokes: use remote gateways) |
| Spoke to spoke through the hub | Peer each spoke to the hub, put **Azure Firewall/NVA** in the hub, add **UDRs** on spokes (`0.0.0.0/0` or the other spoke’s prefix to the firewall IP) and enable forwarded traffic |
| Spoke to spoke directly (best performance) | Peer the two spokes directly |
| Many VNets, many regions, branch offices | **Azure Virtual WAN** |
| Manage large peering meshes centrally | **Azure Virtual Network Manager** (connectivity configurations) |

AWS equivalents: **Transit Gateway** for hub-and-spoke. GCP: **Network Connectivity Center**, or **Shared VPC** to share one network across projects.

```bash
VNET1=$(az network vnet show -g $RG -n vnet-lab  --query id -o tsv)
VNET2=$(az network vnet show -g $RG -n vnet-lab2 --query id -o tsv)

# Two directions are required
az network vnet peering create -g $RG -n lab-to-lab2 --vnet-name vnet-lab  --remote-vnet $VNET2 \
  --allow-vnet-access --allow-forwarded-traffic
az network vnet peering create -g $RG -n lab2-to-lab --vnet-name vnet-lab2 --remote-vnet $VNET1 \
  --allow-vnet-access --allow-forwarded-traffic

az network vnet peering list -g $RG --vnet-name vnet-lab -o table
az network vnet peering show -g $RG --vnet-name vnet-lab -n lab-to-lab2 --query peeringState -o tsv   # Connected

# Hub and spoke with a shared gateway
az network vnet peering create -g $RG -n hub-to-spoke --vnet-name vnet-hub --remote-vnet <spoke-id> \
  --allow-vnet-access --allow-gateway-transit
az network vnet peering create -g $RG -n spoke-to-hub --vnet-name vnet-spoke --remote-vnet <hub-id> \
  --allow-vnet-access --use-remote-gateways
```

## 11. Bastion host (jump host)

### 11.1 The concept

A **bastion host (jump host)** is a hardened server that is the **single controlled entry point** to private servers. Instead of giving every VM a public IP and opening SSH/RDP to the world, admins connect to the bastion and from there reach private VMs. It shrinks the attack surface to one monitored door.

|  | Self-managed jump VM | **Azure Bastion** |
| --- | --- | --- |
| What it is | A VM you build, patch and secure | Managed PaaS service in your VNet |
| Public exposure | Public IP with SSH/RDP open (risk) | Public IP on Bastion only; TLS 443; VMs stay private |
| Maintenance | You patch, harden, monitor | Microsoft |
| Access method | `ssh -J` / RDP hop | Browser (portal) or native client |
| Cost | VM + disks | Hourly while deployed (plus data) |

**How Azure Bastion works:** you open the Azure portal (HTTPS 443) or use the CLI; Bastion in `AzureBastionSubnet` opens an SSH/RDP session to the VM’s **private IP** inside the VNet. The VM needs **no public IP** and no inbound 22/3389 from the internet. It works with VMs in **peered VNets** (Standard SKU and above).

| SKU | Notes |
| --- | --- |
| **Developer** | Free, shared infrastructure, one VM connection at a time, limited regions; dev/test |
| **Basic** | Dedicated, browser-based RDP/SSH |
| **Standard** | Adds **native client** (`az network bastion ssh/rdp`), shareable links, file transfer, scaling |
| **Premium** | Adds **session recording** and private-only deployment |

SKUs can be upgraded but not downgraded. Dedicated SKUs need `AzureBastionSubnet` (`/26` or larger) and a Standard static public IP (Premium can be private-only).

```bash
az network vnet subnet create -g $RG --vnet-name vnet-lab -n AzureBastionSubnet --address-prefixes 10.0.254.0/26
az network public-ip create -g $RG -n pip-bastion --sku Standard --allocation-method Static
az network bastion create -g $RG -n bas-lab --vnet-name vnet-lab --public-ip-address pip-bastion \
  --sku Standard --enable-tunneling true

# Native-client SSH (Standard or Premium)
az network bastion ssh -n bas-lab -g $RG \
  --target-resource-id $(az vm show -g $RG -n vm-backend --query id -o tsv) \
  --auth-type ssh-key --username azureuser --ssh-key ~/.ssh/id_rsa

az network bastion delete -g $RG -n bas-lab        # billed hourly while it exists: delete after the lab
```

**Self-managed jump host with SSH ProxyJump** (free, fine for labs):

```bash
ssh -J azureuser@<frontend-public-ip> azureuser@10.0.2.4

# ~/.ssh/config
Host backend
  HostName 10.0.2.4
  User azureuser
  ProxyJump azureuser@<frontend-public-ip>
```

Equivalents: AWS **SSM Session Manager** or **EC2 Instance Connect Endpoint** (no bastion VM needed); GCP **Identity-Aware Proxy (IAP) TCP forwarding**.

## 12. Hands-on: build and test the VNet architecture

**Goal:** reproduce your slide’s `my-webapp`: a public **frontend VM** (`10.0.1.4`) and a private **backend VM** (`10.0.2.4`) in one VNet, then add NAT Gateway, peering and diagnostics.

```
Internet --> [public IP] --> NSG --> snet-frontend 10.0.1.0/24: vm-frontend 10.0.1.4
                                          |  allow 8080 (private IP only)
                                          v
                              snet-backend 10.0.2.0/24: vm-backend 10.0.2.4 (no public IP, NAT Gateway for outbound)
```

### Step 1: variables and resource group

```bash
RG=rg-net-lab ; LOC=centralindia
MYIP=$(curl -s https://ifconfig.me)
az group create -n $RG -l $LOC --tags env=lab
```

### Step 2: NSGs

```bash
az network nsg create -g $RG -n nsg-frontend
az network nsg rule create -g $RG --nsg-name nsg-frontend -n Allow-HTTP  --priority 100 --direction Inbound \
  --access Allow --protocol Tcp --source-address-prefixes Internet --destination-port-ranges 80
az network nsg rule create -g $RG --nsg-name nsg-frontend -n Allow-HTTPS --priority 110 --direction Inbound \
  --access Allow --protocol Tcp --source-address-prefixes Internet --destination-port-ranges 443
az network nsg rule create -g $RG --nsg-name nsg-frontend -n Allow-SSH-MyIP --priority 120 --direction Inbound \
  --access Allow --protocol Tcp --source-address-prefixes $MYIP/32 --destination-port-ranges 22

az network nsg create -g $RG -n nsg-backend
az network nsg rule create -g $RG --nsg-name nsg-backend -n Allow-8080-From-Frontend --priority 100 --direction Inbound \
  --access Allow --protocol Tcp --source-address-prefixes 10.0.1.0/24 --destination-port-ranges 8080
az network nsg rule create -g $RG --nsg-name nsg-backend -n Allow-SSH-From-Frontend --priority 110 --direction Inbound \
  --access Allow --protocol Tcp --source-address-prefixes 10.0.1.0/24 --destination-port-ranges 22
```

### Step 3: VNet, subnets, NAT Gateway (NAT first!)

```bash
az network vnet create -g $RG -n vnet-lab --address-prefixes 10.0.0.0/16 \
  --subnet-name snet-frontend --subnet-prefixes 10.0.1.0/24 --network-security-group nsg-frontend

az network public-ip create -g $RG -n pip-nat --sku Standard --allocation-method Static
az network nat gateway create -g $RG -n natgw-lab --public-ip-addresses pip-nat

az network vnet subnet create -g $RG --vnet-name vnet-lab -n snet-backend --address-prefixes 10.0.2.0/24 \
  --network-security-group nsg-backend --nat-gateway natgw-lab
```

### Step 4: VMs

Frontend bootstrap (`frontend.sh`):

```bash
#!/bin/bash
apt-get update -y
apt-get install -y apache2
echo '<h1>Frontend VM 10.0.1.4: my-webapp</h1>' > /var/www/html/index.html
```

Backend bootstrap (`backend.yaml`, needs no internet):

```yaml
#cloud-config
write_files:
  - path: /opt/app/index.html
    content: 'Hello from the BACKEND app server 10.0.2.4'
  - path: /etc/systemd/system/backend.service
    content: |
      [Unit]
      Description=Demo backend
      After=network.target
      [Service]
      WorkingDirectory=/opt/app
      ExecStart=/usr/bin/python3 -m http.server 8080
      Restart=always
      [Install]
      WantedBy=multi-user.target
runcmd:
  - systemctl daemon-reload
  - systemctl enable --now backend.service
```

```bash
az vm create -g $RG -n vm-frontend --image Ubuntu2204 --size Standard_B1s \
  --vnet-name vnet-lab --subnet snet-frontend --private-ip-address 10.0.1.4 \
  --public-ip-sku Standard --nsg '' --custom-data frontend.sh \
  --admin-username azureuser --generate-ssh-keys

az vm create -g $RG -n vm-backend --image Ubuntu2204 --size Standard_B1s \
  --vnet-name vnet-lab --subnet snet-backend --private-ip-address 10.0.2.4 \
  --public-ip-address '' --nsg '' --custom-data backend.yaml \
  --admin-username azureuser --generate-ssh-keys
```

### Step 5: test every statement on your slides

```bash
FIP=$(az vm show -g $RG -n vm-frontend -d --query publicIps -o tsv)

curl http://$FIP                                   # 1. Users reach the frontend      -> works
curl --max-time 5 http://10.0.2.4:8080             # 2. Internet/admin to backend      -> times out (private IP, red cross on slide)

ssh azureuser@$FIP                                 # log in to the frontend
  curl http://10.0.2.4:8080                        # 3. Frontend to backend over private IP -> 'Hello from the BACKEND...'
  ping -c 3 10.0.2.4                               #    VNet internal traffic is allowed by default
  nc -zv 10.0.2.4 8080                             #    port test
  exit

ssh -J azureuser@$FIP azureuser@10.0.2.4           # 4. Reach the private backend through the jump host
  curl -s https://ifconfig.me ; echo               # 5. Outbound works via NAT: prints the NAT Gateway public IP
  exit
az network public-ip show -g $RG -n pip-nat --query ipAddress -o tsv   # compare with step 5
```

Break things on purpose (best way to learn): delete the `Allow-8080-From-Frontend` rule and watch step 3 fail; re-add it and it works.

### Step 6: VNet peering lab

```bash
az network vnet create -g $RG -n vnet-lab2 --address-prefixes 10.1.0.0/16 --subnet-name snet-peer --subnet-prefixes 10.1.1.0/24
az vm create -g $RG -n vm-peer --image Ubuntu2204 --size Standard_B1s --vnet-name vnet-lab2 --subnet snet-peer \
  --private-ip-address 10.1.1.4 --public-ip-address '' --nsg '' --admin-username azureuser --generate-ssh-keys

# From vm-frontend: ping 10.1.1.4  -> fails (different, unconnected VNets)
# Create both peering links (section 10), then:
# From vm-frontend: ping 10.1.1.4  -> works over the Microsoft backbone
```

Try the transitivity test: peer `vnet-lab2` with a third VNet `vnet-lab3` and confirm `vnet-lab` still cannot reach `vnet-lab3`.

### Step 7: route table lab

```bash
az network route-table create -g $RG -n rt-backend
az network route-table route create -g $RG --route-table-name rt-backend -n block-peer \
  --address-prefix 10.1.0.0/16 --next-hop-type None
az network vnet subnet update -g $RG --vnet-name vnet-lab -n snet-backend --route-table rt-backend
# From vm-backend: ping 10.1.1.4 now fails because the UDR blackholes it. Remove the route to restore.
az network nic show-effective-route-table -g $RG -n vm-backendVMNic -o table
```

### Step 8: diagnose with Network Watcher

```bash
# Would NSGs allow this packet? Shows the rule that decides.
az network watcher test-ip-flow -g $RG --vm vm-backend --direction Inbound --protocol TCP \
  --local 10.0.2.4:8080 --remote 10.0.1.4:40000

# Where would the traffic go next?
az network watcher show-next-hop -g $RG --vm vm-backend --source-ip 10.0.2.4 --dest-ip 8.8.8.8

# End-to-end connectivity test (needs the Network Watcher agent extension on the source VM)
az vm extension set -g $RG --vm-name vm-frontend --name NetworkWatcherAgentLinux --publisher Microsoft.Azure.NetworkWatcher
az network watcher test-connectivity -g $RG --source-resource vm-frontend --dest-address 10.0.2.4 --dest-port 8080
```

### Step 9: clean up

```bash
az network bastion delete -g $RG -n bas-lab 2>/dev/null
az group delete -n $RG --yes --no-wait
```

## 13. Real-world project: Tomato network design

### 13.1 Production network with hub-and-spoke

Tomato runs prod and dev, has an office (and a legacy data centre) connected by VPN, and must keep database traffic off the internet.

**Address plan (written down first, never overlapping)**

| Network | CIDR | Notes |
| --- | --- | --- |
| On-premises office | `192.168.0.0/16` | Existing, cannot change |
| **Hub** (Central India) | `10.0.0.0/16` | Shared services |
| → `AzureFirewallSubnet` | `10.0.1.0/26` | Azure Firewall |
| → `GatewaySubnet` | `10.0.2.0/27` | VPN / ExpressRoute gateway |
| → `AzureBastionSubnet` | `10.0.3.0/26` | Bastion |
| → `snet-dns` | `10.0.4.0/28` | DNS resolver |
| **Prod spoke** | `10.1.0.0/16` | Customer-facing workloads |
| → `snet-appgw` | `10.1.5.0/24` | Application Gateway + WAF |
| → `snet-web` | `10.1.1.0/24` | Web VM scale set |
| → `snet-app` | `10.1.2.0/24` | App / API tier |
| → `snet-data` | `10.1.3.0/24` | Databases (SQL on VMs) |
| → `snet-pe` | `10.1.4.0/24` | Private Endpoints (Storage, SQL, Key Vault) |
| **Dev spoke** | `10.2.0.0/16` | Same layout, lower cost |
| **DR spoke** (second region) | `10.11.0.0/16` | Replica of prod for disaster recovery |

**Traffic flows**

1. **Customers:** Front Door → Application Gateway (WAF) in `snet-appgw` → web tier → app tier → database. The only public entry is Front Door/App Gateway.
2. **Private PaaS access:** the app reads food images from Blob Storage and secrets from Key Vault through **Private Endpoints** in `snet-pe`. Public access on those services is disabled. **Private DNS zones** are linked to the hub and spokes.
3. **Outbound internet:** every spoke subnet has a UDR `0.0.0.0/0 → Azure Firewall (10.0.1.4)`. The firewall allows only approved FQDNs (OS updates, payment gateway).
4. **Admins:** Azure Bastion in the hub reaches VMs in any peered spoke by private IP. No VM has a public IP; Azure Policy denies public IPs on NICs in spokes.
5. **Office:** site-to-site VPN (or ExpressRoute) terminates in the hub. **Gateway transit** shares it with all spokes.
6. **Isolation:** prod and dev are not peered to each other. NSGs on each subnet follow the tier rules from Part 2 (web → app 8080 → data 1433). VNet flow logs go to Log Analytics.
7. **Operations:** all of it is defined in Bicep/Terraform; peerings are created by the pipeline or by Azure Virtual Network Manager.

**Why hub-and-spoke?** Shared services are paid for and managed once; spokes stay simple and isolated; a single place to inspect and log traffic; new environments just peer to the hub.

**Cost check:** Azure Firewall and VPN gateways bill hourly. For a small startup, use one VNet, subnet-level NSGs, a NAT Gateway and Bastion (Developer SKU for non-prod) first, and grow into hub-and-spoke later.

### 13.2 Case: a merger with overlapping IP ranges

Tomato acquires a company that also uses `10.0.0.0/16`. Peering is impossible while ranges overlap. Options, in order of preference: re-IP the smaller network; expose only specific services through **Private Link / Private Endpoints** (no routing needed); or place **NAT** in between with an NVA or VPN with NAT rules. Lesson: keep a central IP plan from day one.

### 13.3 Case: lift-and-shift of the legacy order system

VMs for web/app/SQL go into `snet-web`, `snet-app`, `snet-data` (Part 2, section 8.1). The office connects over VPN into the hub, so staff reach the app using private IPs and internal DNS names exactly as they did on-premises.

## 14. AWS and GCP equivalents

### 14.1 Concepts

| Concept | Azure | AWS | GCP |
| --- | --- | --- | --- |
| Private network | Virtual Network (VNet), regional | VPC, regional | VPC network, **global** |
| Address space | VNet address space (can have several) | VPC CIDR (primary + secondary) | Per-subnet ranges only (no VPC-wide CIDR) |
| Subnet | Regional, spans zones | **Zonal** (one AZ) | Regional |
| Reserved IPs per subnet | 5 | 5 | 4 |
| Network card | NIC | ENI | NIC |
| Public IP | Standard Public IP | Elastic IP | External IP |
| Firewall | NSG (stateful, subnet and/or NIC) | Security Group (stateful) + NACL (stateless) | VPC firewall rules (stateful, VPC level, tags) |
| Outbound NAT | NAT Gateway | NAT Gateway | Cloud NAT (on Cloud Router) |
| Route table | Route table (UDR) on subnets | Route table on subnets | Routes (VPC-wide) |
| Internet path | Implicit (system route) | Internet Gateway | Implicit default route |
| VNet-to-VNet | VNet peering (non-transitive) | VPC peering (non-transitive) | VPC peering (non-transitive) |
| Hub transit | Virtual WAN / hub firewall / AVNM | Transit Gateway | Network Connectivity Center |
| Share network across accounts | Peering / Virtual WAN | Resource Access Manager, Transit Gateway | **Shared VPC** |
| VPN | VPN Gateway | Site-to-Site VPN | Cloud VPN |
| Dedicated line | ExpressRoute | Direct Connect | Cloud Interconnect |
| Secure admin access | Bastion | SSM Session Manager / EC2 Instance Connect Endpoint | IAP TCP forwarding |
| Private access to PaaS | Private Endpoint / Service Endpoint | Interface endpoint / Gateway endpoint | Private Service Connect / Private Google Access |
| Private DNS | Private DNS zone | Route 53 private hosted zone | Cloud DNS private zone |
| Central firewall | Azure Firewall | AWS Network Firewall | Cloud NGFW |
| Traffic diagnostics | Network Watcher (IP flow verify, next hop, connection troubleshoot) | VPC Reachability Analyzer, Flow Logs | Connectivity Tests (Network Intelligence Center) |

**Three differences to remember:** AWS subnets are **per AZ** while Azure and GCP subnets are regional. A GCP VPC is **global** while Azure and AWS networks are regional. Azure has **no Internet Gateway**: internet access comes from system routes plus a public IP or outbound method.

### 14.2 Commands

| Task | Azure CLI | AWS CLI | gcloud |
| --- | --- | --- | --- |
| Create network | `az network vnet create -g rg -n vnet --address-prefixes 10.0.0.0/16` | `aws ec2 create-vpc --cidr-block 10.0.0.0/16` | `gcloud compute networks create vpc --subnet-mode=custom` |
| Create subnet | `az network vnet subnet create -g rg --vnet-name vnet -n snet --address-prefixes 10.0.1.0/24` | `aws ec2 create-subnet --vpc-id <vpc> --cidr-block 10.0.1.0/24 --availability-zone ap-south-1a` | `gcloud compute networks subnets create snet --network=vpc --range=10.0.1.0/24 --region=asia-south1` |
| Internet access | Public IP on VM (system route exists) | `aws ec2 create-internet-gateway`, `attach-internet-gateway`, `create-route --destination-cidr-block 0.0.0.0/0 --gateway-id <igw>` | External IP on instance (default route exists) |
| Open port 80 | `az network nsg rule create ... --destination-port-ranges 80` | `aws ec2 authorize-security-group-ingress --group-id <sg> --protocol tcp --port 80 --cidr 0.0.0.0/0` | `gcloud compute firewall-rules create allow-http --network=vpc --allow=tcp:80 --target-tags=web` |
| NAT for private subnet | `az network nat gateway create` then `subnet update --nat-gateway` | `aws ec2 allocate-address`, `create-nat-gateway --subnet-id <public-subnet> --allocation-id <eip>`, add route | `gcloud compute routers create` then `gcloud compute routers nats create --auto-allocate-nat-external-ips --nat-all-subnet-ip-ranges` |
| Custom route | `az network route-table route create --next-hop-type VirtualAppliance` | `aws ec2 create-route --route-table-id <rtb> --destination-cidr-block ... --network-interface-id <eni>` | `gcloud compute routes create ... --next-hop-instance=<vm>` |
| Peering | `az network vnet peering create` (both sides) | `aws ec2 create-vpc-peering-connection`, then `accept-vpc-peering-connection`, plus route entries | `gcloud compute networks peerings create` (both sides) |
| Show effective routes/rules | `az network nic show-effective-route-table` | `aws ec2 describe-route-tables` | `gcloud compute routes list` / Connectivity Tests |

## 15. Interview questions and model answers

### Basic

1. **What is an Azure VNet?** A private, isolated, software-defined network in Azure, scoped to a region and subscription, where resources communicate over private IPs and connect to the internet, other VNets and on-premises networks under your control. Equivalent to AWS VPC and GCP VPC.
2. **Why do we need a VNet?** Isolation, segmentation into tiers, private communication, controlled internet exposure, hybrid connectivity, and security controls (NSGs, route tables, firewalls).
3. **What is CIDR and how many addresses are in `/16`, `/24`, `/27`?** `network/prefix`; addresses = 2^(32−prefix): 65,536; 256; 32. Azure reserves 5 per subnet, so 251 usable in a `/24` and 27 in a `/27`.
4. **Which addresses does Azure reserve in a subnet and why is the first VM `.4`?** `.0` network, `.1` gateway, `.2` and `.3` Azure DNS, last address broadcast.
5. **What is a subnet and why create several?** A slice of the VNet address space. For security boundaries (NSG and route table per subnet), tier separation, and special services such as Bastion and Firewall.
6. **Can resources in different subnets of the same VNet talk to each other?** Yes, by default, through the default NSG rule `AllowVnetInBound`. Restrict with NSGs.
7. **Is a VNet regional or global? Does it span Availability Zones?** Regional; it spans all AZs of that region. It can’t span regions.
8. **Does a VNet cost money?** No. You pay for attached services: public IPs, NAT Gateway, gateways, Firewall, Bastion, peering transfer.
9. **What is a NIC?** The virtual network card that connects a VM to a subnet and holds its private and public IP configuration, NSG association and settings such as accelerated networking and IP forwarding. It is a separate resource.
10. **What is NAT and what does Azure NAT Gateway do?** Translates private source IPs to a public IP for outbound traffic. Azure NAT Gateway is a managed subnet-level outbound-only service with a stable shared public IP and large SNAT port capacity.
11. **What is VNet peering?** A private, low-latency connection between two VNets over the Microsoft backbone, regional or global, with no gateway needed.
12. **What is a bastion host?** A hardened jump server acting as the single entry point to private VMs. Azure Bastion is the managed version delivering RDP/SSH over TLS 443 without public IPs on VMs.

### Intermediate

13. **Is VNet peering transitive?** No. A↔B and B↔C does not allow A↔C. Use direct peering, a hub firewall/NVA with UDRs, gateway transit, Virtual WAN or Virtual Network Manager.
14. **Why does peering stay in ‘Initiated’ state?** The reverse link has not been created. Peering needs a link in each direction to become `Connected`.
15. **What stops two VNets from being peered?** Overlapping address spaces.
16. **Explain ‘Allow forwarded traffic’, ‘Allow gateway transit’ and ‘Use remote gateways’.** Forwarded traffic accepts packets not originating in the peer VNet (NVA scenarios). Gateway transit on the hub shares its VPN/ExpressRoute gateway; Use remote gateways on the spoke consumes it.
17. **Hub-and-spoke: why and how?** Centralise shared services (firewall, gateway, Bastion, DNS) in a hub; spokes peer to it; UDRs send traffic through the firewall; gateway transit shares the VPN. Benefits: cost sharing, central inspection, isolation, easy onboarding.
18. **How do system routes and UDRs interact?** Azure picks the **longest prefix match**; if equal, **UDR over BGP over system**. A UDR for `0.0.0.0/0` overrides the system internet route.
19. **What are the next hop types?** `VnetLocal`, `Internet`, `VirtualNetworkGateway`, `VirtualAppliance`, `None`.
20. **How do you force all internet traffic through Azure Firewall?** Route table with `0.0.0.0/0` next hop type `VirtualAppliance` pointing to the firewall’s private IP, associated with each spoke subnet.
21. **What does ‘IP forwarding’ on a NIC do and when is it needed?** Lets the NIC forward packets not addressed to it. Needed on NVAs/routers. Also enable forwarding in the OS.
22. **NAT Gateway vs public IP vs load balancer outbound rules?** NAT Gateway: outbound only, shared stable IPs, subnet-wide, scalable SNAT. Public IP: per-VM, allows inbound. LB outbound rules: for VMs behind a Standard LB. NAT Gateway takes precedence for its subnet.
23. **What is SNAT port exhaustion?** Outbound connections from many VMs to the same destination run out of available source ports on the shared public IP, causing connection failures. Fix with more public IPs on the NAT Gateway, connection reuse, or Private Endpoints.
24. **Service Endpoint vs Private Endpoint?** Service endpoints extend the subnet identity to a service that keeps its public endpoint. Private Endpoints give the service a private IP in your subnet, allow disabling public access, work from on-premises, and need Private DNS.
25. **Special subnets and sizes?** `GatewaySubnet` (/27 recommended), `AzureBastionSubnet` (/26+), `AzureFirewallSubnet` (/26), dedicated subnet for Application Gateway v2.
26. **How do you size a subnet for 500 hosts?** At least 505 addresses, so `/23` (512, 507 usable).
27. **NSG on subnet vs NIC?** Both apply; inbound subnet then NIC, outbound NIC then subnet. Prefer subnet-level. (Full detail in Part 2.)
28. **Compare Azure, AWS and GCP networking scope.** Azure VNet regional with regional subnets; AWS VPC regional with zonal subnets; GCP VPC global with regional subnets. AWS needs an Internet Gateway; Azure uses system routes.

### Scenario and troubleshooting

29. **A frontend VM cannot reach the backend on port 8080. Troubleshoot.** (1) Is the backend service listening (`ss -tlnp`)? (2) NSG on the backend subnet/NIC allows 8080 from the frontend; check priorities. (3) OS firewall (`ufw`). (4) Effective routes: no UDR blackholing it. (5) Both in the same VNet or correctly peered? (6) Network Watcher **IP flow verify** and **connection troubleshoot**.
30. **Two peered VNets cannot communicate.** Check peering state is `Connected` on both sides, address spaces do not overlap, NSG/UDR/firewall not blocking, forwarded traffic setting if via NVA, and DNS if using names.
31. **A private VM cannot reach the internet or run `apt update`.** New VNets are private by default. Add a NAT Gateway on the subnet, a public IP, LB outbound rule, or route via firewall. Also check NSG outbound rules and a UDR sending traffic to a firewall that blocks it.
32. **Users reach the website but the app cannot reach the database.** Check NSG `snet-data` allows the DB port from `snet-app` only (and not an old rule with the wrong source), DB listens on the private IP, Private Endpoint DNS resolves to the private IP not the public one.
33. **You cannot peer a new VNet to the hub.** Likely overlapping CIDR. Check address spaces. Fix by re-IP or using Private Link instead.
34. **How would you design the network for a new three-tier app?** VNet `/16` from the IP plan; subnets for web, app, data, Private Endpoints, plus gateway/bastion subnets as needed; NSG per subnet with tier rules; no public IPs except on App Gateway/Front Door; NAT Gateway or firewall for outbound; Private Endpoints for PaaS; Bastion for admin; peer into hub if one exists; flow logs and monitoring.
35. **How do you let spokes use the on-premises VPN without creating one per spoke?** Hub-and-spoke with the VPN Gateway in the hub, `Allow gateway transit` on the hub peering, `Use remote gateways` on each spoke.
36. **How do you securely reach VMs that have no public IP?** Azure Bastion, a jump host, VPN/ExpressRoute from the office, or Run Command.
37. **What is the difference between Azure Firewall and NSG in a VNet design?** NSG: free L3/L4 allow/deny per subnet/NIC. Firewall: central, stateful L3 to L7 with FQDN filtering, threat intelligence and logging, used with UDRs.

## 16. Common mistakes cheat sheet

| Mistake | Consequence | Correct habit |
| --- | --- | --- |
| Overlapping CIDRs across VNets/on-prem | Peering and VPN impossible | Central IP plan |
| Making every subnet `/29` or every VNet `/8` | Run out of IPs, or hoard the plan | Size by workload; `/16` VNet, `/24` subnets as a baseline |
| Forgetting the 5 reserved IPs | Subnet full earlier than expected | Usable = 2^(32−n) − 5 |
| Assuming peering is transitive | A cannot reach C | Hub + firewall + UDRs, or direct peering |
| Creating only one side of a peering | State stuck at Initiated | Create both links |
| Private VM with no outbound path | `apt`/API calls fail | NAT Gateway before creating VMs |
| Putting a UDR `0.0.0.0/0` to a firewall without rules | All outbound blocked | Firewall rules first, then route |
| Wrong subnet for special services | Deployment fails | Exact names and `/26`, `/27` sizes |
| Putting databases on public IPs | Exposure | Private subnet + Private Endpoint |
| Leaving Bastion/NAT/Firewall running in labs | Hourly cost | Delete after use |
| Allowing 22/3389 from the internet | Brute force | Bastion or JIT |
| Not enabling IP forwarding on an NVA | Traffic dropped | NIC and OS forwarding |

## 17. One-page revision sheet

- **VNet** = private isolated network, **regional**, spans all AZs, free to create, scoped to a subscription.
- **Address space** `10.0.0.0/16` = 65,536 IPs. **CIDR:** addresses = 2^(32−prefix). Longer prefix = smaller network.
- **Subnet:** regional; Azure reserves **5** IPs (`.0` network, `.1` gateway, `.2` and `.3` DNS, last broadcast); first VM gets `.4`. Min `/29`.
- **Private ranges:** `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`. **Never overlap.**
- Same VNet = open by default (`AllowVnetInBound`). Internet inbound = closed until you add a public IP and an allow rule.
- **NIC** holds the IPs, NSG link, forwarding; separate resource. **NSG** controls traffic (details in Part 2).
- **NAT Gateway:** subnet-level, outbound only, shared stable IPs, \~64k SNAT ports per IP. New VNets are private by default.
- **Route table:** system routes + UDRs; longest prefix wins; UDR > BGP > system. Next hops: `VnetLocal`, `Internet`, `VirtualNetworkGateway`, `VirtualAppliance`, `None`.
- **Peering:** private, backbone, regional or global, **non-transitive**, both directions, no overlap, charged per GB. Gateway transit for hub-and-spoke.
- **Bastion:** managed jump host; `AzureBastionSubnet` `/26`+; SKUs Developer, Basic, Standard (native client), Premium (recording).
- **Private Endpoint** = private IP for a PaaS service (needs Private DNS). **Service Endpoint** = subnet identity to the service.
- **Diagnostics:** `az network nic show-effective-route-table`, `list-effective-nsg`, Network Watcher IP flow verify, next hop, connection troubleshoot.
- **Cross-cloud:** VNet = VPC = VPC; Azure/GCP subnets regional, AWS subnets zonal; GCP VPC is global; AWS needs IGW; Azure NSG \~ AWS SG + NACL \~ GCP firewall rules.
