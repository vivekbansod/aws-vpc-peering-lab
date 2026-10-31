# AWS VPC Peering & Secure Cross-VPC Infrastructure

This repository documents the implementation and verification of a secure cross-VPC networking infrastructure on AWS using **VPC Peering**, custom **Route Tables**, and strict **Security Group** policies. 

The architecture establishes isolated virtual networks with private routing, allowing instances across different Virtual Private Clouds (VPCs) to communicate securely over private AWS network backbones without exposing internal application nodes to the public internet.

---

## 📐 Network Architecture Overview

```text
       +-----------------------------------------------------------------------+
       |                              AWS REGION                               |
       |                                                                       |
       |  +-----------------------------------+     +-----------------------+  |
       |  | VPC A (10.10.0.0/16)             |     | VPC B (10.20.0.0/16)  |  |
       |  |                                   |     |                       |  |
       |  | +-------------------------------+ |     | +-------------------+ |  |
       |  | | Public Subnet (10.10.1.0/24)  | |     | | Private Subnet    | |  |
       |  | |                               | |     | | (10.20.1.0/24)    | |  |
       |  | | [ Bastion Host ]              | |     | |                   | |  |
       |  | | - Private: 10.10.1.10         | |     | | [ App Server B ]  | |  |
       |  | | - Public:  54.252.x.x         | |     | | - Private:        | |  |
       |  | +---------------+---------------+ |     | |   10.20.1.10      | |  |
       |  |                 |                 |     | +---------+---------+ |  |
       |  | +---------------+---------------+ |     |           |           |  |
       |  | | Private Subnet (10.10.2.0/24) | |     |           |           |  |
       |  | |                               | |     |           |           |  |
       |  | | [ App Server A ]              | |     |           |           |  |
       |  | | - Private: 10.10.2.10         | |     |           |           |  |
       |  | +---------------+---------------+ |     |           |           |  |
       |  +-----------------|-----------------+     +-----------|-----------+  |
       |                    |                                   |              |
       |                    +---------> [ VPC PEERING ] <-------+              |
       |                             (pcx-0123456789abcdef0)                   |
       +-----------------------------------------------------------------------+
```

---

## 🗂️ Resource Specifications

### 1. VPC & Subnet Design

| Resource | CIDR Block | Subnet Name | Subnet CIDR | Type |
| :--- | :--- | :--- | :--- | :--- |
| **VPC A** | `10.10.0.0/16` | `ewb-a-public` | `10.10.1.0/24` | Public |
| | | `ewb-a-private` | `10.10.2.0/24` | Private |
| **VPC B** | `10.20.0.0/16` | `ewb-b-private` | `10.20.1.0/24` | Private |

### 2. EC2 Compute Instances

| Instance Name | Subnet | Private IP | Public IP | Security Group |
| :--- | :--- | :--- | :--- | :--- |
| `ewb-bastion` | `ewb-a-public` | `10.10.1.10` | Assigned (`54.252.x.x`) | `ewb-sg-bastion` |
| `ewb-app-a` | `ewb-a-private` | `10.10.2.10` | None | `ewb-sg-app-a` |
| `ewb-app-b` | `ewb-b-private` | `10.20.1.10` | None | `ewb-sg-app-b` |

---

## 🔒 Security Group Policies

To enforce strict least-privilege network control, security groups are scoped to exact IP addresses and security group references:

* **`ewb-sg-bastion`**: Allows SSH (`Port 22`) only from the operator's authorized public IP address (`203.0.113.x/32`).
* **`ewb-sg-app-a`**: Allows ICMP Echo Requests (`Ping`) and SSH (`Port 22`) originating exclusively from the Bastion Security Group (`ewb-sg-bastion`).
* **`ewb-sg-app-b`**: Allows ICMP Echo Requests (`Ping`) and SSH (`Port 22`) originating from `VPC A` subnets across the VPC Peering connection (`10.10.0.0/16`).

---

## 🛣️ Routing Rules & Cross-VPC Peering

A VPC Peering Connection (`pcx-0123456789abcdef0`) was created between `VPC A` (Requester) and `VPC B` (Accepter).

### Route Table Configurations

* **VPC A Route Table (`rtb-vpc-a`)**:
  * `10.10.0.0/16` $\rightarrow$ `local`
  * `0.0.0.0/0` $\rightarrow$ `igw-0123456789abcdef0` (Public Subnet only)
  * `10.20.0.0/16` $\rightarrow$ `pcx-0123456789abcdef0`
* **VPC B Route Table (`rtb-vpc-b`)**:
  * `10.20.0.0/16` $\rightarrow$ `local`
  * `10.10.0.0/16` $\rightarrow$ `pcx-0123456789abcdef0`

---

## 🧪 Verification & Testing Steps

1. **Bastion Host Access**: Connected via SSH to the bastion host (`ewb-bastion`) in `VPC A`.
2. **Local Subnet Connectivity**: Tested internal ICMP ping from `ewb-bastion` (`10.10.1.10`) to `ewb-app-a` (`10.10.2.10`).
3. **Cross-VPC Peering Verification**: Tested cross-VPC ICMP ping from `ewb-bastion` (`10.10.1.10`) to `ewb-app-b` (`10.20.1.10`) in `VPC B`.
4. **Host Identification Handling**: Cleared stale SSH fingerprints following instance redeployment using:
   ```bash
   ssh-keygen -R 10.10.2.10
   ssh-keygen -R 10.20.1.10
   ```

---

## 🧹 Security & Post-Lab Cleanup Notice

In accordance with AWS cloud security best practices and cost management guidelines:
* All provisioned lab resources (EC2 instances, VPC Peering connections, custom Route Tables, and Internet Gateways) were terminated and deleted immediately following project validation.
* All public IP addresses displayed in documentation are randomized placeholders or disassociated AWS pool addresses.
