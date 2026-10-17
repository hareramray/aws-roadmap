# Tutorial 06.1 — Launch a secure disposable instance and test its lifecycle

[Domain tutorials](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../06-compute-and-load-balancing.md)

**Mode:** optional paid sandbox deployment. **Prerequisites:** approved EC2/role permissions and the dedicated VPC from networking tutorials. **Result:** one managed instance, bootstrap logs and a lifecycle matrix. Reviewed 2026-10-18.

## 1. Make a launch worksheet

Record account, Region, AZ, subnet, maintained Amazon Linux 2023 AMI owner/ID/architecture, instance type, encrypted root-volume size/type, IAM role, security group and teardown time. Choose a small compatible type after checking current pricing and capacity. Arm/Graviton requires an Arm image and compatible packages; an x86 AMI cannot simply be paired with it.

Use the public lab subnet for this first minimal exercise only if you deliberately accept a public IPv4 charge and outbound internet path. Alternatively use a private subnet with all required Systems Manager endpoints or approved egress. Price that alternative: multiple interface endpoints or NAT can dominate a short instance lab.

No inbound SSH/RDP rule is needed. The instance still requires outbound connectivity and DNS for its management agent and bootstrap package sources. A private subnet without either path will not become manageable just because a role is attached.

## 2. Configure identity and metadata

Create or use a dedicated instance role with the appropriate Systems Manager core permissions and no broad administrator policy. Attach it through an instance profile. Verify the chosen AMI includes a supported SSM Agent and that it can reach the required service endpoints.

Set metadata options to require IMDSv2. Leave unnecessary metadata exposure disabled according to the application design. Modern SDKs can retrieve temporary credentials through the provider chain; do not copy access keys into the image.

Use an encrypted EBS root volume and record `DeleteOnTermination`. Do not enable hibernation casually: it has instance/image/storage eligibility and retained-RAM requirements. No key pair is necessary when using only the tested Session Manager route, provided your account's access process supports it.

## 3. Bootstrap a visible service

Use this **Bash user-data script on Amazon Linux 2023**, not PowerShell on your laptop. It needs repository access and contains no secrets:

```bash
#!/bin/bash
set -euxo pipefail
dnf install -y httpd
printf '%s\n' 'training release one' > /var/www/html/index.html
printf '%s\n' 'ok' > /var/www/html/health
systemctl enable --now httpd
```

Launch exactly one instance with tags such as `Project=aws-roadmap` and a lab identifier. Wait for EC2 status checks and management registration. Then use Session Manager to open a shell through your approved operator identity.

**Expected result:** `curl -fsS http://127.0.0.1/health` prints `ok`; `curl -fsS http://127.0.0.1/` prints the release. Inspect `systemctl status httpd` and bootstrap logs such as `/var/log/cloud-init-output.log`. Do not assume EC2 “running” means application bootstrap succeeded.

## 4. Verify identity and access boundaries

From the instance, run its installed CLI/SDK identity check using the runtime credential chain and selected Region. Record role/account, not credentials. Confirm that the role cannot administer IAM or read unrelated data.

From your laptop, an unsolicited inbound SSH request should have no permitted path. You do not need to scan ports; inspecting the dedicated group's absence of inbound management rules plus successful Session Manager administration demonstrates the intended mechanism.

Tokenless metadata requests should be rejected under IMDSv2-required configuration. If validating this inside the instance, request only a harmless metadata field and do not print credential endpoints. A failure of an obsolete metadata client is evidence to update that client, not a reason to broaden the instance role.

## 5. Test reboot and stop/start

Write `training-persistent` to a dummy file on the root EBS volume, record its checksum, and record the instance's current private/public addressing. Reboot through the EC2 console. Reconnect and verify the file and web service remain available.

Then stop and start the disposable instance if your configuration permits it. Record how management availability and public addressing behave. EBS persists across the stopped state, but retained storage and associated resources can continue charging. Avoid hardcoding a transient public address into the application.

Instance-store data, where a selected type provides it, has different lifetime guarantees. Do not create expensive local-storage instances solely for this exercise; instead write a scenario where a worker stores its only result locally and is stopped. Correct the design by checkpointing authoritative output to durable storage.

For termination, explain the difference between deleting an instance and retaining an EBS volume whose termination setting preserves it. Hibernation is another supported lifecycle option with its own requirements; it is not an application backup or a general autoscaling strategy.

## 6. Convert the launch into reproducible configuration

Create a dedicated launch template from the verified choices, explicitly pinning the intended AMI and configuration version. A template records launch settings; it does not update already-running instances when edited.

For the next tutorial's private instances, optionally create a dedicated AMI containing the installed service so startup does not need a package download. Use the console's supported image workflow and account for snapshots. Verify a replacement from the image before trusting it. Do not bake transient secrets, personal SSH keys or logs with sensitive content into images.

Write an Image Builder design with source image, build component, test component, release cadence and retirement rule. The test should verify the service response, metadata requirement and absence of embedded keys. This design exercise teaches image maintenance without requiring an additional pipeline deployment.

## 7. Diagnose a failed launch

If Session Manager cannot connect, check instance status, agent, role/profile, DNS, outbound endpoints, network controls and operator permission. If package installation fails, inspect bootstrap logs and repository reachability. If HTTP fails locally, inspect the process/listener before changing a VPC rule.

If the instance cannot launch, distinguish incompatible architecture, quota, capacity, subnet-address exhaustion and invalid image permissions. A quota increase does not guarantee capacity in one specific AZ. Record the exact launch error and request ID.

## Verify and clean up

Pass with a working local health response, expected runtime role, no public inbound administration, and a completed reboot/stop/terminate data-lifetime table. Label any unperformed lifecycle branch as a documented prediction, not observed evidence.

**Cleanup/cost:** terminate the dedicated instance after the next lab if sharing it; inspect retained EBS volumes, AMI snapshots, public addresses, endpoints/NAT, launch template and logs. Delete only dedicated resources no longer needed. Charges are separate for instance use, storage, IPv4, networking and telemetry.

Sources: [EC2 lifecycle](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html), [IMDS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html), [Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html), [Image Builder](https://docs.aws.amazon.com/imagebuilder/latest/userguide/what-is-image-builder.html).
