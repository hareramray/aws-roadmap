# 12 — Enforce tool permissions and scope agent memory

[AI tutorial index](README.md) · [All tutorials](../README.md)

**Mode:** executable local authorization exercise, followed by an optional AgentCore integration. **Prerequisite:** [runtime tutorial](11-agentcore-runtime.md). **Result:** a tool boundary that rejects another tenant's record even when a model asks for it, plus a session/memory design with explicit retention and revocation.

## 1. Draw three distinct identities

Use the fictional request “show my order status.” The human user authenticates to your application. The runtime uses its workload role to invoke AWS services. The order tool receives the user's authorization context and uses its own supported access path to the backing system.

Do not collapse these into “the agent is authenticated.” A runtime role permitted to call a database API could still read every tenant if application authorization is missing. An OAuth token issued for one audience is not automatically valid for another. Record subject, tenant, scopes/roles, issuer, audience, and expiry at the boundary that verifies them.

## 2. Run an authorization wrapper without a model

Save the following as `tool_boundary.py` and run `python tool_boundary.py`. It deliberately models only already-verified identity; token validation belongs in the HTTP/authentication layer.

```python
from dataclasses import dataclass
import re


@dataclass(frozen=True)
class Principal:
    subject: str
    tenant: str
    scopes: frozenset[str]


ORDERS = {
    "blue-001": {"tenant": "blue", "status": "dispatched"},
    "red-002": {"tenant": "red", "status": "processing"},
}


def lookup_order(arguments, principal):
    if set(arguments) != {"order_id"}:
        raise ValueError("Unexpected or missing tool fields")
    order_id = arguments["order_id"]
    if not isinstance(order_id, str) or not re.fullmatch(r"[a-z]+-[0-9]{3}", order_id):
        raise ValueError("Invalid order identifier")
    if "orders:read" not in principal.scopes:
        raise PermissionError("Read scope required")
    order = ORDERS.get(order_id)
    if order is None or order["tenant"] != principal.tenant:
        raise PermissionError("Record unavailable")
    return {"order_id": order_id, "status": order["status"]}


def main():
    blue = Principal("user-1", "blue", frozenset({"orders:read"}))
    assert lookup_order({"order_id": "blue-001"}, blue)["status"] == "dispatched"
    rejected = 0
    for args in [
        {"order_id": "red-002"},
        {"order_id": "blue-001", "tenant": "red"},
        {"order_id": "../../private"},
    ]:
        try:
            lookup_order(args, blue)
        except (ValueError, PermissionError):
            rejected += 1
    assert rejected == 3
    try:
        lookup_order({"order_id": "blue-001"}, Principal("user-1", "blue", frozenset()))
    except PermissionError:
        print("PASS: cross-tenant, extra-field, malformed-ID, and missing-scope checks")
    else:
        raise AssertionError("Missing-scope request unexpectedly succeeded")


if __name__ == "__main__":
    main()
```

The deliberately similar error for absent and unauthorized records limits unnecessary existence disclosure. Decide the appropriate error contract for your application. Never pass database error details or credentials back to a model as a debugging shortcut.

## 3. Connect the wrapper to an agent tool

Register a tool named `lookup_order` with exactly one string property, `order_id`, and forbid extra properties where your framework/schema supports it. Inside the tool implementation, call the wrapper with the principal from the verified server context. Do not allow the model to supply a replacement principal, tenant, scope, database table name, or SQL statement.

Run the four local cases through the model. A model might ask for `red-002`, but execution must still fail at the wrapper. Record the attempted tool arguments and the authorization decision, avoiding sensitive payloads. Test the wrapper directly as well as through the agent; model variation must not determine whether access is enforced.

For state-changing tools, add operation-specific authorization, a reviewable proposed action, an approval/consent step appropriate to the application, and a durable idempotency key. Separate “prepare a refund” from “execute a refund.” A user accepting an answer is not blanket authorization for arbitrary future tool calls.

## 4. Add AgentCore Gateway and Identity deliberately

In an optional dedicated project, use the current Gateway setup to expose only the supported tool target required for this lookup. Inspect the resulting tool schema, inbound authentication, target invocation role, and outbound credential requirements. Keep an allowlist of targets and operations. Test discovery as an authorized and unauthorized caller before allowing invocation. [AgentCore Gateway](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway.html)

Use the current Identity workflow for required credential providers and user-delegated access. Store provider secrets in the supported credential mechanism, not model prompts or source files. Verify token audience/scopes and consent behavior. Establish how expired/revoked tokens are handled and whether a refresh can outlive the user's intended session. [AgentCore Identity](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/identity.html)

A gateway centralizes connectivity; it does not erase application-level tenant rules. If a legacy tool cannot enforce per-user restrictions, narrow its backend permissions or place an authorization adapter in front of it.

## 5. Design short-term and durable memory

Use a table with `tenant`, `subject`, `conversation_id`, `memory_type`, `created_at`, `expires_at`, and `source_version`. Define whether a memory is a conversation event, user preference, summary, or factual record. A model-generated summary can be wrong; do not promote it to an authoritative account attribute without validation.

For an optional AgentCore Memory resource, choose only the supported strategies you need and a deliberate retention setting. Verify how actor/session identifiers and namespaces constrain write and retrieval operations. Test two users with the same conversation label and confirm no cross-user retrieval. IDs chosen by a client are not proof of ownership. [AgentCore Memory](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory.html)

Write a synthetic preference for user 1, retrieve it in a later authorized session, revoke that user's access, and attempt retrieval again. Delete the memory through the documented workflow and verify relevant retrieval paths no longer return it. Consider derived summaries, caches, backups, and logs when defining deletion requirements.

## 6. Bound execution and handle failures

Enforce maximum tool calls, elapsed time, input size, output size, and concurrent work. Reserve resources before starting costly work when possible. In a distributed deployment, per-process counters are not a global budget. Keep persistent operation state for side effects that can survive worker replacement.

Simulate a tool timeout after the backing system may have completed. Discuss why blindly retrying a write is dangerous. For the read-only lookup, bounded retries may be acceptable; for a payment, query by stable operation ID and reconcile the recorded outcome.

## Verification, troubleshooting, and cleanup

Acceptance requires successful authorized reads, rejected cross-tenant and missing-scope calls, no model-supplied identity override, isolated memory, and bounded tool execution. Diagnose identity failures at the boundary where they occur: inbound user token, runtime AWS role, gateway authorization, or target credentials.

Remove the dedicated gateway/targets, credential providers/secrets, memory resource/events, test identity assignments, runtime changes, and logs according to your resource ledger. Revoke test credentials before forgetting the environment. Keep shared identity infrastructure intact. Local-only execution has no AWS charges; hosted identity/memory/runtime/observability use must be estimated.

**Completion:** demonstrate that malicious instructions can change a model's request but cannot grant it an unauthorized capability.
