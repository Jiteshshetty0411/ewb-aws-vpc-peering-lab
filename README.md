# AWS VPC Peering & Private Connectivity Lab

A hands-on implementation of private network connectivity between two Amazon Web Services (AWS) Virtual Private Clouds (VPCs) within the `ap-south-1` (Mumbai) region. This project demonstrates non-overlapping IPv4 CIDR design, custom route table configuration, explicit security group scoping, SSH ProxyJump administration via a Bastion host, and comprehensive network traffic validation.

---

## 🏗️ Architecture Overview

The infrastructure consists of two distinct VPCs:
* **VPC A (`10.10.0.0/16`)**: Houses a public subnet for the Bastion host (`10.10.1.10`) with Internet Gateway access, and a private subnet for `App-A` (`10.10.2.10`).
* **VPC B (`10.20.0.0/16`)**: Houses a completely private subnet for `App-B` (`10.20.1.10`) with no direct Internet Gateway.
* **VPC Peering Connection (`pcx-...`)**: Links VPC A and VPC B directly over private AWS network infrastructure without traversing the public internet.

---

## 🏗️ Architecture Diagram

```text
 [ Your Laptop ] 
       │
       │  (SSH / Port 22 via ewb-lab.pem)
       ▼
+---------------------------------------------------------------------------------+
| AWS Region: ap-south-1 (Mumbai)                                                 |
|                                                                                 |
|  +-----------------------------------+        +------------------------------+  |
|  | VPC A: ewb-vpc-a                  |        | VPC B: ewb-vpc-b             |  |
|  | CIDR: 10.10.0.0/16                |        | CIDR: 10.20.0.0/16           |  |
|  |                                   |        |                              |  |
|  |  +-----------------------------+  |        |                              |  |
|  |  | Public Subnet: ewb-a-public |  |        |                              |  |
|  |  | CIDR: 10.10.1.0/24          |  |        |                              |  |
|  |  | [ Bastion Host ]            |  |        |                              |  |
|  |  | IP: 10.10.1.10 (Public IP)  |  |        |                              |  |
|  |  +--------------┬--------------+  |        |                              |  |
|  |                 │ (ProxyJump)     |        |                              |  |
|  |                 ▼                 |        |                              |  |
|  |  +-----------------------------+  |        |  +-------------------------+ |  |
|  |  | Private Subnet: ewb-a-private| | Peering|  | Private Subnet: ewb-b   | |  |
|  |  | CIDR: 10.10.2.0/24          | ◄========►   | CIDR: 10.20.1.0/24      | |  |
|  |  | [ App-A Server ]            |  | (pcx)  |  | [ App-B Server ]        | |  |
|  |  | IP: 10.10.2.10              |  |        |  | IP: 10.20.1.10          | |  |
|  |  +-----------------------------+  |        |  +-------------------------+ |  |
|  |                                   |        |                              |  |
|  +-----------------------------------+        +------------------------------+  |
|                    ▲                                                            |
|                    │ (Attached via ewb-igw-a)                                   |
|         [ Internet Gateway (IGW) ]                                              |
+---------------------------------------------------------------------------------+
```

---

## 📋 Resource Inventory Table

| Resource Name | Type | Resource ID | IPv4 CIDR / IP | Subnet / Associated RT |
| :--- | :--- | :--- | :--- | :--- |
| `ewb-vpc-a` | VPC | `vpc-XXXXXXXXXXXXXXXXX` | `10.10.0.0/16` | N/A |
| `ewb-vpc-b` | VPC | `vpc-YYYYYYYYYYYYYYYYY` | `10.20.0.0/16` | N/A |
| `ewb-a-public` | Subnet (Public) | `subnet-AAAAAAAAAAAAA` | `10.10.1.0/24` | `ewb-rt-a-public` |
| `ewb-a-private` | Subnet (Private) | `subnet-BBBBBBBBBBBBB` | `10.10.2.0/24` | `ewb-rt-a-private` |
| `ewb-b-private` | Subnet (Private) | `subnet-CCCCCCCCCCCCC` | `10.20.1.0/24` | `ewb-rt-b-private` |
| `ewb-bastion` | EC2 Instance | `i-11111111111111111` | `10.10.1.10` / Public IP | `ewb-a-public` |
| `ewb-app-a` | EC2 Instance | `i-22222222222222222` | `10.10.2.10` / No Public IP | `ewb-a-private` |
| `ewb-app-b` | EC2 Instance | `i-33333333333333333` | `10.20.1.10` / No Public IP | `ewb-b-private` |
| `ewb-a-b-peer` | VPC Peering | `pcx-ZZZZZZZZZZZZZZZ` | N/A | Status: **Active** |

---

## 🛣️ Route Tables Configuration

### 1. Public Route Table (`ewb-rt-a-public`)
* `10.10.0.0/16` → `local`
* `0.0.0.0/0` → `igw-ewb-igw-a` (Internet Access for Bastion)
* `10.20.0.0/16` → `pcx-a-b-peer` (Bastion ProxyJump path to App-B)

### 2. Private Route Table A (`ewb-rt-a-private`)
* `10.10.0.0/16` → `local`
* `10.20.0.0/16` → `pcx-a-b-peer` (Outbound path to App-B)

### 3. Private Route Table B (`ewb-rt-b-private`)
* `10.20.0.0/16` → `local`
* `10.10.0.0/16` → `pcx-a-b-peer` (Return path to VPC A)

---

## 🔒 Security Group Rules

* **`ewb-sg-bastion`**: Allows Inbound SSH (Port 22) strictly from your personal Public IP (`YOUR_LOCAL_IP/32`).
* **`ewb-sg-app-a`**:
  * SSH (Port 22) from Bastion Private IP (`10.10.1.10/32`).
  * Custom ICMP Echo Request from App-B Private IP (`10.20.1.10/32`).
* **`ewb-sg-app-b`**:
  * SSH (Port 22) from Bastion Private IP (`10.10.1.10/32`).
  * Custom ICMP Echo Request from App-A Private IP (`10.10.2.10/32`).

---

## 🧪 Validation & Test Matrix

| Test Scenario | Origin Host | Target IP | Command | Expected Result | Result |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **01/ Baseline Failure** | App-A | `10.20.1.10` | `ping -c 4 10.20.1.10` | 100% Packet Loss (Pre-routes) | PASSED |
| **02/ Forward Path** | App-A | `10.20.1.10` | `ping -c 4 10.20.1.10` | 0% Packet Loss (4 Received) | PASSED |
| **03/ Reverse Path** | App-B | `10.10.2.10` | `ping -c 4 10.10.2.10` | 0% Packet Loss (4 Received) | PASSED |
| **04/ SSH Tunnel** | Local Laptop | App-B | `ssh ewb-app-b` | Shell prompt on App-B via Bastion | PASSED |
| **05/ Controlled Failure**| App-A | `10.20.1.10` | `ping -c 4 10.20.1.10` | Packet loss upon ICMP rule removal | PASSED |
| **06/ Failure Recovery** | App-A | `10.20.1.10` | `ping -c 4 10.20.1.10` | Pings resume upon rule restoration | PASSED |

---

## 💡 Key Learnings & Architecture Notes

1. **Stateful Security Groups**: AWS Security Groups are stateful. When ICMP Echo Request traffic is allowed inbound, return response traffic is automatically allowed out regardless of outbound rules.
2. **Asymmetric Routing Prerequisite**: Establishing a VPC Peering Connection is not sufficient by itself. Both the requester and accepter VPC route tables must explicitly define destination routes pointing to the `pcx` target for bidirectional reachability.
3. **Non-Transitive Peering**: VPC Peering connections do not support transitive routing (e.g., VPC B cannot access the Internet Gateway in VPC A through the peering connection).
4. **SSH ProxyJump**: Utilizing standard SSH configuration (`~/.ssh/config`) with `ProxyJump` allows management of isolated private EC2 instances without placing `.pem` private keys on intermediate Bastion hosts.

---

## 📁 Repository Structure

```text
.
├── README.md
├── report/
│   └── Jitesh_Shetty_EWB_VPC_Peering.pdf
└── screenshots/
    ├── 01-subnets-and-route-tables/
    ├── 02-peering-and-security-groups/
    ├── 03-testing-evidence/
    └── 04-cleanup-evidence/
```
---

## 🙏 Acknowledgements & References

This project serves as a practical proof-of-work built to showcase the AWS Cloud Internship provided by **EWB Tech Academy**. I would like to express my sincere gratitude to EWB Tech Academy for their exceptional work, structured guidance, and commitment to empowering cloud engineering learners through real-world scenarios.

* **Learning Resources & Sessions:** [EWB Tech Academy YouTube Playlist](https://youtube.com/playlist?list=PLMTW_dEnPsEA&si=oYr_DMzZyw1wmxrd)
* **Official Documentation:** [AWS VPC Peering Documentation](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html)