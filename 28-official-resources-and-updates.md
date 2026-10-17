# 28 — Official resources and an update process

[Roadmap index](README.md)

**Review baseline:** 18 October 2026. This repository provides a learning structure, while official documentation supplies current operational details. Before implementation, verify availability, lifecycle, permissions, pricing, quotas, runtime support, and region-specific constraints for your actual account.

## Start with the right source

| Need | Primary source | How to use it |
|---|---|---|
| Find a service guide | [AWS Documentation](https://docs.aws.amazon.com/) | Open the developer/user guide, then security, quotas, pricing, and troubleshooting sections |
| Survey service families | [Overview of AWS](https://docs.aws.amazon.com/whitepapers/latest/aws-overview/introduction.html) | Discover categories and terminology; check lifecycle separately |
| Inspect API/CLI parameters | [AWS CLI reference](https://docs.aws.amazon.com/cli/latest/reference/) | Verify exact options, pagination, return shape, examples, and service behavior |
| Learn CLI setup | [AWS CLI user guide](https://docs.aws.amazon.com/cli/latest/userguide/) | Installation, authentication, configuration precedence, output and troubleshooting |
| Use Python SDK | [Boto3 documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html) | Credential providers, clients, paginators, waiters, errors, and supported options |
| Design a workload | [Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html) | Review all pillars and document evidence and risks |
| Study design examples | [AWS Architecture Center](https://aws.amazon.com/architecture/) | Treat patterns as inputs to decisions, not automatic fits |
| Learn distributed systems | [Amazon Builders' Library](https://aws.amazon.com/builders-library/) | Study timeouts, retries, overload, deployment, and operational patterns |
| Find workshops | [AWS Workshops](https://workshops.aws/) | Check age, service status, prerequisites, cost, and cleanup before starting |
| Structured learning | [AWS Skill Builder](https://skillbuilder.aws/) | Follow role-specific training; inspect access/subscription requirements |
| Check certification scope | [Certification exam guides](https://docs.aws.amazon.com/aws-certification/latest/examguides/) | Match exact code, language, and current lifecycle |
| Estimate cost | [AWS Pricing Calculator](https://calculator.aws/) | Model actual configuration, region, usage, and transfer assumptions |
| Check account offers | [Free Tier FAQ](https://aws.amazon.com/free/free-tier-faqs/) | Read eligibility, plans, credit rules, and account-change consequences |
| Track changes | [What's New with AWS](https://aws.amazon.com/new/) | Filter for your services/regions and follow links to updated docs |
| Investigate a current service event | [AWS Health Dashboard](https://health.aws.amazon.com/health/status) | Correlate relevant public events; account-specific visibility may differ |
| Discover endpoints/quotas | [AWS General Reference](https://docs.aws.amazon.com/general/latest/gr/Welcome.html) | Verify scope and follow service-specific quota documentation |

## Service lifecycle is part of architecture

An accessible documentation page does not prove a service is accepting new customers. An older workshop can remain online after a service enters maintenance or shutdown. Inspect the current lifecycle lists and the service's own notice:

- [Services in maintenance](https://docs.aws.amazon.com/general/latest/gr/maintenance_services.html).
- [Services in sunset](https://docs.aws.amazon.com/general/latest/gr/sunset_services.html).
- [Services in full shutdown](https://docs.aws.amazon.com/general/latest/gr/full_shutdown_services.html).

These categories can affect onboarding, new features, support, migration deadlines, or resource availability. Read the exact notice: a restricted feature, isolated-region variant, or legacy edition does not necessarily mean the entire service is retired. When sources appear inconsistent, prefer the specific current notice for the exact feature/account/region and resolve the ambiguity before building.

Use [the service map](30-service-coverage-map.md) to locate the learning chapter, then check the product status. Keep historical services in migration notes when relevant, not as default choices for new beginner projects.

## Source map by curriculum area

| Area | Entry points |
|---|---|
| IAM and security | [IAM](https://docs.aws.amazon.com/iam/), [IAM Identity Center](https://docs.aws.amazon.com/singlesignon/), [KMS](https://docs.aws.amazon.com/kms/), [Security documentation](https://docs.aws.amazon.com/security/) |
| Networking | [VPC](https://docs.aws.amazon.com/vpc/), [Route 53](https://docs.aws.amazon.com/route53/), [CloudFront](https://docs.aws.amazon.com/cloudfront/), [Elastic Load Balancing](https://docs.aws.amazon.com/elasticloadbalancing/) |
| Compute/storage | [EC2](https://docs.aws.amazon.com/ec2/), [S3](https://docs.aws.amazon.com/s3/), [EFS](https://docs.aws.amazon.com/efs/), [AWS Backup](https://docs.aws.amazon.com/aws-backup/) |
| Databases | [RDS](https://docs.aws.amazon.com/rds/), [DynamoDB](https://docs.aws.amazon.com/dynamodb/), [ElastiCache](https://docs.aws.amazon.com/elasticache/) |
| APIs/events | [Lambda](https://docs.aws.amazon.com/lambda/), [API Gateway](https://docs.aws.amazon.com/apigateway/), [SQS](https://docs.aws.amazon.com/sqs/), [EventBridge](https://docs.aws.amazon.com/eventbridge/), [Step Functions](https://docs.aws.amazon.com/step-functions/) |
| Containers | [ECS](https://docs.aws.amazon.com/ecs/), [EKS](https://docs.aws.amazon.com/eks/), [Kubernetes](https://kubernetes.io/docs/home/) |
| Infrastructure | [CloudFormation](https://docs.aws.amazon.com/cloudformation/), [AWS CDK](https://docs.aws.amazon.com/cdk/), [Terraform](https://developer.hashicorp.com/terraform/docs), [OpenTofu](https://opentofu.org/docs/) |
| Data/analytics | [Glue](https://docs.aws.amazon.com/glue/), [Athena](https://docs.aws.amazon.com/athena/), [Redshift](https://docs.aws.amazon.com/redshift/), [Lake Formation](https://docs.aws.amazon.com/lake-formation/) |
| AI/ML | [Bedrock](https://docs.aws.amazon.com/bedrock/), [SageMaker AI](https://docs.aws.amazon.com/sagemaker/) |
| Operations/governance | [CloudWatch](https://docs.aws.amazon.com/cloudwatch/), [CloudTrail](https://docs.aws.amazon.com/cloudtrail/), [Config](https://docs.aws.amazon.com/config/), [Organizations](https://docs.aws.amazon.com/organizations/), [Control Tower](https://docs.aws.amazon.com/controltower/) |
| Migration | [DMS](https://docs.aws.amazon.com/dms/), [DataSync](https://docs.aws.amazon.com/datasync/), [AWS Transform](https://aws.amazon.com/transform/) |

## How to read a service guide efficiently

1. Read the overview and supported use cases. State the problem in your own words.
2. Read the service's security chapter: authentication, authorization, network access, encryption, logging, and shared responsibility.
3. Inspect its resource model and API lifecycle: create, update, read, list, delete, backup, restore.
4. Read quotas and regional/feature restrictions before designing scale.
5. Read consistency, delivery, failover, and recovery semantics. Record the exceptions.
6. Check pricing dimensions and an estimate using realistic traffic/data.
7. Follow one small getting-started example and identify every created resource.
8. Review troubleshooting, monitoring, and cleanup before expanding the example.

Search results are discovery aids. Open the primary page, check the exact service/feature, and follow the linked reference for operational details. A blog announcement may describe launch behavior; the maintained guide may contain later changes.

## Keep an evidence ledger

For a changing statement, record topic, claim, source URL, review date, applicable region/account/version, and next review trigger. Good triggers include a runtime deprecation notice, an exam-code update, a new architecture requirement, a pricing change, or a service lifecycle announcement.

Example: “Our deployment relies on this runtime/version in this region; reviewed on this date; recheck before the next release and when the runtime deprecation notification arrives.” This is more useful than marking an entire repository “always up to date.”

## Monthly maintenance routine

Review announcements for services you actually use. Recheck lifecycle pages, runtimes, Kubernetes versions, database engine support, SDK/CLI releases, and security advisories. Update one relevant lab, then rerun only the checks affected by the change. Inspect costs and clean up forgotten experiments. Revisit the certification chapter before any exam purchase.

Keep a short change log with the old assumption, new behavior, source, and affected files/projects. Do not rewrite working architecture for every new launch; require a clear improvement in requirements, risk, cost, or operating effort.

## Learning resources to use critically

Community articles, videos, and courses can explain concepts well, but verify commands, account security, prices, limits, and lifecycle against primary documentation. Reject examples that embed credentials, require broad public access without a reason, omit teardown, or claim an expensive architecture is universally free.

Use AI assistants for explanations, practice questions, and draft code, then verify service claims and test generated configurations in a small authorized environment. Never paste real secrets or sensitive business data into learning tools.
