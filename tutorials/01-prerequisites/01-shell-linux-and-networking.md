# 01 — Trace a local web request with shell and network tools

[Prerequisite tutorials](README.md) · [All tutorials](../README.md)

**Mode:** local only. **Result:** a web server you can start, observe, break, diagnose, and stop. You will distinguish files, processes, listeners, DNS, transport, HTTP, and TLS. Python and PowerShell are used on Windows; a separate Linux practice block is explicitly marked Bash.

## 1. Create a dedicated workspace

Open PowerShell in a directory you own. Create a new empty directory such as `aws-prereq-lab`; do not use a directory containing private documents because the demonstration server serves its current directory.

```powershell
New-Item -ItemType Directory -Path './aws-prereq-lab'
Set-Location './aws-prereq-lab'
'AWS local lab' | Set-Content -LiteralPath './index.html' -Encoding ascii
Get-Location
Get-ChildItem -Force
Get-Content -LiteralPath './index.html'
```

If the directory already exists, inspect it and choose another dedicated name rather than overwriting unknown files. Explain absolute versus relative paths. Copy the file to `index-copy.html`, compare its content, then remove only that copy using `Remove-Item -LiteralPath`.

## 2. Run and inspect a process

Start a loopback-only demonstration server:

```powershell
python -m http.server 8000 --bind 127.0.0.1
```

Keep that terminal open. In a second terminal, run:

```powershell
Get-NetTCPConnection -LocalPort 8000 -State Listen
Get-Process -Name python -ErrorAction SilentlyContinue
Test-NetConnection -ComputerName 127.0.0.1 -Port 8000
curl.exe -i http://127.0.0.1:8000/
curl.exe -i http://127.0.0.1:8000/does-not-exist
```

Expected: a listener, a successful TCP test, HTTP success for the existing page, and HTTP 404 for the absent file. Inspect the request lines printed in the server terminal. A 404 proves that the application answered; it is not a network timeout.

The built-in server is a learning tool, not a hardened production server. Binding to loopback keeps this exercise local. Do not open a firewall or expose it publicly to make the lesson work.

## 3. Cause and diagnose two different failures

Stop the server with Ctrl+C. Repeat the TCP and HTTP checks. Expect connection failure rather than a valid HTTP 404. Restart it and request the missing file again. Explain the difference: one failure occurs before an HTTP response, the other is an application response.

While the server is running, try to start a second server on the same address/port. Observe the bind error. Use the listening socket's owning process ID to determine which process already owns the port. Do not terminate unrelated Python processes merely because their names match.

## 4. Inspect DNS and TLS separately

```powershell
Resolve-DnsName docs.aws.amazon.com
Test-NetConnection -ComputerName docs.aws.amazon.com -Port 443
curl.exe -I https://docs.aws.amazon.com/
```

The DNS result resolves a hostname; it does not prove the endpoint is reachable. The TCP check establishes transport reachability; it does not prove the certificate or HTTP behavior. The HTTPS request also exercises TLS and the application protocol. A corporate proxy or filtered network can affect these checks; record the environment instead of treating all failures as AWS outages.

Read status code and headers. Distinguish `Content-Type`, redirect `Location`, caching headers, and certificate verification. Do not disable TLS verification as a routine fix for an incorrect clock, hostname, trust chain, or intercepting proxy configuration.

## 5. Calculate addresses and routes

Save as `cidr_lab.py`:

```python
from ipaddress import ip_network, ip_address

network = ip_network("10.20.0.0/24")
halves = list(network.subnets(new_prefix=25))
assert len(halves) == 2
assert ip_address("10.20.0.10") in halves[0]
assert ip_address("10.20.0.200") in halves[1]
print(network.num_addresses, [str(part) for part in halves])
```

Expect 256 addresses and `10.20.0.0/25`, `10.20.0.128/25`. This is address arithmetic, not a count of usable AWS subnet addresses; platform reservations are a separate rule. Draw routes for a private subnet, an outbound gateway, and an internet destination. Identify the most specific matching route and the return path.

Repeat the arithmetic with a small IPv6 example and note that IPv6 routing/security must be designed explicitly. Do not describe IPv6 as IPv4 with a longer number and assume NAT behavior is unchanged.

## 6. Linux practice in an existing Linux environment

If you already have an authorized Linux VM, WSL distribution, or container, use these Bash commands. Installation of WSL/VM software is a separate setup decision; it is not required for the PowerShell lesson.

```bash
pwd
ls -la
ps -ef
df -h
free -m
ss -lnt
```

In a dedicated directory, create a text file, inspect its ownership and mode, and change only that file's owner-executable bit with `chmod u+x <your-file>` if appropriate. Explain owner/group/other permissions, executable bits, and why a filename extension does not determine Linux execution permissions.

Where the environment uses systemd, inspect one known service with `systemctl status` and its recent logs with `journalctl -u <service>`. Containers and some WSL setups may not use systemd; do not treat a missing service manager as a broken AWS application. Learn process exit codes, environment inheritance, standard output/error, and graceful versus forced termination.

## 7. Write the diagnostic report

Create a table for each experiment: command, expected layer, observed result, evidence, and conclusion. Explain how a process can exist without listening, a listener can accept TCP while returning HTTP errors, and a valid page can depend on a failing database.

Extend the report with DNS A/AAAA/CNAME/TXT/NS/MX meanings, TTL caching, recursive versus authoritative resolution, HTTP methods/status families, cookies, proxies, and browser CORS. Use examples from observed headers where possible. CORS restrictions are browser behavior; they do not authorize a server-side API call.

Stop the server, verify port 8000 is no longer listening, and remove only your dedicated synthetic files if no longer wanted. No AWS resources or charges are involved. Continue with [Python, Git, SQL, and containers](02-python-git-sql-and-containers.md).

References: [Python HTTP server](https://docs.python.org/3/library/http.server.html), [Python ipaddress](https://docs.python.org/3/library/ipaddress.html), [PowerShell documentation](https://learn.microsoft.com/powershell/).
