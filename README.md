# AWS Multi-VPC Peering & Network Security Architecture Report

## 1. Executive Summary
This project demonstrates the design, deployment, and validation of a secure multi-VPC cloud network on Amazon Web Services (AWS). The core objective was establishing non-transitive, private Layer 3 connectivity between two isolated Virtual Private Clouds (`10.10.0.0/16` and `10.20.0.0/16`) via AWS VPC Peering.

Administrative host management is conducted securely via an OpenSSH ProxyJump Bastion architecture without exposing private application subnets to the public internet. Defense-in-depth is implemented through strict route table scoping and stateful Security Group boundaries.

---

## 2. Infrastructure & Network Topology

| Resource Name | Type | CIDR / IP Address | Role / Security Function |
| :--- | :--- | :--- | :--- |
| **ewb-vpc-a** | AWS VPC | `10.10.0.0/16` | Requester VPC |
| **ewb-vpc-b** | AWS VPC | `10.20.0.0/16` | Accepter VPC |
| **ewb-subnet-a-public** | Public Subnet | `10.10.1.0/24` | Attached to Internet Gateway (`ewb-igw-a`) |
| **ewb-subnet-a-private** | Private Subnet | `10.10.2.0/24` | Isolated internal workload subnet (VPC A) |
| **ewb-subnet-b-private** | Private Subnet | `10.20.1.0/24` | Isolated internal workload subnet (VPC B) |
| **ewb-bastion** | EC2 (t2.micro) | `32.236.173.191` (Public) / `10.10.1.10` | Administrative Jump Host (Port 22 locked to admin `/32`) |
| **ewb-app-a** | EC2 (t2.micro) | `10.10.2.10` (Private) | Private workload instance in VPC A |
| **ewb-app-b** | EC2 (t2.micro) | `10.20.1.10` (Private) | Private workload instance in VPC B |
| **ewb-a-b-peer** | VPC Peering Link | Active (`pcx-...`) | Point-to-point Layer 3 routing link |

---

## 3. Administrative Access Configuration

All administrative sessions to private application subnets are tunneled through the Bastion jump host using OpenSSH ProxyJump. No application instances hold public IP addresses.

### OpenSSH Client Configuration (`~/.ssh/config`)
```ssh-config
Host ewb-bastion
    HostName 32.236.173.191
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes

Host ewb-app-a
    HostName 10.10.2.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes
    ProxyJump ewb-bastion

Host ewb-app-b
    HostName 10.20.1.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes
    ProxyJump ewb-bastion
```
## 4. Empirical Test Results & Verification

### Test 01: Baseline Network Isolation (Pre-Peering)

* **Objective:** Verify absolute network isolation between disparate VPCs before route definition.
* **Source Node:** `ewb-app-a` (`10.10.2.10`)
* **Target Node:** `ewb-app-b` (`10.20.1.10`)
* **Execution:**

```bash
hostname -I
ping -c 4 10.20.1.10
```

- **Output:**

```text
PING 10.20.1.10 (10.20.1.10) 56(84) bytes of data.
--- 10.20.1.10 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3072ms
```

- **Analysis:** Successful negative control. Packets failed to reach the target because no route existed for destination CIDR `10.20.0.0/16` in the source VPC route table.

---

### Test 02: Forward Peering Transit (VPC A to VPC B)

* **Objective:** Validate private Layer 3 packet transit across active VPC Peering.
* **Route Injected:** `10.20.0.0/16` -> Target: `pcx-...` on `ewb-rt-a-private`
* **Source Node:** `ewb-app-a` (`10.10.2.10`)
* **Target Node:** `ewb-app-b` (`10.20.1.10`)
* **Execution:**

```bash
ping -c 4 10.20.1.10
```

- **Output:**

```text
PING 10.20.1.10 (10.20.1.10) 56(84) bytes of data.
64 bytes from 10.20.1.10: icmp_seq=1 ttl=64 time=1.04 ms
64 bytes from 10.20.1.10: icmp_seq=2 ttl=64 time=0.98 ms
64 bytes from 10.20.1.10: icmp_seq=3 ttl=64 time=1.01 ms
64 bytes from 10.20.1.10: icmp_seq=4 ttl=64 time=1.02 ms

--- 10.20.1.10 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 0.980/1.012/1.040/0.022 ms
```

- **Analysis:** Verified forward-path transit through private AWS VPC Peering without traversing the public internet. The measured average round-trip time was approximately `1.012 ms`.

---

### Test 03: Reverse Symmetric Routing (VPC B to VPC A)

* **Objective:** Confirm bidirectional routing symmetry and return-path viability.
* **Route Injected:** `10.10.0.0/16` -> Target: `pcx-...` on `ewb-rt-b-private`
* **Source Node:** `ewb-app-b` (`10.20.1.10`)
* **Target Node:** `ewb-app-a` (`10.10.2.10`)
* **Execution:**

```bash
hostname -I
ping -c 4 10.10.2.10
```

- **Output:**

```text
10.20.1.10
PING 10.10.2.10 (10.10.2.10) 56(84) bytes of data.
64 bytes from 10.10.2.10: icmp_seq=1 ttl=64 time=1.02 ms
64 bytes from 10.10.2.10: icmp_seq=2 ttl=64 time=0.99 ms
64 bytes from 10.10.2.10: icmp_seq=3 ttl=64 time=1.01 ms
64 bytes from 10.10.2.10: icmp_seq=4 ttl=64 time=1.03 ms

--- 10.10.2.10 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3003ms
```

- **Analysis:** Confirmed bidirectional connectivity through the configured routes in both VPC route tables.

---

### Test 04: Controlled Failure (Security Group Boundary)

* **Objective:** Demonstrate decoupling of Layer 3 Routing and Layer 4/Stateful Firewall rules.
* **Intervention:** Inbound ICMP rule temporarily deleted from `ewb-sg-app-b` while route tables remained configured.
* **Source Node:** `ewb-app-a` (`10.10.2.10`)
* **Target Node:** `ewb-app-b` (`10.20.1.10`)
* **Execution:**

```bash
ping -c 4 10.20.1.10
```

- **Output:**

```text
PING 10.20.1.10 (10.20.1.10) 56(84) bytes of data.
--- 10.20.1.10 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3080ms
```

- **Analysis:** Confirms defense-in-depth: the VPC router permits transit across peering, but the destination Security Group blocks incoming ICMP traffic.

---

## 5. Security Posture & Hardening Summary

1. **Least Privilege Ingress:** Port 22 administrative access to `ewb-bastion` is restricted strictly to the authorized administrator's public IPv4 `/32`.

2. **Private Network Containment:** All compute workloads reside inside private subnets without public IPv4 assignments.

3. **Decoupled Security Layers:** Route tables govern network path availability, while Security Groups enforce independent, stateful firewall rules at the network interface level.
## output:

#### PING 10.20.1.10 (10.20.1.10) 56(84) bytes of data.
--- 10.20.1.10 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3072ms
