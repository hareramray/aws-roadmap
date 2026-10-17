# Tutorial 06.2 — Build a replaceable service behind an ALB

[Domain tutorials](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../06-compute-and-load-balancing.md)

**Mode:** optional paid sandbox deployment; all steps can instead be completed as a labeled design exercise. **Prerequisites:** tested image/template from tutorial 06.1 and two-AZ network. **Result:** observed target replacement and a release/rollback record. Reviewed 2026-10-18.

## 1. Separate traffic components

Write the flow as client → listener → listener rule → target group → target port → process. Use the tutorial service on HTTP port 80 with health path `/health`. This lab contains only synthetic public text; production private data requires suitable authentication and TLS throughout the intended trust boundaries.

Create `tutorial-alb-sg` allowing the chosen client source on the test listener port. Create `tutorial-app-sg` accepting port 80 only from the ALB group. Keep direct public access to the application instances closed. A healthy target should be reachable from the load balancer even if your laptop cannot connect to it directly.

## 2. Create the load balancer and group

Create an internet-facing ALB across the two public AZ subnets. Create an instance target group on port 80 and set HTTP health checks to `/health`, expecting the success status your test server returns. Configure a listener forwarding to that target group.

For a no-domain lab, an HTTP listener restricted to your test source is sufficient for this dummy content. For an optional HTTPS exercise, use a domain you control and a valid certificate in the required Region, attach it to an HTTPS listener and redirect HTTP appropriately. Do not pretend the provided ALB domain is covered by an arbitrary custom certificate.

A target group and a listener are different resources. A target registered in the group can remain unhealthy if the group checks the wrong port, the enabled AZ does not include it, or the service is not ready.

## 3. Create bounded replaceable capacity

Use a dedicated Auto Scaling group with the tested launch template, desired capacity two, minimum two and a small explicit maximum such as three. Place instances in the two private app subnets. Use the baked service image or provide a fully working bootstrap dependency path; do not add a package install requiring internet access to an isolated subnet without planning egress.

Attach the ALB target group and enable the health-check behavior appropriate to the exercise. Set a documented grace period that accommodates actual startup rather than hiding persistent failures. Ensure your image starts the web service and produces `/health` without manual login.

**Expected result:** two targets eventually become healthy and the ALB DNS name returns the release text. Record launch activity, bootstrap time and health transition time. “Desired two” with zero healthy targets is not a successful service.

## 4. Perform a controlled replacement

Terminate one instance belonging only to this dedicated group. Observe target deregistration/unhealthy transition, scaling activity, replacement launch and new target readiness. Repeatedly make a small number of normal requests during the exercise and record errors/timeouts, if any; do not conduct an unbounded load test.

The group should restore desired capacity without manual instance configuration. Measure the interval rather than claiming zero downtime. If both instances share an unavailable startup dependency, replacement can fail even though the template is valid.

Keep state outside replaceable compute. If a user session disappears because it existed only in one instance's memory, sticky sessions may hide but do not solve the underlying failure/recovery problem. Choose an explicit session-state strategy.

## 5. Introduce and repair a health error

On the dedicated target group, temporarily set the health path to `/missing-training-path`. Observe failing health checks and record the reason. Restore `/health` promptly. Do not perform this on a production group or assume all-unhealthy behavior provides safe isolation; load-balancer fallback/fail-open behavior must be understood for the chosen type.

Compare three checks: TCP port open, shallow HTTP readiness, and a deep check that queries every optional downstream service. The first can miss application failure; the last can remove all targets because one optional dependency is slow. Write a readiness rule that reflects the ability to serve the operation you promise.

## 6. Release and rollback through template versions

Create a second dedicated image/configuration whose response is `training release two`. Test it independently, create a new launch-template version, and explicitly select that version for the group. An unpinned “latest” assumption can make rollback ambiguous.

Use an instance refresh with capacity/health settings appropriate to the two-instance lab. Observe old/new targets and record when the new release is healthy. Roll back by selecting the known-good template/image version and replacing affected instances through the supported workflow. A template setting change alone does not rewrite existing disks.

Discuss lifecycle hooks for initialization/draining, scale-in protection for selected work, and how those settings can delay replacement. Every delay needs a timeout and failure procedure. A worker stuck indefinitely in a lifecycle hook can defeat capacity objectives.

## 7. Compare other balancers and transport behavior

For a fictional TCP game protocol, evaluate an NLB: listener protocol, static-address needs, source-IP behavior, target security, idle connection behavior and health must fit the protocol. For a compatible firewall appliance fleet, evaluate GWLB and its routing/encapsulation requirements. Neither is a drop-in ALB path-routing replacement.

Inspect cross-zone settings for the actual type and target group. ALB and NLB defaults differ; verify the setting rather than assuming the two-AZ diagram guarantees equal traffic. Add connection idle timeout, keep-alive, deregistration delay and graceful process termination to the runbook.

For HTTP services using proxy headers, trust only the intended proxy path when deriving client identity. TLS termination at the ALB and encryption to the target are separate decisions. gRPC/WebSocket applications need their supported protocol and connection behavior checked explicitly. Classic Load Balancer belongs in migration recognition, not the default new lab.

## Verify, troubleshoot, and clean up

Pass with healthy targets, observed replacement, a controlled health failure/recovery and a release rollback. Investigate ALB 5xx versus target-generated 5xx separately. Check target reason codes, SG paths, listener/rule configuration, health path, enabled AZs and application logs before scaling up.

**Cleanup/cost:** delete the dedicated Auto Scaling group and its instances, ALB/listeners, target group, templates and unneeded lab images/snapshots. Remove dedicated certificates/records only if created for this lab and unused. Inspect networking and logs separately. ALB, compute, storage, IPv4, transfer and management-path resources can all charge independently.

Sources: [Elastic Load Balancing](https://docs.aws.amazon.com/elasticloadbalancing/), [ALB configuration](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html), [EC2 Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/).
