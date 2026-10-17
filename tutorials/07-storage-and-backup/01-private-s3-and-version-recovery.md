# Tutorial 07.1 — Recover private objects and design their storage lifecycle

[Domain tutorials](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../07-storage-and-backup.md)

**Mode:** small paid-usage sandbox lab plus storage-model design. **Result:** version recovery and an access/lifecycle worksheet. Use fabricated text only. Reviewed 2026-10-18.

## 1. Create the isolated bucket

Choose your account and Region explicitly. In S3 create a uniquely named general purpose bucket, enable versioning, retain Block Public Access and use policy-based ownership with ACLs disabled where appropriate. Select S3-managed encryption for the baseline so KMS permissions do not obscure the first access test. Do not enable Object Lock.

Record bucket name, Region, ARN and owner. Create local `notes.txt` with `version-one`, then upload it as `training/notes.txt`. Record size, checksum where exposed, version ID, storage class and metadata. A slash in the key is a naming convention; it is not a filesystem directory that must exist separately.

## 2. Read, overwrite and list

Download the current object and compare its content. Change the local content to `version-two`, upload using the same key, and immediately read/list through S3. The current value should reflect the successful overwrite. Strong S3 consistency does not create transactions across several keys or instantly invalidate a separate CDN cache.

Use the versions view to see both versions. Request/download the older version explicitly and verify its content remains `version-one`. If using the CLI, pass the intended profile and Region on every command and use the returned version ID rather than inventing one.

HEAD retrieves metadata without the body; GET retrieves data; LIST enumerates keys with pagination. A large listing requires continuation handling. COPY can create another object/version; it is not a rename transaction across arbitrary objects. For large data, range reads retrieve selected bytes, but the application must still interpret the object's format correctly.

## 3. Delete and restore intentionally

Delete `training/notes.txt` without selecting an individual version. Inspect the delete marker and the retained earlier versions. The current ordinary read should no longer return the object as before, while a permitted version-specific read can still retrieve retained data.

Restore by removing the relevant delete marker or by copying the desired old version to become current, using the console's supported workflow. State which operation you performed and the resulting current version. Permanently deleting a particular version removes that recovery option.

**Negative test:** attempt anonymous access to the direct object URL; it should fail. Then use the restricted reviewer role from the IAM tutorial to read `training/notes.txt` and deny `private/notes.txt`. Different errors such as 403/404 can depend on permissions, so record expected authorization behavior rather than treating every status as an existence oracle.

## 4. Verify integrity and multipart behavior

Calculate a local file hash with `Get-FileHash` and compare a downloaded copy. Do not assume an ETag is always an MD5 checksum; multipart and encryption behavior can differ. Use the supported checksum mechanism when verifying larger transfers.

Create a design for a 2 GB synthetic archive: multipart upload, retry only failed parts, complete only after successful part verification, and abort abandoned uploads. A lifecycle cleanup for incomplete multipart uploads avoids indefinitely retaining unused parts. You do not need to generate/upload 2 GB to learn the workflow.

Store tags such as `DataClass=Training` separately from custom metadata in your design. Tags can drive lifecycle/access decisions; control who may alter authorization-sensitive tags. Metadata changes may involve copying/replacing object metadata rather than editing a local file header in place.

## 5. Design secure delegation and delivery

Issue a short-lived presigned GET for the dummy object through a permitted identity using a supported console/SDK workflow. Anyone possessing the URL can potentially use it during its effective validity, so avoid putting it into public notes. A temporary signing session can shorten the effective lifetime.

For browser uploads, define an allowed origin/method in CORS only if needed. CORS controls browser behavior; it does not grant object access or prevent nonbrowser calls that are otherwise authorized. Test access with the intended user identity separately.

For edge delivery, compare a private S3 REST origin plus CloudFront OAC against an S3 website endpoint. Use the private REST origin for this design. Define the distribution-scoped bucket permission and leave public access blocked. TLS, cache keys and authenticated responses require the DNS/edge tutorial's checks.

For encryption, compare SSE-S3, SSE-KMS, supported dual-layer choices and client-side encryption by threat/ownership requirement. SSE-KMS adds key-policy and recovery dependencies; changing bucket default encryption does not automatically rewrite all old objects. Client-side encryption moves key and decryption responsibilities into your application.

## 6. Build a lifecycle policy from access evidence

Use three fictional datasets: frequently read thumbnails, unpredictable-access original documents, and retained audit archives with a tolerated retrieval delay. Compare frequent-access classes, Intelligent-Tiering and archive classes accordingly. Record minimum-duration, retrieval and small-object costs from current pricing before choosing.

For a disposable lab, draft rather than accelerate archive transitions. Define current-version expiration, noncurrent-version retention and incomplete multipart cleanup separately. A current-object rule alone can leave many old versions charging indefinitely. One Zone choices change failure scope; use them only when data can be recreated or the risk is accepted.

Do not enable compliance retention as a reversible experiment. Object Lock governance/compliance and legal holds belong to the protected-backup design in tutorial 07.3.

## 7. Compare specialized bucket models

For a low-latency zonal processing workload, write an S3 Express/directory-bucket design that identifies its zone, endpoint, authorization and supported-feature differences. Do not copy a general purpose bucket policy blindly.

For a dataset of `order_id, customer_id, amount, event_time`, outline an Iceberg table in S3 Tables: define schema, table bucket, catalog/query-engine integration and maintenance ownership. For semantic retrieval, define an S3 Vectors index with one fixed embedding dimension, document IDs and filterable tenant metadata. Insert three synthetic embedding records conceptually, query by a nearby vector, and require the correct tenant filter. These models use distinct APIs and access controls; neither is merely an ordinary object-prefix rename.

## Verify, troubleshoot, and clean up

Pass with recorded old/current versions, restored matching content, denied anonymous/cross-prefix access and a lifecycle worksheet. If restore fails, check version ID, permissions and encryption key, not only the object's apparent filename. If CDN content is stale, separate cache behavior from S3 reads.

**Cleanup/cost:** permanently remove every dedicated object version and delete marker, abort unfinished multipart uploads, then delete the bucket. Deleting current objects alone may leave versions. Include requests, storage, retrieval, transitions, KMS, transfer and optional telemetry in the cost model.

Sources: [S3 guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/), [versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html), [classes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html), [S3 Tables](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html), [S3 Vectors](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors.html).
