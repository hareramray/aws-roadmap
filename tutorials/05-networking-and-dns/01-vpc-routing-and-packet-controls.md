# Tutorial 05.1 — Build a two-AZ VPC and explain every packet path

[Domain tutorials](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../05-networking-and-dns.md)

**Mode:** console-guided sandbox network configuration plus routing exercises. **Result:** a VPC with no required compute/NAT deployment and a complete connection matrix. Reviewed 2026-10-18.

## 1. Allocate nonoverlapping addresses

First verify that `10.42.0.0/16` does not overlap networks you will connect. If it does, select an approved alternative and update every row below. Use two available AZs, named A and B in your notes, and record their actual AZ IDs.

| Subnet name | Example CIDR | AZ | Intended routing |
| --- | --- | --- | --- |
| tutorial-public-a | 10.42.0.0/24 | A | Internet gateway |
| tutorial-public-b | 10.42.1.0/24 | B | Internet gateway |
| tutorial-app-a | 10.42.10.0/24 | A | Private; no baseline internet egress |
| tutorial-app-b | 10.42.11.0/24 | B | Private; no baseline internet egress |

Leave address space for database subnets and growth. Managed services consume network interfaces and addresses too. AWS reserves some subnet addresses; do not budget as if every address is assignable. In a company, add this allocation to IPAM or the authoritative address inventory before creating connected networks.

## 2. Create only the baseline components

In the VPC console, select the intended account/Region. Create `tutorial-network` using the VPC-only option rather than a wizard that silently adds NAT gateways. Enable the DNS settings needed by later endpoint/private-name exercises. Create the four subnets with the chosen AZs.

Create and attach `tutorial-igw`. Create `tutorial-public-routes` and associate the two public subnets. Add `0.0.0.0/0` targeting the internet gateway. Create separate private route tables for app-a and app-b; associate the corresponding subnets and retain only appropriate local routes initially.

**Verify:** inspect each subnet's explicit route-table association. A route table name containing “private” has no technical effect. The association and routes determine behavior. The VPC's main route table is not automatically the one you intended.

## 3. Trace three IPv4 requests

For a hypothetical EC2 instance in public-a, list all requirements for direct internet communication: suitable public IPv4 addressing, route, gateway, SG/NACL permissions, functioning host and application. A public-subnet route does not make a private address internet-routable.

For an app-a instance with no public address and no default route, predict an internet package download: it fails. Design a conventional zonal NAT path without deploying it: app-a default route → NAT gateway in public-a → internet gateway; responses follow the translation state back. Add a separate AZ-B path if the failure objective requires avoiding dependence on AZ A. Evaluate current deployment modes rather than assuming all NAT configurations share that topology.

For app-a to app-b, use the local VPC route. NAT is unnecessary for this internal connection. Add an overlapping on-premises `10.42.10.0/24` network to the design and explain why ordinary routing cannot uniquely represent both intended destinations. Address planning prevents this migration problem more effectively than random route edits.

## 4. Apply longest-prefix and return-path reasoning

Suppose a route table has `0.0.0.0/0` toward NAT and `10.80.0.0/16` toward a transit attachment. A packet to `10.80.1.10` takes the more specific route. Then remove the transit attachment while leaving its route; inspect the resulting blackhole state in a hypothetical change review.

Write the return route from `10.80.1.10` to the app subnet. A forward route alone is not connectivity. If outbound and inbound paths cross different stateful inspection appliances, asymmetric routing may break otherwise valid traffic. Identify route ownership on both sides before troubleshooting filters.

## 5. Build workload security groups

Create dedicated load-balancer, application and database groups without attaching live compute. Document these rules:

- Load balancer accepts the intended client HTTPS source.
- Application accepts TCP 8080 from the load-balancer group.
- Database accepts the chosen engine port from the application group.

For PostgreSQL in this example use TCP 5432. Group references select eligible source interfaces in supported paths; they do not import the referenced group's rules. An application-to-database response is allowed by the security group's connection tracking when the initiating traffic was permitted.

Now create a **local NACL worksheet**, not a restrictive live replacement. Include inbound service ports and outbound client ephemeral ports, with numbered allow/deny rules in first-match order. Reverse directions for an outbound-initiated connection. A NACL is stateless, so permitting one direction does not permit replies automatically. Choose the actual client's ephemeral range from its platform documentation when implementing; there is no single copied range appropriate to every topology.

## 6. Add IPv6 as a separate path

For a design extension, request/associate an appropriate IPv6 allocation according to your environment, allocate supported subnet prefixes, and enable the intended addressing. Write separate IPv6 SG/NACL rules. An IPv4 deny does not close an IPv6 path.

For outbound native IPv6 without internet-initiated connections, draw `::/0` to an egress-only internet gateway. For intentionally public IPv6, evaluate normal internet-gateway routing plus security controls. Do not assume an IPv4 NAT rule governs either path.

For an IPv6-only client reaching an IPv4-only name, enable DNS64 in the design and route the synthetic `64:ff9b::/96` range through a NAT gateway for NAT64. Keep native IPv6 routing distinct. Predict that an IPv6-only application using a literal IPv4 address cannot benefit from DNS64 name synthesis by itself. Verify service/protocol support before implementing an IPv6-only migration.

## 7. Investigate filter exceptions and telemetry

Ordinary security groups/NACLs do not block the Amazon-provided resolver or metadata using the same rules as arbitrary destination traffic. DNS policy belongs in appropriate Resolver DNS Firewall configuration; instance metadata protection uses metadata options and application/host controls. Do not claim a `deny 169.254.169.254` subnet rule establishes IMDS security.

VPC Flow Logs show connection metadata and accept/reject observations, not HTTP payloads. Reachability Analyzer evaluates supported configuration paths, not whether a process is listening. Use them after forming a specific hypothesis; both can have associated charges.

## Verify, troubleshoot, and clean up

Submit a packet matrix for internet→ALB, ALB→app, app→database, app→internet and IPv6-only→IPv4-only. For each give DNS/address, routes, SG/NACL directions and expected result. Change one target port to 8081 and predict a health-check failure rather than guessing that all failures are DNS.

**Troubleshooting order:** identity/Region, DNS/address family, route association, target/gateway status, return route, SG, NACL, host listener and TLS/application. **Cleanup:** delete only dedicated endpoints if added, groups, custom route tables, subnets, gateway attachment/gateway and VPC in dependency order. NAT, endpoints, public addresses, traffic and telemetry can incur charges even without application success.

Sources: [VPC guide](https://docs.aws.amazon.com/vpc/latest/userguide/), [security groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html), [NACLs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html), [DNS64/NAT64](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-nat64-dns64.html).
