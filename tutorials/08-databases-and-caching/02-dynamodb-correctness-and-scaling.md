# Tutorial 08.2 — Prove DynamoDB conditional writes and model access patterns

[Domain tutorials](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../08-databases-and-caching.md)

**Mode:** tiny sandbox table with optional design-only advanced branches. **Result:** duplicate/stale-write rejection and a key/index/consistency plan. Reviewed 2026-10-18.

## 1. Create an explicitly scoped table

Choose a unique lab table name. Confirm account/profile/Region before creating it; on-demand is a simple starting capacity choice, not a guarantee of zero cost or unlimited instantaneous scaling.

```powershell
$labProfile = 'roadmap'
$labRegion = 'ap-south-1'
$labTable = 'roadmap-orders-training'
aws sts get-caller-identity --profile $labProfile --region $labRegion
aws dynamodb create-table --table-name $labTable --attribute-definitions AttributeName=orderId,AttributeType=S --key-schema AttributeName=orderId,KeyType=HASH --billing-mode PAY_PER_REQUEST --profile $labProfile --region $labRegion
aws dynamodb wait table-exists --table-name $labTable --profile $labProfile --region $labRegion
```

If the name already belongs to another resource, choose another name rather than deleting it. Tag the dedicated table and document deletion ownership. Do not enable global tables or multiple expensive diagnostic integrations for the first test.

## 2. Insert exactly once

Save `order.json` as UTF-8:

```json
{
  "orderId": {"S": "order-001"},
  "customerId": {"S": "customer-a"},
  "quantity": {"N": "3"},
  "version": {"N": "1"}
}
```

Run the conditional write twice:

```powershell
aws dynamodb put-item --table-name $labTable --item file://order.json --condition-expression 'attribute_not_exists(orderId)' --return-consumed-capacity TOTAL --profile $labProfile --region $labRegion
```

Expected: first succeeds, second returns a conditional-check failure. Treat the second as a business duplicate/conflict according to the request's idempotency design, not a transient error to retry indefinitely. A numeric DynamoDB attribute uses a string representation in this low-level API JSON; that is different from storing it as type `S`.

## 3. Reject a stale update

Save `update.json`, replacing the table name if changed:

```json
{
  "TableName": "roadmap-orders-training",
  "Key": {"orderId": {"S": "order-001"}},
  "UpdateExpression": "SET quantity = :q, #v = :next",
  "ConditionExpression": "#v = :expected",
  "ExpressionAttributeNames": {"#v": "version"},
  "ExpressionAttributeValues": {
    ":q": {"N": "4"}, ":next": {"N": "2"}, ":expected": {"N": "1"}
  },
  "ReturnValues": "ALL_NEW"
}
```

```powershell
aws dynamodb update-item --cli-input-json file://update.json --profile $labProfile --region $labRegion
```

Run it twice. The first should return quantity four/version two. The second should fail because it still expects version one. This implements optimistic concurrency for this item. A separate unconditioned read followed by a write would not provide the same protection.

## 4. Read with intentional consistency

Create a key JSON file containing only `orderId`, then use `get-item` with `--consistent-read` and without it. A strong table read can verify the latest successful write; eventual reads may already show it and need not visibly be stale in a small test. Do not claim an experiment proves there is never propagation delay.

Tables/LSIs offer strong-read options; GSIs and Streams are eventually consistent. Transactions coordinate supported multi-item operations with their own costs/limits; they are not implied by ordinary batch APIs. Batch operations can return unprocessed items that need bounded retry handling.

## 5. Design a customer-history access pattern

The original primary key supports one order lookup. To list orders by customer, design a GSI with `customerId` as partition key and a sortable creation/order identifier as sort key. Define which attributes are projected and why. The application uses `Query` on the index rather than scanning all orders and filtering by customer.

Write five synthetic rows for two customers and predict each query's results. Add a page limit of two; require the client to continue using the returned continuation key until complete. A filter expression is applied after reads and does not turn an expensive scan into an efficient key lookup.

Compare this separate-index design with a composite-key table containing `PK=ORDER#id`/`SK=DETAILS` and `SK=ITEM#id` rows. State the operations helped and complicated by each. A single-table design is a choice, not a universal rule.

## 6. Plan expiry, events and recovery

For an expiring session record, use an expiry attribute in the required numeric epoch-seconds format and enable TTL only after checking the field. TTL deletion is asynchronous: the application must reject an expired session before the item is physically removed. Use synthetic expired data, not an access-control bypass test on real users.

For Streams, define the view needed by a worker and how it handles repeated processing and per-item order. A stream event is not permission to repeat a billing operation. Add failure handling and replay ownership.

Design PITR/on-demand backup, export/import and restore into a **new** table. List the additional recovery work: permissions, application table name, indexes/configuration, alarms and integrations. Deletion protection and backup retention serve different purposes.

## 7. Evaluate capacity and geography

Model 1,000 writes per second evenly distributed versus the same writes to one hot partition key. Aggregate capacity alone does not guarantee the hot key succeeds. Examine item size, consistency, transactions, GSI write amplification, throttling reasons, warm-throughput behavior and configured limits.

Compare provisioned capacity/autoscaling with on-demand from workload predictability and budget. For DAX, define which supported read patterns benefit and how cache consistency affects correctness; strong reads should not be treated as stale-cache reads. Test cache invalidation behavior separately from table correctness.

For global tables, choose MREC or MRSC deliberately. MREC uses asynchronous replication/conflict handling; MRSC has different cross-Region latency/topology and feature restrictions. Check current restrictions for transactions, TTL and LSIs before combining features. Do not create a multi-Region table just to demonstrate this concept; write regional read/write/conflict tests and expected outcomes instead.

## Verify, troubleshoot, and clean up

Pass with duplicate creation rejected, stale update rejected, current item verified and a paginated index design. For unexpected success, inspect the actual condition and table/key; for denial, inspect IAM/resource policy and Region; for throttling, inspect the specific reason and key distribution before blindly retrying.

**Cleanup/cost:** delete only the dedicated table after preserving evidence; inspect explicit backups, exports, streams consumers and logs separately. Requests, storage, indexes, backups, diagnostics and replication can create charges. DynamoDB Local can support early API learning but does not reproduce all managed-service behavior.

Sources: [DynamoDB guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/), [read consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html), [global tables](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/V2globaltables_HowItWorks.html).
