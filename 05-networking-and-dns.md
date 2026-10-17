# 05 — VPC networking, DNS, edge delivery, and hybrid connectivity

[Roadmap index](README.md) · Prerequisites: [network fundamentals](01-prerequisites.md), [cloud foundations](03-cloud-foundations.md), [IAM](04-iam-and-identity.md)

Reviewed: **2026-10-18**. Learn to trace packets in both directions. A network diagram is incomplete unless it includes DNS, routing, filtering, return traffic, and the identities allowed to change those controls.

## Outcomes

Design a multi-AZ VPC, distinguish public and private connectivity, explain IPv4 and IPv6 paths, choose endpoints and inter-network connectivity, configure DNS deliberately, and troubleshoot a failed connection using evidence.

## 1. Addressing and subnet design

A VPC is a Regional network. A subnet belongs to one Availability Zone. Plan nonoverlapping CIDR ranges across VPCs and on-premises systems before connecting them. Address overlap complicates peering, transit routing, private services, and migrations. IP Address Manager (IPAM) can support allocation and visibility at larger scale.

Allocate subnet space for growth, managed-service network interfaces, deployment surges, and failover. A subnet with room for today's servers may have no space for tomorrow's load balancer scaling or container tasks. Learn AWS-reserved addresses and service-specific subnet sizing requirements from current documentation instead of assuming every address is assignable.

Use separate route tables for meaningful routing boundaries. A **public subnet** has a route to an internet gateway. An IPv4 EC2 instance also needs a public IPv4 address or Elastic IP, appropriate routes, and permitted traffic to communicate directly through that gateway. Putting an instance in a public subnet alone does not make its private IPv4 address internet-routable.

A **private subnet** lacks direct internet-gateway routing for the relevant traffic. It may have outbound access through NAT, private access to AWS APIs, or no external route. “Private” does not automatically mean “cannot reach the internet,” nor does it remove the need for authentication.

## 2. Routing and return paths

Learn route destinations, targets, subnet associations, the local route, longest-prefix matching, propagated routes, and blackhole routes. Trace both the forward and return path. Asymmetric routing can break stateful firewalls and inspection appliances even when both destinations have routes.

For conventional zonal public NAT gateways, put the gateway in a public subnet with a path to an internet gateway and route the private subnet's IPv4 internet traffic to it. Design and price the available NAT deployment mode for the selected Region; do not assume one NAT path per VPC meets an AZ failure objective. NAT has ongoing and traffic-related costs, and cross-AZ paths can add dependencies and transfer costs.

Network translation permits responses to outbound connections; it is not an inbound publishing mechanism. To expose an application, normally use a suitable load balancer, API endpoint, or edge service with intentional authentication and filtering.

## 3. IPv6 is a separate design

Dual-stack workloads need IPv4 and IPv6 routes, security rules, DNS records, dependency support, and monitoring. A rule for `0.0.0.0/0` does not cover `::/0`. Do not assume that an IPv4-only inspection control protects an IPv6 path.

For outbound native IPv6 internet communication without allowing internet-initiated connections, use an **egress-only internet gateway** and the relevant IPv6 route. It is stateful and handles IPv6 only. A normal internet gateway can support externally initiated IPv6 communication when routing and resource controls allow it. [Egress-only internet gateways](https://docs.aws.amazon.com/vpc/latest/userguide/egress-only-internet-gateway.html)

IPv6-only workloads reaching IPv4-only destinations can use **DNS64 and NAT64**. Route 53 Resolver can synthesize an IPv6 address for an IPv4-only name, and a NAT gateway translates the traffic. This is different from native IPv6 egress; the claim that NAT gateways can never participate in IPv6 connectivity is incorrect. [DNS64 and NAT64](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-nat64-dns64.html)

## 4. Security groups and network ACLs

| Property | Security group | Network ACL |
| --- | --- | --- |
| Main attachment | Supported resources/network interfaces | Subnet |
| Rules | Allow rules | Allow and deny rules |
| Evaluation | Applicable allows combine | Numbered rules; first matching rule wins |
| Connection tracking | Stateful | Stateless |
| Return traffic | Automatically allowed for tracked permitted connections | Must be permitted explicitly |
| Typical purpose | Workload communication policy | Additional subnet-level guardrails |

Security groups are stateful; network ACLs are stateless. For a custom ACL, account for response traffic and ephemeral ports in the actual client/server flow. Do not copy an arbitrary ephemeral-port range without understanding operating-system and managed-service behavior. [Security groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html), [network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)

Use separate groups for load balancer, application, and database. For example, the application accepts its service port from the load balancer's group, and the database accepts its engine port from the application's group. A group reference identifies eligible source interfaces in supported topologies; it does not copy rules from the referenced group.

These controls are not universal packet filters. In particular, ordinary security groups and NACLs cannot be used to block the Amazon-provided DNS resolver or instance metadata in the same way as arbitrary destination traffic. Use appropriate Resolver DNS Firewall and instance-metadata controls. Host firewalls, service policies, and application authorization remain separate layers.

## 5. Private access to AWS and services

| Option | Best starting use | Design concern |
| --- | --- | --- |
| Gateway VPC endpoint | Supported S3 and DynamoDB access via route tables | Endpoint policy, route association, bucket/table permission |
| Interface VPC endpoint / PrivateLink | Supported service API or private service access through endpoint ENIs | DNS, endpoint security groups, AZ placement, per-endpoint costs |
| VPC peering | Direct VPC-to-VPC network connectivity | Nontransitive routing and address overlap |
| Transit Gateway | Hub connectivity for many VPCs and networks | Route-table segmentation, propagation, attachment charges |
| VPC Lattice | Application-level connectivity and policy across services | Service authentication, associations, protocol and feature support |

An endpoint policy filters access through the endpoint; it does not grant the caller IAM permission or automatically disable public service access. For stronger data-perimeter requirements, coordinate identity policies, resource policies, organization controls, DNS, and routing.

PrivateLink exposes a service without establishing general bidirectional network routing between all participating networks. Peering is not transitive: connecting A to B and B to C does not automatically connect A to C. Transit Gateway can provide centralized connectivity but needs deliberate route isolation; one careless propagated route can defeat intended segmentation.

## 6. DNS and service discovery

Understand the difference between registration, authoritative hosting, and recursive resolution. In Route 53, study public and private hosted zones, delegation, A/AAAA/CNAME records, aliases for supported AWS targets, TTLs, and health-based routing.

An alias is not simply a CNAME with a different spelling; it has AWS-specific target behavior and can support a zone apex for supported targets. Private hosted zones answer within associated networks and resolver paths; they do not create connectivity to their answers.

Learn simple, weighted, latency, failover, geolocation, geoproximity, and multivalue routing as mechanisms with different goals. DNS routing acts when clients resolve names. Caching means a DNS change does not instantly move every existing connection, and low TTLs alone do not guarantee a recovery objective.

For hybrid DNS, use Route 53 Resolver inbound/outbound endpoints and forwarding rules appropriately. Avoid forwarding loops, overlapping private namespaces, and accidental public fallback. Resolver query logging and DNS Firewall add visibility and filtering. AWS Cloud Map provides service discovery capabilities where dynamic service registration fits the workload.

## 7. Edge and global traffic

CloudFront caches and delivers content through edge locations. Study origins, cache behaviors, cache keys, TTLs, invalidations, origin access control, signed URLs/cookies, TLS, compression, logs, and WAF integration. Personalized responses require careful cache-key and caching decisions; caching an authenticated response under a shared key can expose user data.

For private S3 origins, use the appropriate REST origin and origin access control. S3 website endpoints have different behavior and are not interchangeable with a private REST origin.

Global Accelerator provides a global entry point and routes supported traffic to healthy Regional endpoints. It does not cache content like CloudFront. Choose based on protocol, routing, latency, and origin requirements; some systems use both for different traffic.

## 8. Hybrid connectivity and advanced study

Site-to-Site VPN provides encrypted connectivity over network paths such as the internet; study redundant tunnels, BGP, routing, and failover. Direct Connect provides dedicated connectivity options but should not be assumed to encrypt all traffic by default. Add appropriate encryption and redundant paths according to requirements.

Client VPN serves individual remote users. Network Firewall and Gateway Load Balancer support different inspection architectures. Cloud WAN is relevant when managing a larger global network. Study MTU/path-MTU discovery, connection tracking limits, NAT port exhaustion, DNS query limits, and transit routing only after the basic request path is clear.

## 9. Implementation and troubleshooting workflow

Write a connectivity matrix first: source, destination, protocol, port, identity, DNS name, route, and expected result. Then allocate addresses, build subnets, associate route tables, attach gateways/endpoints, set security controls, and enable focused telemetry.

When a connection fails, inspect DNS resolution, resolved address family, route in each direction, gateway state, security groups, NACLs, host firewall, listener, TLS name/certificate, and application response. VPC Flow Logs expose flow metadata, not packet contents. Reachability Analyzer evaluates supported configuration paths; it does not prove that the application process is listening or healthy.

## Lab A — Two-AZ VPC plan and route validation

**Type:** console-guided sandbox lab; no compute or NAT required for the baseline.

1. Choose a nonoverlapping private CIDR and create a dedicated lab VPC.
2. Create public and private subnets in two AZs; record CIDRs and AZ IDs.
3. Create and attach an internet gateway. Associate only the public route table with a default IPv4 route to it.
4. Keep private route tables without internet egress. Create separate empty application/database security groups and document intended communication.
5. Optionally add an S3 gateway endpoint, associating only the chosen private route tables and a lab-scoped endpoint policy.

**Verify:** inspect every subnet association and route target. Explain why no instance can yet receive internet traffic and why the endpoint alone does not authorize S3 access. **Cleanup/cost:** remove the endpoint, custom groups, route tables, subnets, internet gateway attachment/gateway, and VPC in dependency order. Do not add NAT gateways, interface endpoints, paid analysis, or public addresses without estimating their charges.

## Lab B — Packet-path failure exercise

**Type:** local design exercise, or optional extension of [compute lab](06-compute-and-load-balancing.md).

Draw browser → DNS → HTTPS load balancer → application → database. Include return ports, health checks, private DNS, and deployment access. Introduce three faults separately: a wrong target-group port, a missing database group rule, and a private subnet with no required API endpoint or egress.

**Verify:** predict the symptom and the best first evidence source for each fault before testing. For an optional live test, change only your dedicated sandbox resources, restore each rule immediately, and record results. **Cleanup/cost:** restore the baseline and use the compute lab's teardown; log ingestion, analysis tools, and managed network services can incur charges.

## Mastery checklist

- [ ] Trace IPv4 and IPv6 requests and responses without skipping DNS.
- [ ] Explain stateful security groups and stateless NACLs.
- [ ] Design private API access and explain its remaining IAM requirements.
- [ ] Compare peering, transit routing, PrivateLink, and application networking.
- [ ] Diagnose routing, filtering, DNS, TLS, and application failures separately.
- [ ] Identify NAT, endpoint, IPv4, transfer, and logging costs.

Further references: [VPC user guide](https://docs.aws.amazon.com/vpc/latest/userguide/), [Route 53 developer guide](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/), [PrivateLink guide](https://docs.aws.amazon.com/vpc/latest/privatelink/), [CloudFront guide](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/), [Direct Connect guide](https://docs.aws.amazon.com/directconnect/latest/UserGuide/).
