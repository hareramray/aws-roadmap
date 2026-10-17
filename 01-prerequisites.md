# 01 — Prerequisites: the skills underneath AWS

[Roadmap index](README.md) · Next: [Account and tooling](02-account-and-tooling.md)

You do not need to master every prerequisite before touching AWS. You do need enough understanding to distinguish an application bug from an operating-system, network, identity, or data problem. Practice locally where possible.

## 1. Operating systems and Linux

Learn processes, threads, standard input/output/error, exit codes, environment variables, users, groups, permissions, services, packages, filesystems, mount points, and signals. Understand the difference between installed software, a running process, and a service configured to start after reboot.

On Linux, practice `pwd`, `ls`, `cd`, `mkdir`, `cp`, `mv`, `cat`, `less`, `head`, `tail`, `find`, `rg`, `chmod`, `ps`, `top`, `df`, `du`, `free`, `ss`, `systemctl`, and `journalctl`. Learn what each command observes before using commands that modify files or permissions. Understand absolute versus relative paths, symbolic links, ownership, executable bits, and filesystem capacity versus inode exhaustion.

On Windows, learn PowerShell objects and pipelines: `Get-ChildItem`, `Get-Content`, `Select-Object`, `Where-Object`, `Get-Process`, `Get-Service`, `Get-NetTCPConnection`, and `Test-NetConnection`. PowerShell and Bash have different quoting, environment-variable, and line-continuation rules. Do not paste a multiline Bash command into PowerShell unchanged.

**Local exercise:** run a tiny HTTP server, identify its process and listening port, capture its logs, stop it, restart it, and confirm the endpoint changes from reachable to unreachable and back. Explain why a process can be running while its HTTP health check fails.

## 2. Networking

Understand the practical layers: link connectivity, IP addressing, transport protocols, TLS, and application protocols. Learn IPv4/IPv6, private address ranges, CIDR, subnets, gateways, route selection, ports, ephemeral ports, NAT, and stateful connection tracking. Practice calculating the size and boundaries of a CIDR block; AWS-specific reserved addresses are covered in the networking chapter.

Trace a browser request: DNS lookup resolves a name, a route selects the next hop, TCP or another transport establishes communication, TLS authenticates and encrypts, and HTTP carries the request and response. A DNS success does not prove that port 443 is reachable. A successful TCP connection does not prove the certificate or application is correct.

Learn DNS record types A, AAAA, CNAME, MX, TXT, NS, SOA, and PTR; recursive versus authoritative resolution; TTL and caching; public versus private naming. Learn HTTP methods, status codes, headers, cookies, content types, request bodies, connection reuse, caching, proxies, and CORS. CORS is a browser access policy; it is not a replacement for server-side authentication.

**Local exercise:** use `Resolve-DnsName`, `Test-NetConnection`, and `curl.exe` on a public documentation endpoint. Record which test proves DNS resolution, which proves transport reachability, and which shows application behavior. Do not scan networks you do not own or have permission to test.

## 3. Programming and APIs

Choose Python or TypeScript/JavaScript first. Learn variables, data structures, functions, exceptions, modules, dependencies, virtual environments, package-lock files, basic unit tests, and logging. You should be able to read a JSON response, validate input, write a file, call an HTTP API, and handle failure without discarding useful context.

Understand REST-style APIs, pagination, authentication, authorization, request IDs, status codes, timeouts, throttling, and retries. Distinguish a timeout before a request is sent from a timeout after a server may have committed the change. A blind retry can duplicate a side effect; use idempotency keys or conditional state transitions when needed.

Learn to separate configuration from code and secrets from both. Log identifiers and timings rather than passwords, tokens, or complete personal records. Handle credentials through the platform's credential provider chain instead of hardcoding them.

**Local exercise:** write a small API with `GET /health` and `POST /jobs`. Validate an input field, return a useful 4xx error for invalid input, attach a request ID, and add one test for a success and one for failure. Keep the first version local and simple.

## 4. Git and collaboration

Practice `git status`, `git diff`, `git add`, `git commit`, `git log`, branches, merges, pull requests, and resolving a small conflict. Learn `.gitignore`, meaningful commits, reviewable changes, and how to restore a file without deleting unrelated work. A Git commit is a versioned explanation of a change, not just a backup button.

Exclude credential files, `.env` files containing secrets, private keys, database dumps, local state containing sensitive data, and generated dependency directories. If a secret is committed, removing it in the next commit does not revoke it or remove it from history. Rotate/revoke the credential first, assess use, and then remediate history according to repository policy.

**Local exercise:** create a branch, change an API response, review the diff, merge it, and explain the resulting history. Add a README with setup, tests, configuration variables, and limitations.

## 5. SQL and data modeling

Learn tables, rows, columns, primary keys, foreign keys, indexes, joins, filtering, aggregation, transactions, constraints, and query plans. Understand normalization and why an index accelerates some reads but adds write and storage overhead. Practice parameterized queries; string concatenation with user input is not a safe query-building strategy.

Learn ACID at a practical level: atomicity concerns all-or-nothing changes, consistency concerns declared invariants, isolation concerns concurrent work, and durability concerns committed data surviving supported failures. Isolation levels and replication consistency differ by database and configuration.

Then study access-pattern-driven modeling: entity keys, partition keys, sort keys, denormalization, hot keys, secondary indexes, TTL, and conditional writes. NoSQL does not mean that schema and relationships disappear; it changes where they are modeled and enforced.

**Local exercise:** model users, orders, and order items in SQLite or PostgreSQL. Query revenue by customer, inspect an index, and wrap an order plus its items in a transaction. Then describe a key-value representation for “latest orders for a customer.”

## 6. Containers and software packaging

Understand images versus containers, layers, registries, tags versus digests, build context, ports, volumes, environment variables, and process lifecycle. A container shares the host kernel; it is not a full virtual machine. Restarting the same container preserves its writable layer, while removing or replacing the container loses data stored only in that layer. Temporary memory-backed mounts have different lifetimes. [Docker storage](https://docs.docker.com/engine/storage/)

Practice a small Dockerfile, a non-root runtime user where supported, a minimal runtime image, `.dockerignore`, and a health check. Separate build-time secrets from runtime secrets. Pin dependencies and inspect image contents instead of assuming a small image is automatically secure.

**Local exercise:** containerize your API, map a local port, and compare file persistence after restarting the same container versus removing and recreating it. Repeat with a volume. This prepares you for ECS/EKS; Kubernetes is not required for the first cloud lab.

## 7. Distributed-system fundamentals

Learn latency, throughput, concurrency, availability, durability, fault isolation, consistency, replication, partitioning, caching, queueing, backpressure, retries, jitter, circuit breakers, and load shedding. These ideas explain why a design that works with one user can fail under traffic or partial outages.

- **Availability and durability:** an intact copy can be temporarily inaccessible; a responsive service can still lose recent data if its durability design is weak.
- **Horizontal and vertical scaling:** more replicas versus a larger node. Both can be limited by shared state, quotas, locks, or the database.
- **At-least-once delivery:** a message may be processed multiple times. A consumer must tolerate duplicates when the processing contract requires it.
- **Eventual consistency:** replicas may lag. Design user-facing behavior for the documented consistency contract of the actual operation.
- **CAP:** under a network partition, a distributed system faces a tradeoff between availability and strong consistency; it is not a universal instruction to “choose any two” without context.
- **Tail latency:** p95/p99 response times reveal slow requests hidden by averages.

**Paper exercise:** an API charges a payment and then times out. Describe why retrying the entire operation is dangerous and how a stable request identifier plus a durable transaction record changes the design.

## 8. Security and delivery basics

Distinguish authentication from authorization, encryption in transit from at rest, hashing from encryption, and a secret from a nonsecret identifier. Learn least privilege, key rotation, software supply-chain risks, patching, dependency scanning, threat modeling, audit logs, backups, and recovery testing.

Understand build versus deploy versus release. A deployed binary may be hidden behind a feature flag; a release can expose behavior without replacing infrastructure. Learn unit, integration, contract, smoke, load, and recovery tests, and choose tests according to risk.

## Readiness check

- [ ] Explain a request from DNS resolution to database response.
- [ ] Diagnose an unavailable local application using process, port, and log evidence.
- [ ] Write and test a small API or automation script.
- [ ] Use Git without committing secrets or generated credential files.
- [ ] Explain a transaction, an index, and a partition key.
- [ ] Describe why retries, queues, caching, and replicas can introduce new failure modes.
- [ ] Read JSON/YAML and distinguish shell syntax from configuration syntax.

If a check fails, practice that part locally and continue with a smaller AWS exercise. You do not need to postpone all cloud learning until every prerequisite is perfect.

## Primary references

- [Python tutorial](https://docs.python.org/3/tutorial/) — language and standard-library fundamentals.
- [PowerShell documentation](https://learn.microsoft.com/powershell/) — shell behavior and commands.
- [Pro Git](https://git-scm.com/book/en/v2) — version-control concepts and workflows.
- [PostgreSQL tutorial](https://www.postgresql.org/docs/current/tutorial.html) — relational-database exercises.
- [Docker getting started](https://docs.docker.com/get-started/) — container concepts and local practice.
- [Amazon Builders' Library](https://aws.amazon.com/builders-library/) — distributed-system lessons for later study.
