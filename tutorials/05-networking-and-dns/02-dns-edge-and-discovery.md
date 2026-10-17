# Tutorial 05.2 — Trace DNS and deliver a private origin through an edge cache

[Domain tutorials](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../05-networking-and-dns.md)

**Mode:** local DNS reasoning with an optional S3/CloudFront sandbox. **Result:** a DNS decision table, cache behavior test and secure-origin design. Reviewed 2026-10-18.

## 1. Separate three DNS responsibilities

Use the fictional domain `portal.example.com`; `example.com` is documentation-only and is not yours to modify. A registrar manages registration/delegation, an authoritative provider hosts records, and a recursive resolver finds answers for clients. Route 53 can participate in hosting and registration, while the VPC Resolver participates in resolution.

On PowerShell, inspect a real public name without changing it:

```powershell
Resolve-DnsName -Name aws.amazon.com -Type A
Resolve-DnsName -Name aws.amazon.com -Type AAAA
```

Record record type, answer and TTL; answers can vary. A successful lookup proves name resolution, not TCP reachability, TLS correctness or application health. Conversely, an HTTP error response usually shows that more than DNS already worked.

## 2. Build a hosted-zone design

Create a local table for `portal.example.com` pointing to a CloudFront distribution, `api.example.com` pointing to an ALB, and `db.internal.example.com` resolving privately. For real implementation use only a domain you control; no purchase is needed for this exercise.

For supported AWS targets, Route 53 aliases can serve the zone apex where a conventional CNAME is unsuitable. A/AAAA represent address records; CNAME refers to another name. Document both IPv4 and IPv6 behavior where enabled. An alias is an AWS-specific mechanism, not a generic replacement for every CNAME.

The private hosted zone needs appropriate VPC association and resolver behavior. DNS does not establish routes or authorize a database connection. **Negative case:** a laptop on the public internet cannot use a private-zone answer merely because the public zone exists. A VPN/hybrid resolver path must be intentionally designed.

## 3. Choose DNS routing deliberately

Write six hypothetical changes and select a strategy: two equivalent endpoints, a canary receiving a small traffic share, the nearest-latency endpoint, primary/standby recovery, a geographic content policy, and several independently healthy address answers. Compare simple, weighted, latency, failover, geolocation/geoproximity and multivalue routing based on the actual requirement.

Now add a client that cached an answer before a failover. Explain why changing the record is not instantaneous connection migration. Existing TCP connections and caches can outlive the intended change. Lower TTLs increase refresh opportunities but do not prove an RTO. Health checks must measure a useful endpoint, not just a server that answers any request.

## 4. Optional private-origin distribution

Create a dedicated private general purpose S3 bucket in the chosen Region, using dummy content only and keeping Block Public Access enabled. Upload `index.html` containing `training release one`. Do not use the S3 website endpoint for this private-origin exercise.

Create a CloudFront distribution with the bucket's REST origin, origin access control (OAC), HTTPS viewer redirect, a default root object of `index.html`, and a deliberately short test cache policy. Apply the generated/scoped bucket policy for that distribution's origin access. Review the distribution ARN restriction rather than accepting a wildcard principal/resource combination.

Use CloudFront's provided domain; it avoids buying a domain or configuring a custom certificate just for the lab. Wait for distribution deployment according to the console state, then request `https://YOUR_DISTRIBUTION_DOMAIN/index.html`. Expected: release one is returned. Direct anonymous access to the S3 object should remain denied.

## 5. Observe cache behavior

Overwrite the object with `training release two`. Request the same CloudFront URL immediately and after the configured TTL. Record the response content and relevant cache headers. Depending on current cache state, an earlier version may remain until expiry or invalidation; this does not contradict S3 strong consistency.

Invalidate only `/index.html` if needed for the experiment, or use versioned asset names such as `app.release-two.js` for immutable deployments. Compare the operational/cost tradeoff. Do not issue repeated broad invalidations as a substitute for understanding cache behavior.

For compression, inspect content encoding and client accept-encoding behavior using supported settings. For TLS, record which hostname the certificate must cover; a DNS alias alone does not make an unrelated certificate valid. Custom-domain certificate placement/validation has service-specific requirements that you must verify before implementing.

## 6. Protect personalized content

Add a hypothetical `GET /profile` response containing the signed-in user's name. If all users share the same cache key and the response is cacheable, one user's data can be served to another. Either do not cache that private response or design a correctly scoped authenticated caching strategy with justified key dimensions.

List which query parameters, headers and cookies affect content, which are forwarded to the origin, and which belong in the cache key. More dimensions can reduce cache effectiveness; fewer can change correctness. Origin-request policy and cache policy serve different purposes.

For controlled downloads, compare signed URLs/cookies with application-issued capabilities. Explain who signs, how expiry works and what happens if a capability leaks. WAF evaluates supported requests according to its integration; it does not replace application authorization. Avoid using genuine private files while testing these controls.

## 7. Add hybrid DNS and discovery on paper

Draw on-premises resolver → Route 53 Resolver inbound endpoint for the AWS private namespace. Draw VPC resolver → outbound endpoint/forwarding rule for a corporate namespace. Use separate fictional names and document the authoritative side to avoid a forwarding loop.

Enable query logging only when needed and priced. DNS Firewall can filter supported resolver requests; it does not inspect arbitrary encrypted application payloads. Cloud Map can register and discover dynamic service endpoints, but its discovery results still require network and workload authorization. Simulate a service instance deregistration and note client-cache implications.

Compare Global Accelerator with the CloudFront design: Accelerator directs supported traffic to healthy Regional endpoints through a global entry, while CloudFront provides content caching/distribution. A TCP game service and static web assets need different choices. Do not add both without distinct requirements.

## Verify and clean up

Pass when private S3 access is denied, CloudFront access works, cache changes are observed, and the personalized-response hazard is explained. For an origin 403, check bucket/OAC policy and origin type before making the bucket public. For TLS errors, compare requested hostname and certificate. For stale content, inspect cache keys, TTL and origin version.

**Cleanup/cost:** disable and delete the dedicated distribution using its supported lifecycle, remove OAC if unused, delete all lab object versions/bucket and optional logs/records. Distribution requests, transfer, invalidations, DNS zones/queries and logs may be billable. Leave unrelated domains and distributions untouched.

Sources: [Route 53 guide](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/), [CloudFront guide](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/), [Cloud Map guide](https://docs.aws.amazon.com/cloud-map/latest/dg/what-is-cloud-map.html).
