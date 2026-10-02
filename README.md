# AZ-700 Lab: Azure Network Management and Connectivity

## Azure Virtual Network Manager, NAT Gateway, Public IP Prefix, DNS Private Resolver and Route Server

## 1. Lab Overview

In this lab, you will build a simplified **hub-and-spoke network architecture in Azure Central India**.

You will use:

- **Azure Virtual Network Manager (AVNM)** to create and centrally manage hub-and-spoke connectivity.
- **AVNM Security Admin Rules** to demonstrate centralized network security governance.
- **Azure NAT Gateway** to provide outbound Internet connectivity for the Management subnet.
- **Azure Public IP Prefix** to provide a predictable pool of public IP addresses for NAT Gateway.
- **Azure DNS Private Resolver** to demonstrate conditional DNS forwarding toward an on-premises DNS server.
- **Azure Firewall** as an optional architecture/deployment exercise.
- **Azure Route Server** as an architecture and design exercise only.

The lab deliberately does **not** implement an actual VPN/ExpressRoute connection to on-premises.

---

## 2. Scenario

SimplyKLOUD is deploying its Azure environment in **Central India**.

The initial environment consists of:

- One Hub VNet
- One Management Spoke VNet
- One Production Spoke VNet

```text
                         SIMPLYKLOUD
                         ON-PREMISES
                       192.168.0.0/16
                             |
                       DNS Server
                      192.168.10.10
                             |
                      simplykloud.local
                             |
                    Future VPN / ER
                             X
                       Not configured
                             |
                     +--------------+
                     |   HUB VNET   |
                     |  10.0.0.0/16 |
                     |              |
                     | DNS Resolver |
                     | Firewall*    |
                     | Route Server*|
                     +------+-------+
                            |
                   AVNM Hub-Spoke
                   Connectivity
                       /      \
                      /        \
                     /          \
          +--------------+   +--------------+
          | MANAGEMENT   |   | PRODUCTION   |
          | 10.1.0.0/16  |   | 10.2.0.0/16  |
          |              |   |              |
          | Management VM|   | Production VM|
          | NAT Gateway  |   |              |
          +--------------+   +--------------+

* Optional / design exercise
```

---

## 3. Learning Objectives

At the end of the lab, you should be able to:

1. Create a basic Azure hub-and-spoke architecture.
2. Explain the role of Azure Virtual Network Manager.
3. Create an AVNM Network Group containing spoke VNets.
4. Create an AVNM Hub-and-Spoke Connectivity Configuration.
5. Apply AVNM Security Admin Rules.
6. Explain the difference between Security Admin Rules and NSGs.
7. Deploy a NAT Gateway.
8. Use a Public IP Prefix with NAT Gateway.
9. Verify outbound Internet connectivity through NAT Gateway.
10. Deploy Azure DNS Private Resolver.
11. Configure conditional DNS forwarding.
12. Explain how Azure DNS Private Resolver would communicate with on-premises DNS.
13. Explain the role of Azure Route Server and BGP.
14. Identify situations where Route Server would be useful.
15. Explain why Route Server is not required for this basic hub-and-spoke environment.

---

## 4. Lab Resource Design

All resources should be created in:

**Region: Central India**

Use a single resource group:

```text
rg-az700-networking
```

### VNets

| VNet | Address Space |
|---|---|
| `vnet-hub-ci` | `10.0.0.0/16` |
| `vnet-mgmt-ci` | `10.1.0.0/16` |
| `vnet-prod-ci` | `10.2.0.0/16` |

---

## 5. Subnet Design

### Hub VNet

Create:

| Subnet | Address |
|---|---|
| `AzureFirewallSubnet` | `10.0.0.0/24` |
| `snet-dns-inbound` | `10.0.1.0/28` |
| `snet-dns-outbound` | `10.0.2.0/28` |
| `RouteServerSubnet` | `10.0.3.0/26` |

The following are **design reservations**:

- `AzureFirewallSubnet`
- `RouteServerSubnet`

The Azure Firewall and Azure Route Server do not need to be deployed for the main lab.

### Management VNet

```text
snet-management
10.1.1.0/24
```

### Production VNet

```text
snet-production
10.2.1.0/24
```

---

# 6. Part 1 — Create the VNets

Create the following three VNets.

### Hub

```text
Name: vnet-hub-ci
Address space: 10.0.0.0/16
Region: Central India
```

Create:

```text
AzureFirewallSubnet     10.0.0.0/24
snet-dns-inbound        10.0.1.0/28
snet-dns-outbound       10.0.2.0/28
RouteServerSubnet       10.0.3.0/26
```

### Management

```text
Name: vnet-mgmt-ci
Address space: 10.1.0.0/16
Region: Central India
```

Create:

```text
snet-management
10.1.1.0/24
```

### Production

```text
Name: vnet-prod-ci
Address space: 10.2.0.0/16
Region: Central India
```

Create:

```text
snet-production
10.2.1.0/24
```

---

# 7. Part 2 — Create Test VMs

Create one VM in each spoke.

## Management VM

```text
Name: vm-mgmt-01
VNet: vnet-mgmt-ci
Subnet: snet-management
```

Use a small VM size suitable for training.

For Windows:

```text
Windows Server 2022/2025
```

For Linux:

```text
Ubuntu
```

## Production VM

```text
Name: vm-prod-01
VNet: vnet-prod-ci
Subnet: snet-production
```

The VMs are primarily used to test:

- Network connectivity
- Security rules
- NAT Gateway
- Routing

---

# 8. Part 3 — Create Azure Virtual Network Manager

Search for:

**Virtual Network Manager**

Select:

**Create**

Use:

```text
Name:
avnm-az700

Region:
Central India

Scope:
Your subscription
```

Complete the deployment.

---

# 9. Part 4 — Create the AVNM Network Group

Go to:

**Virtual Network Manager → Network Groups**

Create:

```text
ng-az700-spokes
```

### Important

Add only:

```text
vnet-mgmt-ci
vnet-prod-ci
```

### DO NOT add:

```text
vnet-hub-ci
```

The Hub VNet is selected separately when creating the Hub-and-Spoke Connectivity Configuration.

Your structure should therefore be:

```text
Azure Virtual Network Manager
           |
           +-- Network Group
               |
               +-- vnet-mgmt-ci
               |
               +-- vnet-prod-ci
```

The Hub remains outside the Network Group.

---

# 10. Part 5 — Create Hub-and-Spoke Connectivity

Go to:

**Virtual Network Manager → Configurations**

Create a:

**Connectivity Configuration**

Name:

```text
cc-hub-spoke
```

Select:

```text
Topology: Hub and spoke
```

Select the Network Group:

```text
ng-az700-spokes
```

For the Hub VNet, select:

```text
vnet-hub-ci
```

The resulting topology is:

```text
                vnet-hub-ci
                     |
              +------+------+
              |             |
       vnet-mgmt-ci    vnet-prod-ci
```

### Important

The Hub VNet is **not** a member of the Network Group. It is selected as the Hub in the Connectivity Configuration.

---

# 11. Deploy the Connectivity Configuration

After creating the configuration:

**Deploy → Deploy configuration**

Select the appropriate:

- Subscription
- Region

Deploy the configuration.

Wait for the deployment to complete.

---

# 12. Verify Hub-and-Spoke Connectivity

Open:

```text
vnet-hub-ci
```

Check:

**Peerings**

Also check:

```text
vnet-mgmt-ci
vnet-prod-ci
```

Verify that connectivity has been established.

---

# 13. Important Question

### Why didn't we add the Hub VNet to the Network Group?

Because the **Hub-and-Spoke Connectivity Configuration identifies the Hub separately**.

The Network Group represents the collection of spoke VNets participating in the configuration.

Conceptually:

```text
Network Group
     |
     +-- Spoke 1
     +-- Spoke 2
     +-- Spoke 3

Connectivity Configuration
     |
     +-- Hub = Hub VNet
     +-- Spokes = Network Group
```

---

# 14. Part 6 — Configure AVNM Security Admin Rules

Go to:

**Virtual Network Manager → Configurations**

Create a:

**Security Configuration**

Name:

```text
sc-az700-security
```

Create a rule collection:

```text
rc-global-security
```

---

# 15. Create Security Admin Rule

Create a rule that blocks Internet-originated RDP.

Example:

```text
Name:
Deny-Internet-RDP

Direction:
Inbound

Protocol:
TCP

Source:
Internet

Destination:
Any

Destination Port:
3389

Action:
Deny
```

Use an appropriate priority.

Deploy the security configuration to:

```text
ng-az700-spokes
```

---

# 16. Security Admin Rules vs NSG

### Security Admin Rules

Used for:

- Centralized governance
- Organization-wide security requirements
- Network administrator controls
- Rules applied through AVNM

### NSGs

Used for:

- Subnet-level security
- NIC-level security
- Application/workload-specific security
- More granular workload controls

Conceptually:

```text
          AVNM
            |
     Security Admin Rule
            |
            v
           NSG
            |
            v
        NIC / VM
```

### Question

If the application team creates an NSG allowing Internet RDP, but the organization's AVNM Security Admin Rule denies Internet RDP, which policy should control the traffic?

Discuss this with your instructor.

---

# 17. Part 7 — Azure Firewall Subnet

The Hub already contains:

```text
AzureFirewallSubnet
10.0.0.0/24
```

For this lab, **Azure Firewall deployment is optional**.

### Optional deployment

Deploy:

```text
Azure Firewall
Name: fw-hub-ci
```

Use:

```text
AzureFirewallSubnet
```

Do not spend significant time creating complex firewall policies.

The purpose is simply to demonstrate where the firewall fits:

```text
                    HUB
                     |
              Azure Firewall
                     |
            +--------+--------+
            |                 |
       Management         Production
```

### Important

Do not configure the Management subnet to force all Internet traffic through the firewall.

The lab specifically demonstrates another architecture:

```text
Management VM
      |
 NAT Gateway
      |
 Internet
```

---

# 18. Architecture Question — Why Doesn't Everything Need to Go Through Firewall?

A common misconception is:

> "If I have Azure Firewall in the Hub, every spoke must send Internet traffic through it."

That is not automatically true.

Traffic forwarding depends on routing.

For example:

```text
Management
    |
NAT Gateway
    |
Internet
```

can coexist with:

```text
Production
    |
UDR
    |
Azure Firewall
    |
Internet
```

This gives the architect flexibility to choose the appropriate security and routing model for each workload.

---

# 19. Part 8 — Create Public IP Prefix

Search for:

**Public IP Prefix**

Create:

```text
Name:
pip-prefix-mgmt

Region:
Central India

Prefix:
Small training prefix
```

For example:

```text
/29
```

A `/29` provides a small contiguous range of public IP addresses suitable for this training demonstration.

---

# 20. Part 9 — Create NAT Gateway

Search for:

**NAT Gateway**

Create:

```text
Name:
nat-mgmt-ci

Region:
Central India
```

Associate the Public IP Prefix created earlier.

```text
nat-mgmt-ci
      |
      +-- Public IP Prefix
          pip-prefix-mgmt
```

---

# 21. Associate NAT Gateway with Management Subnet

Open:

```text
vnet-mgmt-ci
```

Go to:

**Subnets → snet-management**

Associate:

```text
NAT Gateway:
nat-mgmt-ci
```

Save the configuration.

---

# 22. Test NAT Gateway

Connect to:

```text
vm-mgmt-01
```

### Windows PowerShell

```powershell
(Invoke-WebRequest -Uri "https://api.ipify.org").Content
```

### Linux

```bash
curl https://api.ipify.org
```

Record the result.

It should correspond to an address associated with the NAT Gateway's public IP configuration.

---

# 23. Important NAT Gateway Questions

### Question 1

Does NAT Gateway provide inbound Internet connectivity?

**No.**

### Question 2

Does NAT Gateway inspect application traffic?

**No.**

### Question 3

Is NAT Gateway a firewall?

**No.**

### Question 4

What is its primary purpose?

**Providing scalable outbound SNAT connectivity.**

---

# 24. Why Use a Public IP Prefix?

Imagine that SimplyKLOUD has several public-facing services and the security team wants to allow-list a known range of public IP addresses.

Instead of requesting unrelated public IP addresses individually, a Public IP Prefix provides a predictable contiguous range.

```text
Public IP Prefix
      |
      +-- IP 1
      +-- IP 2
      +-- IP 3
      +-- IP 4
      +-- ...
```

---

# 25. Part 10 — Azure DNS Private Resolver

Search for:

**DNS Private Resolver**

Create:

```text
Name:
dnspr-hub-ci

Region:
Central India

VNet:
vnet-hub-ci
```

---

# 26. Configure Inbound Endpoint

Create an inbound endpoint:

```text
Name:
inbound-dns

Subnet:
snet-dns-inbound
```

Subnet:

```text
10.0.1.0/28
```

---

# 27. Configure Outbound Endpoint

Create:

```text
Name:
outbound-dns

Subnet:
snet-dns-outbound
```

Subnet:

```text
10.0.2.0/28
```

The outbound endpoint will be used for DNS forwarding toward external/on-premises DNS servers.

---

# 28. Create DNS Forwarding Ruleset

Create:

```text
Name:
dnsruleset-simplykloud
```

Create a rule for:

```text
Domain:
simplykloud.local
```

Forward the query to:

```text
192.168.10.10
```

Optionally add:

```text
192.168.10.11
```

Architecture:

```text
Azure VM
   |
DNS Query
   |
DNS Private Resolver
   |
Forwarding Rule
   |
simplykloud.local
   |
192.168.10.10
   |
On-premises DNS
```

---

# 29. Important: There Is No On-Premises Connection

The lab does **not** create:

- Site-to-Site VPN
- ExpressRoute
- Virtual WAN
- On-premises network

Therefore:

```text
Azure
 |
DNS Private Resolver
 |
192.168.10.10
 |
X
 |
No network path
```

The forwarding configuration exists, but the hypothetical DNS server cannot actually be reached.

This is intentional.

---

# 30. Why Configure It If It Doesn't Work?

The objective is to understand the architecture.

In a real environment:

```text
Azure
 |
DNS Private Resolver
 |
VPN / ExpressRoute
 |
On-premises
 |
DNS Server
192.168.10.10
```

The DNS forwarding rule would then allow Azure workloads to resolve:

```text
server01.simplykloud.local
database.simplykloud.local
dc01.simplykloud.local
```

through the on-premises DNS infrastructure.

---

# 31. DNS Architecture Question

### Question

Why not configure:

```text
192.168.10.10
```

directly as the DNS server for every VNet?

Discuss:

- Centralized DNS architecture
- Hybrid DNS
- Conditional forwarding
- Avoiding direct dependency from every VNet
- Future scalability
- Hub-based network architecture

---

# 32. Part 11 — Azure Route Server

### This section is design-only.

Do **not** deploy Azure Route Server as part of the main lab.

Route Server becomes much more meaningful when an NVA is available to establish BGP peering.

---

# 33. Route Server Scenario

Imagine SimplyKLOUD later deploys an SD-WAN NVA.

```text
                    SD-WAN NVA
                         |
                       BGP
                         |
                  Azure Route Server
                         |
                  Azure VNet routing
```

The NVA may advertise:

```text
192.168.0.0/16
```

representing the on-premises network.

Instead of manually creating UDRs everywhere, BGP can dynamically exchange routes.

---

# 34. Route Server Question

### Without Route Server

You could have:

```text
NVA
 |
Manual UDR
 |
Azure
```

As the environment grows:

```text
NVA 1
NVA 2
VPN
SD-WAN
ExpressRoute
Multiple VNets
```

manual route management becomes more difficult.

### With Route Server

Conceptually:

```text
                 Azure Route Server
                  /             \
                BGP             BGP
                /                 \
          SD-WAN NVA          Other NVA
                |
          On-premises
```

Route Server provides a centralized BGP route exchange point between Azure and supported network virtual appliances.

### Important

Azure Route Server does **not** become the traffic-forwarding device.

It exchanges routes.

The actual traffic still travels through the appropriate network path/NVA.

---

# 35. Route Server Design Questions

Discuss:

1. Why would an organization use Route Server?
2. Why is BGP useful in a large network?
3. What is the role of the NVA?
4. Why do we need a dedicated `RouteServerSubnet`?
5. Why are we not deploying Route Server in this lab?

Expected answer to question 5:

> Because a meaningful Route Server demonstration requires a suitable NVA/BGP scenario, and deploying Route Server without an NVA would add deployment time without providing a useful practical demonstration.

---

# 36. Final Architecture

```text
                         ON-PREMISES
                      192.168.0.0/16
                            |
                     DNS 192.168.10.10
                            |
                     simplykloud.local
                            |
                      Future VPN/ER
                            X
                       Not deployed
                            |
                   +------------------+
                   |    HUB VNET      |
                   |   10.0.0.0/16    |
                   |                  |
                   | Firewall Subnet  |
                   | DNS Inbound      |
                   | DNS Outbound     |
                   | RouteServerSubnet|
                   +--------+---------+
                            |
                     AVNM Hub-Spoke
                            |
                 +----------+----------+
                 |                     |
                 |                     |
        MANAGEMENT VNET          PRODUCTION VNET
         10.1.0.0/16             10.2.0.0/16
                 |                     |
          Management VM            Production VM
                 |                     |
           NAT Gateway                  |
                 |                     |
         Public IP Prefix              |
                 |                     |
              Internet                 |
```

---

# 37. Final Validation Checklist

### Virtual Network Manager

- [ ] AVNM created
- [ ] Spoke Network Group created
- [ ] Management VNet added
- [ ] Production VNet added
- [ ] Hub VNet **not** added to Network Group
- [ ] Hub-and-Spoke Connectivity Configuration created
- [ ] Hub VNet selected separately
- [ ] Configuration deployed
- [ ] Connectivity verified

### Security

- [ ] Security Configuration created
- [ ] Security Admin Rule created
- [ ] Internet RDP restriction configured
- [ ] Security Admin Rules deployed
- [ ] Difference between AVNM and NSG understood

### Firewall

- [ ] `AzureFirewallSubnet` exists
- [ ] Firewall deployment understood
- [ ] Firewall deployment is optional

### NAT Gateway

- [ ] Public IP Prefix created
- [ ] NAT Gateway created
- [ ] Public IP Prefix associated
- [ ] NAT Gateway associated with Management subnet
- [ ] Management VM outbound connectivity tested

### DNS

- [ ] DNS Private Resolver created
- [ ] Inbound endpoint created
- [ ] Outbound endpoint created
- [ ] DNS forwarding ruleset created
- [ ] `simplykloud.local` rule created
- [ ] `192.168.10.10` configured as hypothetical on-premises DNS
- [ ] Lack of VPN/on-prem connectivity understood

### Route Server

- [ ] Route Server deployment not performed
- [ ] Route Server subnet understood
- [ ] BGP concept understood
- [ ] NVA requirement understood
- [ ] Route Server vs UDR discussed

---

# 38. Cleanup

When the lab is complete, the simplest cleanup method is to delete:

```text
rg-az700-networking
```

This removes the resources created for the lab.

> **Warning:** Do not delete the resource group if it contains resources that you need for other labs.

---

# 39. AZ-700 Review Questions

1. Why should the Hub VNet not be added to the AVNM spoke Network Group?
2. What is the purpose of an AVNM Network Group?
3. What is the difference between AVNM Connectivity Configuration and Security Configuration?
4. What is the difference between AVNM Security Admin Rules and NSGs?
5. Can an NSG override an AVNM security rule?
6. What is the primary purpose of NAT Gateway?
7. Does NAT Gateway provide inbound connectivity?
8. Why might an organization use a Public IP Prefix?
9. What is the difference between NAT Gateway and Azure Firewall?
10. Why might Management traffic use NAT Gateway while Production traffic uses Azure Firewall?
11. What is the purpose of the DNS Private Resolver inbound endpoint?
12. What is the purpose of the outbound endpoint?
13. What is a DNS forwarding rule?
14. Why would we forward `simplykloud.local` to `192.168.10.10`?
15. Why doesn't DNS resolution actually work in this lab?
16. What additional component is required to make the hypothetical on-premises DNS reachable?
17. What problem does Azure Route Server solve?
18. What protocol does Route Server use to exchange routes?
19. What is the role of the NVA?
20. Does Azure Route Server forward application traffic?

---

## Instructor Note

The intended learning flow is:

```text
AVNM
 ↓
"How do I centrally manage my network?"
 ↓
Hub-Spoke Connectivity
 ↓
"How do I centrally enforce security?"
 ↓
Security Admin Rules
 ↓
"How do I give this subnet outbound Internet?"
 ↓
NAT Gateway
 ↓
"How do I control the public IP range?"
 ↓
Public IP Prefix
 ↓
"How do I resolve on-premises names?"
 ↓
DNS Private Resolver
 ↓
"What if I have SD-WAN/NVAs and dynamic routes?"
 ↓
Route Server + BGP
```

This provides a coherent AZ-700 architecture lab rather than five unrelated service demonstrations.
