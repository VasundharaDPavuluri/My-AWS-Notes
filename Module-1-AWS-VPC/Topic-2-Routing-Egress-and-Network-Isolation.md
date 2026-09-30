# Topic-2: Routing, Egress and Network Isolation

## Overview

A VPC can have well-designed subnets and still be exposed to unnecessary risk or fail to reach the services it depends on.

The difference often comes down to **routing**:

- Where traffic is allowed to go.
- How traffic gets there.
- Which paths are intentionally unavailable.
- How workloads access the internet and AWS services.
- How network paths behave during failures.

---

## 1. Route Tables Define the Traffic Path

A subnet's route table determines where traffic is directed based on the destination CIDR and route target.

Common route targets include:

| Route Target | Purpose |
|---|---|
| Local | Communication within the VPC |
| Internet Gateway (IGW) | Internet connectivity for eligible resources |
| NAT Gateway | Outbound internet access from private subnets |
| VPC Endpoint | Private connectivity to supported AWS services |
| Transit Gateway | Connectivity between multiple VPCs and networks |
| Virtual Private Gateway | VPN connectivity to on-premises networks |

A route table does not simply define whether a subnet is "public" or "private". It defines the traffic paths available to resources inside that subnet.

### Example

```text
                    Internet
                       |
                       |
              Internet Gateway
                       |
             +---------+---------+
             |                   |
        Public Route       Private Route
             |                   |
       Public Subnet         NAT Gateway
                                 |
                          Private Subnet
```

---

## 2. Public vs Private Subnets

A common misconception is that a subnet is public simply because resources inside it have public IP addresses.

A subnet is generally considered **public** when its route table contains a route to an Internet Gateway.

Example:

```text
0.0.0.0/0  →  Internet Gateway
```

A private subnet typically does not have a direct route to the Internet Gateway.

For outbound internet access, the traffic can instead follow:

```text
Private Subnet
      |
      v
 NAT Gateway
      |
      v
Internet Gateway
      |
      v
  Internet
```

This allows private resources to initiate outbound connections without providing them with a direct inbound path from the internet.

---

## 3. Design Egress Based on Workload Needs

Private workloads may still need external connectivity.

Examples include:

- Downloading operating system updates.
- Calling external APIs.
- Accessing package repositories.
- Sending telemetry to external services.
- Accessing AWS services.

A common production pattern is:

```text
Private Application
        |
        v
   NAT Gateway
        |
        v
Internet Gateway
        |
        v
     Internet
```

However, routing all outbound traffic through NAT is not always the best design.

Consider:

- Traffic volume.
- NAT Gateway cost.
- Availability requirements.
- AWS service requirements.
- Data transfer considerations.
- Security requirements.

---

## 4. VPC Endpoints for Private AWS Service Access

When workloads need to communicate with supported AWS services, VPC endpoints can provide private connectivity without sending the traffic through the public internet path.

Example:

```text
Private Application
        |
        v
   VPC Endpoint
        |
        v
    AWS Service
```

Examples of AWS services commonly accessed through VPC endpoints include:

- Amazon S3
- Amazon DynamoDB
- Amazon CloudWatch
- Amazon ECR
- AWS Systems Manager

The exact endpoint type and availability depend on the AWS service.

### Why Use VPC Endpoints?

They can help:

- Keep service traffic within AWS networking paths.
- Reduce dependence on internet egress.
- Improve network isolation.
- Potentially reduce NAT-related traffic and cost.

Endpoint design should still consider service support, DNS configuration, security policies, and cost.

---

## 5. Network Isolation Is an Intentional Routing Decision

Network isolation is not achieved simply by naming a subnet "private".

It comes from the combined design of:

```text
Route Tables
     +
Security Groups
     +
Network ACLs
     +
VPC Endpoints
     +
Connectivity Architecture
```

For example, a database subnet should generally not have a direct route to the public internet.

A typical application architecture may look like:

```text
                  Internet
                     |
                     v
              Internet Gateway
                     |
              Public Subnet
                     |
              Load Balancer
                     |
                     v
             Private App Subnet
                     |
                     v
             Private DB Subnet
```

The application tier can communicate with the database tier only on the required ports and protocols.

---

## 6. Control Application-to-Database Traffic

A production network should not allow unrestricted communication between every subnet.

For example:

```text
Internet
   |
   v
Load Balancer
   |
   v
Application Tier
   |
   | TCP 5432
   v
PostgreSQL Database
```

The database security group should allow the required database port only from the appropriate application security group or network source.

This creates a controlled communication path instead of broad network access.

---

## 7. Multi-AZ Egress Design

NAT Gateway placement should also be considered when designing a highly available VPC.

A simplified architecture might look like:

```text
                 Internet
                    |
              Internet Gateway
                    |
        +-----------+-----------+
        |                       |
       AZ-A                    AZ-B
        |                       |
 Private Subnet            Private Subnet
        |                       |
 NAT Gateway              NAT Gateway
        |                       |
        +---------- Internet --+
```

A single NAT Gateway can become a cross-AZ dependency.

For production workloads, evaluate whether NAT Gateways should be deployed per Availability Zone.

Consider:

- Availability.
- Cross-AZ traffic.
- Data transfer costs.
- Failure scenarios.
- Operational complexity.

There is no single design that fits every workload. The decision should be based on availability requirements, traffic patterns, and cost.

---

## 8. Hybrid and Multi-VPC Routing

Routing becomes more complex when the VPC connects to other networks.

Examples include:

- VPC-to-VPC communication.
- On-premises connectivity.
- Shared services VPCs.
- Centralized inspection VPCs.
- Transit Gateway architectures.

Example:

```text
                 On-Premises
                      |
                     VPN
                      |
               Transit Gateway
                /            \
               /              \
          VPC-A              VPC-B
        Application        Shared Services
```

When adding connectivity, review:

- CIDR overlap.
- Route propagation.
- Route table associations.
- Security group rules.
- Network ACLs.
- DNS resolution.
- Return paths.

A route that allows traffic in one direction does not automatically guarantee a valid return path.

---

## 9. Think About Failure and Operations

Routing should be designed with failure scenarios in mind.

Consider questions such as:

- What happens if a NAT Gateway becomes unavailable?
- Does an application depend on a resource in another Availability Zone?
- Is traffic unnecessarily crossing Availability Zones?
- Can private workloads still reach required AWS services?
- What happens if a VPN or Transit Gateway path fails?
- Is there a valid return route?
- Can the traffic path be observed during an incident?

These questions help move the design from simple connectivity to production-ready networking.

---

## 10. Troubleshooting VPC Routing

When a workload cannot reach another destination, check the complete traffic path.

A useful troubleshooting flow is:

```text
Source
  |
  v
Subnet
  |
  v
Route Table
  |
  v
Target
  |
  v
Security Group
  |
  v
Network ACL
  |
  v
Destination
```

Also verify the return path.

### Useful AWS capabilities

- VPC Flow Logs
- VPC Reachability Analyzer
- Route tables
- Security group rules
- Network ACL rules
- NAT Gateway metrics
- Transit Gateway route tables

Do not troubleshoot only the security group. A connectivity issue can also be caused by routing, DNS, NACLs, missing endpoints, or an incorrect return path.

---

## Example Production VPC Traffic Flow

A simplified production architecture:

```text
                         Internet
                            |
                            v
                   Internet Gateway
                            |
              +-------------+-------------+
              |                           |
         Public Subnet              Public Subnet
              |                           |
        Load Balancer              Load Balancer
              |                           |
              +-------------+-------------+
                            |
                            v
                    Private App Tier
                     /           \
                    /             \
                   v               v
             NAT Gateway      VPC Endpoint
                   |               |
                   v               v
               Internet       AWS Services
                                   
                    |
                    v
              Private DB Tier
```

The important point is not the exact architecture, but the intentional definition of each traffic path.

---

## Common Mistakes

### 1. Treating a Private Subnet as Completely Isolated

A private subnet may still have outbound internet access through a NAT Gateway.

### 2. Sending All AWS Service Traffic Through NAT

Where appropriate, VPC endpoints can provide private connectivity to supported AWS services.

### 3. Using a Single NAT Gateway Without Considering Failure

A single NAT Gateway can create a cross-AZ dependency for workloads in other Availability Zones.

### 4. Ignoring Return Routes

Bidirectional communication requires a valid path in both directions.

### 5. Allowing Broad Network Access

Avoid unrestricted security group and NACL rules when only specific application flows are required.

### 6. Ignoring CIDR Overlap

Overlapping CIDRs can create routing problems when connecting VPCs or on-premises networks.

### 7. Troubleshooting Only Security Groups

Connectivity depends on multiple layers:

```text
Route Table
     ↓
Network Path
     ↓
Security Group
     ↓
NACL
     ↓
Destination
```

---

## Production Checklist

- [ ] Route tables are associated with the correct subnets.
- [ ] Public subnets have intentional Internet Gateway routes.
- [ ] Private subnets do not have unnecessary direct internet routes.
- [ ] NAT Gateway placement has been evaluated for availability and cost.
- [ ] VPC endpoints are used where appropriate.
- [ ] Application-to-database traffic is restricted to required ports.
- [ ] Security groups follow least-privilege principles.
- [ ] Network ACLs are reviewed where applicable.
- [ ] VPC CIDRs do not create connectivity conflicts.
- [ ] Return paths have been validated.
- [ ] Multi-VPC and hybrid routes are documented.
- [ ] VPC Flow Logs are available where required.
- [ ] Connectivity troubleshooting procedures are documented.

---

## Key Takeaway

Routing is more than connecting subnets.

It defines the permitted paths between:

- Workloads.
- AWS services.
- The internet.
- Other VPCs.
- On-premises networks.

A production VPC should make every traffic path deliberate:

```text
What can communicate?
        ↓
Through which path?
        ↓
Using which network controls?
        ↓
What happens if the path fails?
```

### Architect-Level Insight

A strong VPC design does not simply provide connectivity.

It provides **controlled connectivity** — where every important traffic path has a clear purpose, security boundary, operational owner, and failure consideration.

<img width="1254" height="1254" alt="AWS VPC Post-2" src="https://github.com/user-attachments/assets/61c1dc4e-201e-489f-98be-fb7f2c8fae05" />
