# Topic-1: Designing a Production VPC

## 1. Overview

A production VPC is more than a network with a few subnets. It is the foundation that determines how workloads communicate, how access is controlled, and how the environment can scale and recover.

The key design decisions should be made before provisioning resources:

- How much IP address space will workloads need?
- How should subnets be distributed across Availability Zones?
- Which resources need internet access?
- Where should network boundaries exist?
- How will the VPC connect to other networks?
- How will the network be operated and expanded over time?

A VPC that works for a small deployment may become difficult to operate if these decisions are postponed.

---

## 2. Plan the VPC CIDR

The VPC CIDR defines the private IPv4 address range available to the network.

For example:

```text
VPC CIDR: 10.0.0.0/16
```

This provides a large address space that can be divided into smaller subnet ranges.

### Key considerations

**Current and future IP demand**

- Estimate the number of workloads and their expected growth.
- Account for services that consume multiple IP addresses, such as container platforms.
- Leave room for additional subnets and future network requirements.

**Connectivity and address overlap**

- Check for overlap with on-premises networks and other VPCs.
- Consider future VPC peering, Transit Gateway, VPN, or Direct Connect connectivity.
- Overlapping CIDR ranges can complicate or prevent routing between networks.

**Subnet allocation**

- Plan subnet ranges before creating resources.
- Keep an address allocation record so teams can avoid accidental reuse.

> A CIDR range is an architectural decision. Changing it later can require significant network redesign.

---

## 3. Design Subnets Around Workload Roles

Subnets are created within a single Availability Zone. A production design commonly separates workloads by their network requirements and access patterns.

### Example subnet layout

```text
VPC: 10.0.0.0/16
|
+-- Availability Zone A
|   |
|   +-- Public Subnet
|   |   +-- Internet-facing Load Balancer
|   |   +-- NAT Gateway, if used
|   |
|   +-- Private Application Subnet
|   |   +-- Application workloads
|   |
|   +-- Private Data Subnet
|       +-- Database workloads
|
+-- Availability Zone B
    |
    +-- Public Subnet
    |   +-- Internet-facing Load Balancer
    |   +-- NAT Gateway, if used
    |
    +-- Private Application Subnet
    |   +-- Application workloads
    |
    +-- Private Data Subnet
        +-- Database workloads
```

### Public subnets

A subnet is considered public when its route table has a route to an Internet Gateway.

Typical resources include:

- Internet-facing load balancers
- NAT Gateways, where required

A resource in a public subnet does not automatically become reachable from the internet. Its own addressing, security group rules, and other controls still matter.

### Private application subnets

These typically host application services that should not accept direct inbound connections from the public internet.

Depending on the design, workloads may use NAT Gateways or VPC endpoints to reach external services.

### Private data subnets

These typically host databases and other stateful services.

Access should be limited to the required application or administrative paths rather than broadly allowing traffic from the VPC.

> Subnet placement should reflect traffic requirements and security boundaries—not simply resource names.

---

## 4. Use Multiple Availability Zones

For production workloads, plan subnets across multiple Availability Zones in the Region.

A multi-AZ layout can support:

- Distribution of application instances
- Load balancing across zones
- Reduced dependency on a single zone
- More options for recovery during a zone-level disruption

However, placing resources in multiple AZs does not automatically make an application highly available.

The application tier, load balancing, data layer, and failover behavior must also be designed for resilience.

### Example

```text
                  Internet
                     |
             Internet Gateway
                     |
          +----------+----------+
          |                     |
        AZ A                  AZ B
          |                     |
    Public Subnet         Public Subnet
          |                     |
    App Subnet             App Subnet
          |                     |
    Data Subnet            Data Subnet
```

The actual resource placement and data replication strategy depend on the workload and the AWS service being used.

---

## 5. Plan Egress and Private Connectivity

Private workloads often need to reach AWS services, software repositories, or external APIs. Decide how that traffic should flow.

### Common connectivity options

| Option | Typical use |
|---|---|
| NAT Gateway | Outbound internet access from private subnets |
| VPC Endpoint | Private access to supported AWS services |
| Site-to-Site VPN | Encrypted connectivity to an external network |
| Direct Connect | Dedicated network connectivity to AWS |
| Transit Gateway | Centralized routing between multiple VPCs and networks |

The right choice depends on traffic patterns, security requirements, availability needs, cost, and operational complexity.

For example, a VPC endpoint can avoid sending supported AWS service traffic through a NAT Gateway. A NAT Gateway is not a general solution for private connectivity to an on-premises network.

---

## 6. Design for Operations and Growth

A production VPC should remain understandable after the original implementation team moves on.

Establish conventions for:

- Resource names and tags
- CIDR and subnet allocation
- Route table ownership
- Security group rules
- Network logging and monitoring
- Change review and deployment
- Documentation and operational ownership

Consider how engineers will troubleshoot connectivity, identify the route a packet takes, and determine which team owns a network component.

A network design that is technically functional but difficult to operate can become a long-term source of risk.

---

## 7. Common Design Mistakes

- Choosing a VPC CIDR without considering future connectivity.
- Using one large subnet for unrelated workload types.
- Assuming that multi-AZ placement alone guarantees application resilience.
- Giving private workloads broad outbound access without reviewing the need.
- Overlooking the IP address demand of containerized workloads.
- Creating overlapping CIDR ranges across networks that may need to communicate.
- Treating route tables, security groups, and network ACLs as interchangeable controls.
- Leaving network ownership and operational procedures undocumented.

---

## 8. Production VPC Design Checklist

Before deploying a production VPC, confirm:

- [ ] The VPC CIDR supports expected growth.
- [ ] CIDR ranges do not conflict with connected networks.
- [ ] Subnets are planned across the required Availability Zones.
- [ ] Subnets reflect workload and traffic requirements.
- [ ] Public and private routing is clearly defined.
- [ ] Egress paths are intentional and documented.
- [ ] Private connectivity requirements are understood.
- [ ] Security boundaries and access rules are reviewed.
- [ ] IP capacity is sufficient for current and planned workloads.
- [ ] Monitoring, ownership, and troubleshooting procedures are defined.

---

## 9. Architect-Level Takeaway

A production VPC is not designed only for the workloads running today.

It should account for address capacity, availability, traffic flows, security boundaries, connectivity, and future change before the first resource is deployed.

**The goal is not simply to create a VPC. It is to create a network foundation that can safely support evolving workloads.**

<img width="1536" height="1024" alt="AWS VPC-Post-1" src="https://github.com/user-attachments/assets/902faaee-ba97-4586-9f4e-f2d53522bfe8" />
