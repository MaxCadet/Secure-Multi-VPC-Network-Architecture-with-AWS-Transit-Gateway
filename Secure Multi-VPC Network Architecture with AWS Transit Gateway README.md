# Secure Multi-VPC Network Architecture with AWS Transit Gateway 

A hands-on AWS networking lab demonstrating enterprise-grade VPC segmentation, centralized routing, and least-privilege network access controls using Transit Gateway. Built and validated entirely via the AWS Console with live traffic tests proving the security boundaries.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    AWS Account (us-east-2)                   │
│                                                             │
│  ┌──────────────┐     ┌──────────────┐     ┌─────────────┐ │
│  │  tgw-prod    │     │   tgw-dev    │     │ tgw-shared  │ │
│  │ 10.0.0.0/16  │     │ 10.1.0.0/16  │     │ 10.2.0.0/16 │ │
│  │              │     │              │     │             │ │
│  │  ec2-prod    │     │  ec2-dev     │     │ ec2-shared  │ │
│  │ 10.0.1.194   │     │ 10.1.1.213   │     │ 10.2.1.57   │ │
│  └──────┬───────┘     └──────┬───────┘     └──────┬──────┘ │
│         │                    │                     │        │
│         └────────────────────┴─────────────────────┘        │
│                              │                              │
│                   ┌──────────┴──────────┐                   │
│                   │   Transit Gateway    │                   │
│                   │     tgw-lab          │                   │
│                   │                     │                   │
│                   │  ┌───────────────┐  │                   │
│                   │  │ tgw-rt-prod   │  │                   │
│                   │  │ 10.2.0.0/16 ──┼──┼──► attach-shared  │
│                   │  └───────────────┘  │                   │
│                   │  ┌───────────────┐  │                   │
│                   │  │  tgw-rt-dev   │  │                   │
│                   │  │ 10.2.0.0/16 ──┼──┼──► attach-shared  │
│                   │  └───────────────┘  │                   │
│                   │  ┌───────────────┐  │                   │
│                   │  │ tgw-rt-shared │  │                   │
│                   │  │ 10.0.0.0/16 ──┼──┼──► attach-prod    │
│                   │  │ 10.1.0.0/16 ──┼──┼──► attach-dev     │
│                   │  └───────────────┘  │                   │
│                   └─────────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

### Traffic Matrix

| Source | Destination | Allowed | Reason |
|--------|-------------|---------|--------|
| Prod | Shared Services | ✅ | `10.2.0.0/16` propagated into `tgw-rt-prod` |
| Dev | Shared Services | ✅ | `10.2.0.0/16` propagated into `tgw-rt-dev` |
| Prod | Dev | ❌ | `10.1.0.0/16` not in `tgw-rt-prod` — dropped at TGW |
| Dev | Prod | ❌ | `10.0.0.0/16` not in `tgw-rt-dev` — dropped at TGW |
| Shared Services | Prod | ✅ | `10.0.0.0/16` propagated into `tgw-rt-shared` |
| Shared Services | Dev | ✅ | `10.1.0.0/16` propagated into `tgw-rt-shared` |

---

## What This Demonstrates

### 1. Transit Gateway as a Network Hub
Unlike VPC Peering (which is point-to-point and non-transitive), Transit Gateway acts as a regional hub that enables transitive routing between any attached VPC, VPN, or Direct Connect connection. A single TGW replaced what would otherwise require N*(N-1)/2 peering connections.

### 2. Route Table-Based Segmentation
The core security pattern: multiple TGW route tables on a single Transit Gateway enforce environment isolation without needing separate physical infrastructure. Prod and Dev share the same TGW but cannot communicate because neither CIDR is propagated into the other's route table. The block occurs at the **TGW routing layer** — not at the security group or NACL level — making it a network-level control that cannot be overridden by instance-level configuration.

### 3. Propagation vs. Static Routes
Route propagation automatically populates TGW route tables when attachments are added. This lab uses propagation exclusively, demonstrating how TGW learns routes dynamically from VPC attachments — mirroring how BGP propagation works in hybrid network environments.

### 4. Shared Services Pattern
The `tgw-shared` VPC acts as a centralized services hub reachable by all environments. In production this pattern hosts NAT Gateways (centralized egress), DNS resolvers, security tooling, and monitoring infrastructure — eliminating the need to deploy these per-VPC and reducing cost and operational overhead.

### 5. Private Connectivity with VPC Endpoints
All EC2 instances are in private subnets with no public IP addresses. AWS Systems Manager Session Manager access is enabled entirely through VPC Interface Endpoints (`ssm`, `ssmmessages`, `ec2messages`), demonstrating zero-trust remote access with no bastion hosts, no open inbound ports, and no internet exposure.

---

## Infrastructure

### VPCs

| Name | CIDR | Purpose |
|------|------|---------|
| tgw-prod | 10.0.0.0/16 | Production workloads |
| tgw-dev | 10.1.0.0/16 | Development workloads (isolated from Prod) |
| tgw-shared | 10.2.0.0/16 | Shared services hub |

### Transit Gateway Route Tables

| Route Table | Associated Attachment | Propagated Routes |
|-------------|----------------------|-------------------|
| tgw-rt-prod | attach-prod | 10.2.0.0/16 (from attach-shared) |
| tgw-rt-dev | attach-dev | 10.2.0.0/16 (from attach-shared) |
| tgw-rt-shared | attach-shared | 10.0.0.0/16 (from attach-prod), 10.1.0.0/16 (from attach-dev) |

### VPC Endpoints (per VPC)

Each VPC has three interface endpoints enabling private SSM access:

- `com.amazonaws.us-east-2.ssm`
- `com.amazonaws.us-east-2.ssmmessages`
- `com.amazonaws.us-east-2.ec2messages`

Private DNS is enabled on all endpoints. DNS support and DNS hostnames are enabled at the VPC level — a required prerequisite for private DNS resolution through interface endpoints.

---

## Proof of Concept — Live Traffic Tests

All tests conducted via SSM Session Manager from EC2 instances in private subnets.

### Test 1: Prod → Shared Services (Expected: SUCCESS)
```
sh-5.2$ ping 10.2.1.57 -c 4
PING 10.2.1.57 (10.2.1.57) 56(84) bytes of data.
64 bytes from 10.2.1.57: icmp_seq=1 ttl=126 time=0.833 ms
64 bytes from 10.2.1.57: icmp_seq=2 ttl=126 time=0.437 ms
64 bytes from 10.2.1.57: icmp_seq=3 ttl=126 time=0.436 ms
64 bytes from 10.2.1.57: icmp_seq=4 ttl=126 time=0.417 ms
4 packets transmitted, 4 received, 0% packet loss
```
✅ Route exists: `tgw-rt-prod` contains `10.2.0.0/16 → attach-shared`

### Test 2: Prod → Dev (Expected: BLOCKED)
```
sh-5.2$ ping 10.1.1.213 -c 4
PING 10.1.1.213 (10.1.1.213) 56(84) bytes of data.
4 packets transmitted, 0 received, 100% packet loss, time 3081ms
```
❌ No route: `10.1.0.0/16` is not propagated into `tgw-rt-prod` — packets dropped at TGW

### Test 3: Dev → Shared Services (Expected: SUCCESS)
```
sh-5.2$ ping 10.2.1.57 -c 4
64 bytes from 10.2.1.57: icmp_seq=1 ttl=126 time=1.65 ms
64 bytes from 10.2.1.57: icmp_seq=2 ttl=126 time=0.443 ms
64 bytes from 10.2.1.57: icmp_seq=3 ttl=126 time=0.451 ms
64 bytes from 10.2.1.57: icmp_seq=4 ttl=126 time=0.421 ms
4 packets transmitted, 4 received, 0% packet loss
```
✅ Route exists: `tgw-rt-dev` contains `10.2.0.0/16 → attach-shared`

### Test 4: Dev → Prod (Expected: BLOCKED)
```
sh-5.2$ ping 10.0.1.194 -c 4
PING 10.0.1.194 (10.0.1.194) 56(84) bytes of data.
4 packets transmitted, 0 received, 100% packet loss, time 3081ms
```
❌ No route: `10.0.0.0/16` is not propagated into `tgw-rt-dev` — packets dropped at TGW

---

## Key Lessons Learned

**Default TGW route table association must be disabled at creation.** If left enabled, AWS auto-associates every new attachment to a single shared route table, making segmentation impossible without rebuilding. This setting cannot be changed after the TGW is created.

**Association and propagation are distinct concepts.** Association determines which route table governs traffic *arriving from* an attachment. Propagation determines which routes are *advertised into* a route table from an attachment. Understanding the direction of each is essential to reasoning about traffic flow.

**VPC route tables and TGW route tables are independent layers.** Both must be configured correctly for end-to-end connectivity. The VPC route table tells the VPC router to send cross-VPC traffic to the TGW. The TGW route table tells the TGW where to forward that traffic next.

**Explicit subnet associations are required in production.** When creating VPCs manually, subnets use the main route table as a fallback but are not explicitly associated. Explicit associations make routing deterministic and auditable — critical for compliance environments.

**VPC Interface Endpoints require DNS prerequisites.** Private DNS on interface endpoints requires both `enableDnsSupport` and `enableDnsHostnames` set to true at the VPC level. These are not enabled by default on manually created VPCs.

**The security boundary lives at the TGW routing layer.** Segmentation enforced via TGW route tables cannot be bypassed by modifying security groups or NACLs on individual instances. This makes it a stronger, infrastructure-level control compared to host-based firewall rules.

---

## Planned Enhancements (v2)

- [ ] Convert full infrastructure to Terraform (modular: `vpc/`, `tgw/`, `endpoints/`, `iam/`)
- [ ] Add Inspection VPC with AWS Network Firewall for east-west traffic inspection
- [ ] Centralized egress via NAT Gateway in Shared Services VPC
- [ ] VPC Flow Logs shipped to CloudWatch and Datadog
- [ ] Multi-account with AWS RAM TGW sharing across Organization member accounts
- [ ] Cross-region TGW peering attachment

---

## Author

**Max Cadet** — AWS Certified Cloud Practitioner | Pursuing AWS Solutions Architect Associate  
GitHub: [@MaxCadet](https://github.com/MaxCadet)
