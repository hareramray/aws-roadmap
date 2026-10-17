# Tutorial 05.3 — Design private service access and diagnose hybrid paths

[Domain tutorials](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../05-networking-and-dns.md)

**Mode:** one optional gateway-endpoint configuration plus worked enterprise-network designs. No VPN circuit, appliance or enterprise fleet is required. **Result:** connectivity matrices and a diagnostic runbook. Reviewed 2026-10-18.

## 1. Define four distinct requirements

The portal needs private S3 API access, an internal partner service, connectivity to a corporate network, and user access to a private application. These are not one networking problem. Write a matrix with source, destination, protocol/port, DNS name, identity, route, filter and business owner.

Use fictional networks `10.42.0.0/16` for the application, `10.60.0.0/16` for shared services and `10.80.0.0/16` for the corporate network. Validate nonoverlap before any real deployment. Include a negative requirement: development must not reach the production database.

## 2. Optional S3 gateway endpoint

In the dedicated VPC from tutorial 05.1, create the supported S3 gateway endpoint for your chosen Region. Associate only the intended private route tables. Inspect the resulting service-prefix routes; no EC2 instance has been launched yet, so routing inspection alone is the baseline verification.

Set an endpoint policy scoped to your lab bucket and intended principal/actions using the current service-supported syntax. Keep IAM and bucket permissions separately scoped. The endpoint policy filters requests using that endpoint; it does not grant access by itself.

If you later run a private instance with appropriate credentials, test a read from the lab bucket and a denied request outside scope. Record the identity and request path. A public S3 request outside the endpoint can still succeed if other policies allow it; endpoint creation does not automatically disable public service endpoints or form a complete data perimeter.

## 3. Design an interface endpoint

Choose a supported AWS API needed by a private workload, such as an appropriate Systems Manager endpoint. Use the current service guide to identify all endpoints required for your complete workflow rather than assuming one endpoint covers every agent action.

Plan endpoint network interfaces in the required AZs, security groups permitting HTTPS from intended clients, private DNS, and endpoint policy where supported. Compare per-endpoint/AZ and traffic costs against the existing egress design. The private DNS name resolving to endpoint addresses is one verification; a successful authenticated API call is another.

**Negative case:** DNS resolves but TCP 443 times out. Inspect endpoint SG, client SG/ACL, routing and host behavior. If TCP works but the API denies access, examine identity/resource/endpoint policies. Do not combine those failures under the label “endpoint broken.”

## 4. Compare peering, transit and PrivateLink

For two VPCs requiring selected IP connectivity, draw peering and routes in both directions. Add a third VPC and show why A↔B plus B↔C does not create transitive A↔C reachability. Peering is not an implicit router through another VPC.

For many VPCs, design Transit Gateway route tables separating production, development and shared services. Record attachment association and propagation separately. A development attachment accidentally propagating into a production table can defeat the intended boundary. Test the negative path as deliberately as the successful shared-service path.

For exposing only a partner's supported service, evaluate PrivateLink instead of general network routing. Draw provider service, endpoint consumer, allowed principals, endpoint connection approval and DNS. It does not imply arbitrary bidirectional access between every address in both networks.

For a global organization, extend the routing-policy problem to Cloud WAN: segments, connectivity policy, geographic footprint and operational ownership. The worked deliverable is a route/isolation policy, not a purchased global network.

## 5. Add application networking and user access

Use VPC Lattice for a design where several services across VPCs/accounts need supported application connectivity. Define a service network, a service/target group, associations and authorization policy. Test that an unassociated or unauthorized client is rejected. Lattice target groups are distinct from ELB target groups; do not assume identifiers or configuration are interchangeable.

For employees accessing an internal web application, evaluate Verified Access. Specify identity/device trust providers, application endpoint and access policy. A fictional compliant managed device with an assigned user should pass; an untrusted device or unauthorized user should fail. Record how the application still enforces its own business permissions.

Compare this with Client VPN, which supplies network access for individual clients under configured authentication/routes. Choose from required protocols, device posture, user experience and operational needs. A private user-access product does not replace service-to-service workload identity.

## 6. Design corporate connectivity and inspection

For Site-to-Site VPN, record customer gateway, redundant tunnels, route exchange/static routes and failover. Simulate one tunnel failure and state which path carries traffic. For Direct Connect, record connection/circuit/provider responsibilities, redundant locations/paths and routing. Do not assume dedicated connectivity automatically encrypts all payloads; design appropriate encryption for the requirement.

Add centralized inspection through Network Firewall or compatible appliances behind Gateway Load Balancer where justified. Draw symmetric outbound/return routes and AZ failure handling. An appliance can be healthy while its policy wrongly blocks the application; separate appliance health from rule correctness.

## 7. Work three diagnostic incidents

**Incident A:** private API hostname resolves to a public address. Check private DNS configuration, resolver path and forwarding rules before changing routes. An endpoint exists but the client may not use its namespace.

**Incident B:** small requests work; large responses hang across a hybrid path. Inspect MTU, path-MTU discovery and permitted control traffic, not just the application timeout. Avoid choosing a jumbo-frame value merely because one local link supports it.

**Incident C:** outbound connections fail under load but work at low volume. Investigate NAT connection/port pressure, destination concentration, client connection reuse and connection-tracking limits. Increasing application retries may worsen this symptom.

For each, list a hypothesis, evidence source and reversible next action. Flow Logs provide metadata, Reachability Analyzer reasons about supported configurations, resolver logs show queries, and host/application logs expose listening/TLS behavior. None alone proves the entire application transaction.

## Verify and clean up

Submit the four requirement paths and the denied development→production path. Review every return route, DNS owner, trust policy and paid dependency. A successful ping is not proof of an authenticated database transaction.

**Cleanup/cost:** remove only the dedicated gateway endpoint and any optional interface endpoints/logging created for the lab. No transit attachments, circuits, VPNs or firewalls are required for the design exercise. If optional implementations are added, account for endpoint hours, traffic processing, transfer, tunnels, attachments and telemetry before deployment, then remove them using dependency-aware teardown.

Sources: [PrivateLink](https://docs.aws.amazon.com/vpc/latest/privatelink/), [Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html), [VPC Lattice](https://docs.aws.amazon.com/vpc-lattice/latest/ug/what-is-vpc-lattice.html), [Verified Access](https://docs.aws.amazon.com/verified-access/latest/ug/what-is-verified-access.html), [Direct Connect](https://docs.aws.amazon.com/directconnect/latest/UserGuide/).
