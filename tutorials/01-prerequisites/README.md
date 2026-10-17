# Prerequisite tutorials

[All tutorials](../README.md) · [Original roadmap chapter](../../01-prerequisites.md)

Complete these locally before provisioning larger AWS systems. Each tutorial produces evidence you can reuse while troubleshooting cloud workloads.

| Order | Tutorial | Outcome |
|---|---|---|
| 1 | [Shell, Linux, and networking](01-shell-linux-and-networking.md) | Trace DNS, ports, HTTP, TLS, and process failures |
| 2 | [Python, Git, SQL, and containers](02-python-git-sql-and-containers.md) | Implement and version a transactional example, then package it |
| 3 | [Distributed systems, security, and testing](03-distributed-systems-security-and-testing.md) | Demonstrate idempotency, retry/caching risks, and suitable tests |

## Coverage mapping

| Original topic | Worked tutorial |
|---|---|
| Operating systems, Linux, PowerShell, files, permissions, processes, services, logs | 01 |
| IPv4/IPv6, CIDR, ports, routing, DNS, HTTP, TLS, proxies, CORS | 01 |
| Python, APIs, JSON, exceptions, configuration, pagination/retries | 02–03 and [CLI/SDK operations](../02-account-and-tooling/02-cli-sdk-and-resource-lifecycle.md) |
| Git, branches, review, conflicts, secret handling | 02 |
| SQL, keys, indexes, joins, transactions, parameterized queries | 02 |
| Images, containers, layers, volumes, non-root packaging | 02 |
| Scaling, availability/durability, consistency, CAP, latency, queues, caches, idempotency | 03 |
| Authentication/authorization, encryption, secure configuration, test types, delivery basics | 02–03 and the security/delivery subject folders |

**Evidence gate:** explain the failing layer of a local request, show a rolled-back invalid transaction, and demonstrate a duplicate request producing only one logical result.
