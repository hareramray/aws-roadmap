# Design a hybrid route, DNS path, and disconnected-site response

[Domain index](README.md) · [All tutorials](../README.md)

This design tutorial uses a fictional headquarters, factory, and AWS application network. It produces a validated address plan, route/DNS matrix, failure tests, and an edge-platform decision. It does not order circuits, install hardware, or change enterprise routing.

## 1. Validate addressing before choosing a gateway

Use headquarters `10.10.0.0/16`, factory `10.20.0.0/16`, and AWS application VPC `10.30.0.0/16`. Save `check_networks.py`:

```python
from ipaddress import ip_network
from itertools import combinations

networks = {
    "headquarters": ip_network("10.10.0.0/16"),
    "factory": ip_network("10.20.0.0/16"),
    "aws-app": ip_network("10.30.0.0/16"),
}
for (a, left), (b, right) in combinations(networks.items(), 2):
    if left.overlaps(right):
        raise SystemExit(f"OVERLAP: {a} and {b}")
print("PASS: no overlap in declared networks")
assert ip_network("10.30.10.0/24").subnet_of(networks["aws-app"])
print("PASS: proposed application subnet belongs to VPC")
```

Run it, then change the factory to `10.30.0.0/17` and confirm overlap is detected. Restore the original input. This validates only declared ranges; include partner, recovery, container/pod, and future acquisition ranges in a real IPAM process.

Reserve subnet space for load balancers, managed interfaces, rollout surges, and failure capacity. Allocate separate route tables for meaningful segmentation. An unused-looking CIDR may already be reserved elsewhere, so organizational inventory is part of the decision.

## 2. Trace both directions

Create this connectivity matrix:

| Source | Destination | Purpose | Required evidence |
| --- | --- | --- | --- |
| HQ clients | AWS private API | Business requests | DNS, forward/return routes, TLS, application authorization |
| Factory collector | AWS ingestion endpoint | Telemetry upload | Outbound route, endpoint identity, retries/buffering |
| AWS app | HQ directory/DNS | Identity and name resolution | Conditional forwarding, routes, firewall, timeout |
| Security operator | Managed node | Administration | Approved identity, Session Manager/network path, logs |

Write route destinations and next hops for each side. Check longest-prefix matches and propagated routes. A route to AWS on the on-premises router is insufficient if AWS has no return route. Inspect asymmetric routing through stateful appliances and avoid assuming every firewall accepts it.

Define BGP advertisements and filters for dynamic routing, including which prefixes must never propagate between tenants or environments. Use Transit Gateway route tables for deliberate segmentation in a regional hub. Consider Cloud WAN for a larger policy-driven global topology; do not add it solely because there are two networks.

## 3. Choose transport and encryption

Begin with Site-to-Site VPN when its characteristics meet the requirements. Plan both supported tunnels, routing, tunnel monitoring, failover, and backup capacity. Dedicated Direct Connect connectivity may fit sustained bandwidth or network-consistency needs, but includes provisioning lead time, carrier/partner ownership, location diversity, and commercial commitments.

For Direct Connect, document the relevant gateway and virtual-interface architecture using current supported combinations. Learn private/transit/public interface roles rather than treating them as interchangeable. Direct Connect does not encrypt traffic by default. Evaluate application TLS, supported MACsec, or an appropriate VPN-over-Direct-Connect design according to the protection scope.

Two circuits do not provide independent resilience if they share a carrier path, building entry, router, power source, or change process. Write the failure matrix for each shared dependency. Confirm the backup path can handle essential traffic even if it cannot carry every batch transfer.

## 4. Make hybrid DNS explicit

Use fictional private zones `corp.example.invalid` and `aws.example.invalid`. Specify which resolver is authoritative for each and how conditional forwarding works. Route 53 Resolver inbound endpoints allow supported on-premises queries toward AWS; outbound endpoints/rules support queries toward on-premises resolution.

Write test names and expected answers: an AWS private API resolves privately from HQ, an internal corporate directory resolves from the application, and a nonexistent private name does not unexpectedly fall back to a public lookalike. Avoid forwarding loops where each resolver sends the same suffix back to the other.

During a planned DNS test, inspect the actual resolver used, returned address family, TTL, and authoritative source. A correct DNS answer does not create network reachability, and a working IP connection does not prove TLS hostname validation succeeds. Include IPv6 if the actual design supports it; IPv4 rules do not automatically cover IPv6.

## 5. Choose what must execute at the edge

The factory must keep a local safety/control loop working when WAN connectivity is absent. Cloud upload and analytics can pause. Choose a local runtime and persistent bounded buffer for telemetry; reserve cloud services for work that can tolerate network latency or interruption.

Greengrass V2 can fit supported local components and cloud coordination. Define device identities, component versions, updates, rollback, storage limits, and reconnect behavior. EKS Hybrid Nodes attach on-premises/edge machines to an AWS-hosted EKS control plane; EKS Anywhere has a different local cluster/control-plane operating model. ECS Anywhere is another option for supported external container instances. Verify disconnection behavior and prerequisites rather than assuming all “hybrid” products are autonomous offline platforms.

Calculate buffer capacity: 100 devices × 1 KB/message × 1 message/second × 24 hours is 8.64 GB decimal before indexes, metadata, redundancy, and headroom. Set an overflow policy and telemetry freshness signal. On reconnect, throttle backlog replay so it does not overwhelm the cloud consumer.

## 6. Compare locality and VMware options

Outposts provides supported AWS infrastructure at a customer site. Evaluate facility power/cooling, physical access, service link, local dependencies, capacity, and supported disconnected behavior. Local Zones place selected infrastructure nearer certain locations while retaining specific parent-Region dependencies. Wavelength addresses supported carrier-edge placement. CloudFront accelerates delivery and suitable edge execution; it is not a full replacement for local application/database infrastructure.

For VMware workloads, compare native EC2 rehosting, application replatforming, retaining the current platform, and Amazon EVS with VMware Cloud Foundation. Write responsibility for licensing, supported VCF versions, management, networking, patches, backups, and capacity. EVS is an enterprise design branch, not a low-cost beginner deployment.

Existing VMware Cloud on AWS customers must verify current Broadcom/AWS purchasing and support arrangements rather than copying the historical AWS-resold model. Snowball also requires lifecycle review: current commercial-Region support is scheduled to end December 31, 2026, with stated exceptions. Use current supported online/partner transfer options where they fit instead of treating an old Snow tutorial as a new deployment recommendation.

## 7. Rehearse network failure and acceptance

Tabletop these failures: one VPN tunnel down, primary circuit down, corporate DNS unavailable, overlapping partner subnet, expired certificate, and factory WAN loss. For each, record customer effect, observed signal, decision owner, recovery, and data reconciliation.

Use a safe local simulation for the buffer: stop a mock uploader, append numbered synthetic messages to a local file, then replay with a stable message ID and deduplication. Verify all expected IDs reach the destination once as business records even if transport retries occur. Do not unplug an enterprise router to complete a tutorial.

## Verification, cost, and cleanup

Acceptance evidence includes passing CIDR checks, detected overlap, bidirectional route matrix, DNS ownership, encryption choice, buffer calculation, and a failure decision log. A reviewer should identify every dependency needed for factory operation during WAN loss.

Troubleshoot in order: name resolution, address family, route in both directions, tunnel/circuit state, security groups/NACLs/firewalls, MTU, listener/TLS, and application authorization. Flow metadata cannot prove application correctness. Document carrier and on-premises escalation ownership.

Design work has no AWS charges. Live VPN, Transit Gateway, Cloud WAN, Resolver endpoints, Direct Connect, Outposts, VMware, and edge services have distinct recurring and usage costs. Do not order hardware or connectivity for this exercise. Remove only dedicated test connections/endpoints/routes after verifying no consumers remain; preserve shared infrastructure.

References: [Hybrid connectivity guidance](https://docs.aws.amazon.com/whitepapers/latest/hybrid-connectivity/hybrid-connectivity.html), [Direct Connect encryption](https://docs.aws.amazon.com/directconnect/latest/UserGuide/encryption-in-transit.html), [EKS deployment options](https://docs.aws.amazon.com/eks/latest/userguide/eks-deployment-options.html), [Amazon EVS](https://aws.amazon.com/evs/), [AWS Outposts](https://docs.aws.amazon.com/outposts/), [Snowball notice](https://aws.amazon.com/snowball/).
