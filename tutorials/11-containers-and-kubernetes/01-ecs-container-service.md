# Build an image and run an ECS service

[Container tutorial index](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../11-containers-and-kubernetes.md)

**Outcome:** Build a nonroot HTTP container, verify it locally, and optionally deploy one Fargate task behind a tightly scoped test security group. AWS steps incur ECR, Fargate, public IPv4, and logging charges. They require an existing sandbox VPC/public subnet with a route to an internet gateway. No database, secret, or customer data belongs in this exercise.

## 1. Create an observable application

Save `app.py`. It returns a release identifier and exposes a health route. SIGTERM starts shutdown from another thread because calling `shutdown()` in the `serve_forever()` thread would deadlock. The application has no persistent state.

```python
import json
import os
import signal
import threading
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer


class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path not in ("/", "/health"):
            self.send_error(404)
            return
        body = json.dumps({"status": "ok", "release": os.getenv("RELEASE", "local")}).encode()
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)


if __name__ == "__main__":
    server = ThreadingHTTPServer(("0.0.0.0", 8080), Handler)
    def stop(signum, frame):
        threading.Thread(target=server.shutdown, daemon=True).start()
    signal.signal(signal.SIGTERM, stop)
    print(json.dumps({"event": "started", "uid": os.getuid()}), flush=True)
    server.serve_forever()
    server.server_close()
    print('{"event":"stopped"}', flush=True)
```

Create a Dockerfile. Select a supported Python slim image and resolve its digest from the official registry before setting `$env:PYTHON_BASE`; the tutorial deliberately supplies no stale digest. This small application needs no build toolchain, so a second build stage would add no useful separation. Add stages when compilation creates artifacts that can be copied into a smaller runtime image.

```dockerfile
ARG PYTHON_BASE
FROM ${PYTHON_BASE}
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app
COPY app.py /app/app.py
USER 10001:10001
EXPOSE 8080
CMD ["python", "/app/app.py"]
```

Create `.dockerignore` containing `.git`, `.env`, `*.pem`, and everything unrelated to the image. The Dockerfile copies only `app.py`; do not use a broad `COPY . .` that accidentally includes credentials.

## 2. Build and verify locally

Docker must already be installed/running. Set `PYTHON_BASE` to the reviewed image reference including `@sha256:...`, then run:

```powershell
if (-not $env:PYTHON_BASE -or $env:PYTHON_BASE -notmatch '@sha256:') { throw 'Set a reviewed base digest' }
docker build --platform linux/amd64 --build-arg "PYTHON_BASE=$env:PYTHON_BASE" -t roadmap-http:v1 .
if ($LASTEXITCODE -ne 0) { throw 'Build failed' }
docker run --detach --name roadmap-http --read-only --cap-drop ALL --security-opt no-new-privileges --publish 127.0.0.1:8080:8080 roadmap-http:v1
if ($LASTEXITCODE -ne 0) { throw 'Start failed' }
Invoke-RestMethod http://127.0.0.1:8080/health
docker exec roadmap-http id
docker stop --time 10 roadmap-http
docker logs roadmap-http
docker rm roadmap-http
```

Expect status `ok`, UID 10001, and both started/stopped logs. A stop that consumes the full timeout suggests signal handling failed. Read-only filesystem errors reveal hidden writes; fix the path or add a narrowly scoped temporary volume, not an unrestricted writable root. Save the image ID, base digest, source commit, architecture, and scan result. Generate an SBOM/signature using your team's approved tooling and retain the immutable artifact identity.

## 3. Optional: publish to ECR

Confirm profile, Region, and account using STS. Create one uniquely named ECR repository with immutable tags and scan configuration appropriate to your account, or use a dedicated precreated lab repository. Record its URI as `$env:LAB_IMAGE_REPO` and its registry hostname as `$env:LAB_REGISTRY`. Authenticate with `aws ecr get-login-password --region $env:AWS_REGION | docker login --username AWS --password-stdin $env:LAB_REGISTRY`; check both AWS and Docker results when automating, since pipeline exit status alone can hide an upstream failure.

Tag and push `roadmap-http:v1` to the repository's `v1` tag, check each exit code, then use ECR `describe-images` to obtain its digest. Deploy `repository-uri@sha256:digest`, recorded as the CloudFormation `Image` parameter. A mutable tag is not sufficient release evidence. Credentials must never be added to image layers or build arguments.

## 4. Deploy an explicit Fargate stack

Save `ecs.yaml`. Use a single reviewed public subnet and your own current public IPv4 `/32` as `TesterCidr`; never open this synthetic unauthenticated endpoint to all addresses. The stack is a learning service without TLS or an ALB, and its direct task IP can change during replacement.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Parameters:
  VpcId: {Type: AWS::EC2::VPC::Id}
  SubnetId: {Type: AWS::EC2::Subnet::Id}
  TesterCidr: {Type: String}
  Image: {Type: String}
Resources:
  Cluster:
    Type: AWS::ECS::Cluster
  Logs:
    Type: AWS::Logs::LogGroup
    Properties: {RetentionInDays: 7}
  ExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal: {Service: ecs-tasks.amazonaws.com}
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - !Sub arn:${AWS::Partition}:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
  SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Synthetic HTTP from one tester
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - {IpProtocol: tcp, FromPort: 8080, ToPort: 8080, CidrIp: !Ref TesterCidr}
  Task:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Cpu: '256'
      Memory: '512'
      NetworkMode: awsvpc
      RequiresCompatibilities: [FARGATE]
      RuntimePlatform: {CpuArchitecture: X86_64, OperatingSystemFamily: LINUX}
      ExecutionRoleArn: !GetAtt ExecutionRole.Arn
      ContainerDefinitions:
        - Name: web
          Image: !Ref Image
          Essential: true
          ReadonlyRootFilesystem: true
          PortMappings: [{ContainerPort: 8080, Protocol: tcp}]
          Environment: [{Name: RELEASE, Value: v1}]
          HealthCheck:
            Command: [CMD, python, -c, "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8080/health', timeout=2)"]
            Interval: 30
            Timeout: 5
            Retries: 3
            StartPeriod: 10
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Ref Logs
              awslogs-region: !Ref AWS::Region
              awslogs-stream-prefix: web
  Service:
    Type: AWS::ECS::Service
    Properties:
      Cluster: !Ref Cluster
      TaskDefinition: !Ref Task
      DesiredCount: 1
      LaunchType: FARGATE
      AvailabilityZoneRebalancing: DISABLED
      DeploymentConfiguration:
        MinimumHealthyPercent: 0
        MaximumPercent: 100
        DeploymentCircuitBreaker: {Enable: true, Rollback: true}
      NetworkConfiguration:
        AwsvpcConfiguration:
          AssignPublicIp: ENABLED
          Subnets: [!Ref SubnetId]
          SecurityGroups: [!Ref SecurityGroup]
Outputs:
  ClusterName: {Value: !Ref Cluster}
  ServiceName: {Value: !GetAtt Service.Name}
```

Validate the template, create/review a CloudFormation change set with these four parameters and IAM capability, then execute it in the sandbox. Use the [CloudFormation tutorial](../12-infrastructure-as-code/01-cloudformation-cdk-sam.md) for that exact workflow. The 0/100 deployment configuration minimizes simultaneous lab tasks but allows downtime. Availability Zone rebalancing is explicitly disabled because AWS does not support it with a 100% deployment maximum; this is also a single-subnet lab. Production availability requires a different measured configuration. See [ECS rebalancing constraints](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-rebalancing.html).

## 5. Observe, break, and recover

Wait for service stabilization with a bounded timeout. Inspect ECS service events, task stopped reasons, container health, and CloudWatch logs. Find the running task's ENI public IP and request `http://that-ip:8080/health` from the allowed tester address. Expect release `v1`. Do not assume a successful task launch proves network reachability.

Create a change set using an intentionally nonexistent image tag. Review that only the task/service changes. Observe image-pull errors and the deployment circuit breaker; preserve the prior digest to restore the working revision if rollback cannot occur, especially on an initial deployment without a completed predecessor. A failed pull can mean an incorrect digest, architecture mismatch, missing execution-role permission, or no network route. An application AWS denial concerns the task role instead; this app has no task role because it calls no AWS APIs.

For production, place tasks in private subnets with planned NAT/endpoints, add an ALB with HTTPS, allow the task port only from the ALB security group, and verify health/drain timing. Add Service Connect for supported service-to-service discovery when useful. Measure CPU, memory, latency, and queue backlog before autoscaling. ECS on EC2 adds host patching/capacity duties; Spot adds interruption handling; finite jobs may fit AWS Batch better than a permanent service. App Runner is closed to new customers and App Mesh is retired; neither is required here. ECS Express Mode can simplify provisioning, but inspect all generated dependencies and costs.

## 6. Teardown and explain

Delete the stack and wait for completion. Verify service/tasks, ENIs, security group, cluster, execution role, and logs are gone. Delete only the dedicated ECR images/repository and local image created for this tutorial. The preexisting VPC/subnet are not lab-owned. If you added ALB, NAT, endpoints, capacity providers, or disks, remove their dedicated resources separately and inspect billing after usage settles.

Explain the difference between an image digest, task revision, running task, and service. Which role pulls an image? Why can a public subnet task still fail to reach ECR? Which deployment setting deliberately accepts downtime?

Official references: [Fargate task parameters](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html), [ECS IAM roles](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/security-iam-roles.html), [CloudFormation ECS service](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-ecs-service.html), [deployment circuit breaker](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-circuit-breaker.html).
