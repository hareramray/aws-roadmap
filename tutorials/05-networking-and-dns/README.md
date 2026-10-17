# Networking and DNS tutorials

[All tutorials](../README.md) · [Roadmap chapter](../../05-networking-and-dns.md) · Reviewed 2026-10-18

| Tutorial | Result |
| --- | --- |
| [01 — VPC routing and packet controls](01-vpc-routing-and-packet-controls.md) | Two-AZ subnet plan, IPv4/IPv6 paths and tested rule reasoning |
| [02 — DNS, edge delivery and discovery](02-dns-edge-and-discovery.md) | Private-origin cache exercise and DNS failure matrix |
| [03 — Private services, hybrid networks and diagnostics](03-private-services-and-hybrid.md) | Endpoint/transit/application-access designs and diagnosis evidence |

## Coverage mapping

| Source headings and named subtopics | Tutorials |
| --- | --- |
| Addressing/subnets: CIDR, overlap, IPAM, interface growth, public/private semantics | 01 |
| Routes/return paths: association, longest prefix, propagation, blackholes, NAT, asymmetry | 01, 03 |
| IPv6: dual stack, egress-only gateway, DNS64/NAT64 | 01 |
| SG/NACL: state, allow/deny, ordering, ephemeral ports, group references, metadata/DNS exceptions | 01 |
| Private access: gateway/interface endpoints, policies, PrivateLink, peering, Transit Gateway, VPC Lattice | 03 |
| DNS: registration/delegation, public/private zones, records/alias/TTL, routing, Resolver, DNS Firewall, Cloud Map | 02, 03 |
| Edge: CloudFront origins/cache keys/invalidation/OAC/signed access/TLS/WAF/compression; Global Accelerator | 02 |
| Hybrid/advanced: VPN, Direct Connect, Client VPN, Cloud WAN, Network Firewall, GWLB, MTU, NAT/connection limits | 03 |
| Workflow, VPC lab, packet-failure lab, Flow Logs/Reachability Analyzer | 01, 03 |
| Coverage-map extensions: Verified Access | 03 |
| Mastery and common failures | Verification/negative tests in all |

Optional paid network resources are labeled explicitly. Design-only branches still require completed tables and predicted results, not simply reading a product link.
