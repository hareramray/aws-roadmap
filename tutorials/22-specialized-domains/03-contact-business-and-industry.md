# Design a customer journey and reconcile its business records

[Domain index](README.md) · [All tutorials](../README.md)

This tutorial combines a contact-center flow with a local integration/reconciliation exercise and industry-specific acceptance checks. It sends no calls, emails, texts, or partner documents. Use synthetic identities and reserved example domains throughout.

## 1. Write a customer-contact flow with failure paths

Design a fictional order-support flow: entry → language selection → authentication or limited unauthenticated path → order lookup → agent queue → resolution/callback alternative. Define hours of operation, maximum wait, unavailable-agent behavior, and a safe response when the order database is slow.

In Amazon Connect, map this into supported flows, queues, routing profiles, agent status, and integration blocks. Keep authentication and object authorization outside a caller-supplied order number: knowing an order ID must not expose another customer's details. A CRM lookup timeout should produce a controlled handoff, not an endless loop.

Prepare this test table before any optional implementation:

| Case | Expected result | Evidence |
| --- | --- | --- |
| Authenticated customer, available agent | Correct queue and authorized order context | Flow path and synthetic contact record |
| No available agent | Defined callback/message alternative | Queue timeout branch |
| Wrong customer/order combination | No protected order details | Authorization denial |
| CRM unavailable | Limited explanation and safe transfer | Timeout/error branch |
| Outside business hours | Honest availability message | Hours-of-operation branch |

A design-only walkthrough is sufficient here. If using a real Connect sandbox, restrict tests to participants you control and check phone-number availability, country requirements, recording settings, and charges first. Release dedicated numbers and remove test recordings according to retention rules afterward.

## 2. Separate communications from application events

SES is an email delivery path, not a complete customer-permission system. For a real implementation, verify sending identity/domain configuration, sandbox/production access requirements, recipient eligibility, bounce/complaint handling, suppression, and reputation. A successful send API response does not prove inbox delivery.

AWS End User Messaging supports relevant messaging channels with their own identity, registration, country, and delivery requirements. Use a local outbox of intended messages in this tutorial instead of contacting recipients. Model states `created`, `submitted`, `delivered`, `failed`, and `suppressed`; do not collapse submission and delivery into one success flag.

For embedded real-time communication, inspect the exact Chime SDK capability. SIP media applications/PSTN Audio entered maintenance in September 2026, and other features have separate notices. Do not assume either that the entire SDK has retired or that every historical example is available. Pinpoint's retiring engagement experience also requires migration assessment before following an older campaign tutorial.

## 3. Reconcile imported business documents locally

Suppose partner invoices arrive twice because an acknowledgment was lost. Use a stable partner/document/version key and reject conflicting duplicates. Save `reconcile.py`:

```python
records = [
    {"partner": "demo", "invoice": "I-1", "version": 1, "amount_minor": 12000},
    {"partner": "demo", "invoice": "I-2", "version": 1, "amount_minor": 8000},
    {"partner": "demo", "invoice": "I-1", "version": 1, "amount_minor": 12000},
    {"partner": "demo", "invoice": "I-3", "version": 1, "amount_minor": -1},
]
accepted, rejected, duplicates = {}, [], []
for record in records:
    key = (record["partner"], record["invoice"], record["version"])
    if record["amount_minor"] < 0:
        rejected.append(record)
    elif key in accepted:
        if accepted[key] != record:
            raise ValueError("same document key has conflicting content")
        duplicates.append(record)
    else:
        accepted[key] = record
assert len(records) == len(accepted) + len(rejected) + len(duplicates)
total = sum(record["amount_minor"] for record in accepted.values())
assert total == 20000
print({"received": len(records), "accepted": len(accepted),
       "rejected": len(rejected), "duplicates": len(duplicates),
       "accepted_amount_minor": total})
```

Expect received 4, accepted 2, rejected 1, duplicates 1, total 20000. This is a normalized business-record model, not a production EDI parser. Change the duplicate I-1 amount and expect a conflict rather than silent overwriting.

## 4. Map the integration to AWS services

Use Transfer Family when the boundary is supported managed file exchange. Define partner authentication, permitted storage prefix, upload completion, file naming, duplicate policy, quarantine, acknowledgment, and retention. Do not start processing a partially uploaded file because it appears in a listing.

Use B2B Data Interchange for supported EDI transformations. Begin with a tiny synthetic document matching a supported standard/version, define its mapping to the normalized schema above, validate the transformation, and compare monetary totals and identifiers. Syntax-valid EDI can still contain a business-invalid amount or unknown partner.

Use AppFlow for supported SaaS data transfers when its connector semantics fit. Check incremental fields, delete handling, credentials, pagination, filtering, schema drift, and rate limits. Use EventBridge or APIs for suitable event-driven integrations, preserving event IDs and a reconciliation path.

Maintain one authoritative system for each field. A CRM may own contact preference, an ERP may own invoice status, and the order application may own fulfillment state. Define conflict resolution rather than allowing two scheduled syncs to overwrite each other indefinitely.

For Amazon Quick/analytics or connected business workflows, govern data access and action permissions separately. A dashboard viewer should not automatically receive authority to modify the ERP. Test row-level access, stale data, source deletion, and action authorization using synthetic identities.

## 5. Apply industry-specific acceptance checks

Use this matrix to extend the same integration discipline:

| Domain | Worked design step | Acceptance/failure test |
| --- | --- | --- |
| Healthcare records / HealthLake | Map a synthetic patient/resource identifier and supported FHIR representation | Reject invalid references and unauthorized record access |
| Medical imaging / HealthImaging | Inventory synthetic DICOM metadata and an image identity/version | Verify ingest/read behavior and metadata integrity without real patient data |
| Genomics / supported HealthOmics workflows | Version workflow, reference data, input checksum and output provenance | Rerun a tiny supported sample and compare defined scientific outputs |
| Supply chain / Connect Decisions | Normalize fictional product/location IDs and a stock event | Detect duplicate shipment, stale inventory and unit mismatch |
| Financial services | Link transaction IDs, source positions and reconciliation totals | Demonstrate balanced totals and visible unresolved exceptions |
| SAP/mainframe | Inventory interfaces, transaction/batch windows and licensing constraints | Validate one synthetic business cycle and rollback boundary |
| Manufacturing/automotive | Model asset/device identity, units, safety boundary and reconnect | Reject impossible units and preserve local safe operation |
| Telecom | Define session identity, latency, network/provider ownership and charging events | Reconcile duplicate session events and a failed provider handoff |

HealthLake, HealthImaging, and HealthOmics have distinct data and workflow models. Check exact capability availability; HealthOmics variant/annotation stores have separate lifecycle notices. A supported or eligible cloud service does not make the complete application compliant. Record organizational requirements, access purpose, provenance, retention, and specialist review before real sensitive data is introduced.

Amazon Connect Decisions is described by AWS as the next generation of AWS Supply Chain. Use current product naming and documentation, and evaluate forecasting/decision quality against known outcomes. An attractive forecast chart is not proof that product identifiers, lead times, or inventory data are correct.

For SAP and mainframe modernization, use current AWS Transform/partner paths and verify the exact feature lifecycle. A converted program needs transaction, numerical, encoding, batch-window, and integration regression tests. Preserve a source baseline and keep migration completion separate from successful business modernization.

## 6. Build an operational contract

For each connector, document owner, schedule, credentials, schema, retry budget, deduplication key, dead-letter/quarantine location, freshness target, and reconciliation query. Alert on missing files and stale watermarks, not only explicit API errors. A silent source that sends nothing can produce no error while the business data is obsolete.

Define deletion across source, transformed records, exports, dashboards, recordings, caches, and backups. Record which copies cannot be immediately erased because of legitimate retention requirements and how access is restricted. Use data minimization before populating test systems.

## Verification, troubleshooting, cost, and cleanup

Deliver the contact test table, communication state machine, reconciliation output, conflicting-duplicate failure, connector mapping, and at least one industry-specific synthetic acceptance test. Clearly label designed behavior versus observed live behavior.

For missing records, check partner identity, upload completion, filters, pagination, connector cursor, schema mapping, target permissions, and quarantine. For incorrect totals, inspect duplicates, versions, currency/unit conversion, time zone, and cancellation semantics. For calls/messages, distinguish flow submission, provider acceptance, final delivery, and customer consent status.

The local baseline has no AWS charges. Optional Connect, phone numbers, calls, messages, recordings, connectors, transformations, industry data stores, and analytics can create recurring or usage charges. Stop schedules and outbound flows first, remove dedicated credentials/resources, delete synthetic records and recordings where appropriate, and release only tutorial-owned numbers or subscriptions. Never send test notifications to real customers.

References: [Amazon Connect](https://docs.aws.amazon.com/connect/), [SES](https://docs.aws.amazon.com/ses/), [AWS End User Messaging](https://aws.amazon.com/end-user-messaging/), [Chime SDK availability](https://docs.aws.amazon.com/chime-sdk/latest/ag/sip-applications-maintenance-mode.html), [B2B Data Interchange](https://docs.aws.amazon.com/b2bi/), [AWS healthcare](https://aws.amazon.com/health/), [Connect Decisions](https://aws.amazon.com/products/connect/decisions/).
