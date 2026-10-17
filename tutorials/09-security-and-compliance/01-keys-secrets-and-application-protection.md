# Tutorial 09.1 — Test encryption context and design secret rotation

[Domain tutorials](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../09-security-and-compliance.md)

**Mode:** optional KMS sandbox exercise with dummy text; secret/certificate/HSM branches are worked designs. **Result:** an encryption-context negative test and lifecycle runbooks. Reviewed 2026-10-18.

## 1. Define the threat and key ownership

The portal stores private originals and a database credential. Encryption at rest addresses particular storage exposure; TLS protects transport; client-side encryption changes who can see plaintext before it reaches a managed service. None prevents an authorized compromised application from requesting plaintext that its permissions permit.

Write three identities: key administrator, application key user and recovery operator. State who may change policy, encrypt/decrypt and restore retained data. Administration and cryptographic use should be deliberately separated rather than assumed from the role name.

For a symmetric encryption KMS key, record key ID, ARN, alias, Region, policy, grants, rotation and deletion owner. An alias points to a key; changing it does not magically re-encrypt data protected by the former key.

## 2. Create a dedicated test key only if needed

In the intended sandbox/Region, create a customer-managed symmetric encryption key dedicated to this lab. Review current key/request charges first. Select only the approved setup administrator and lab user identities in the key policy. Do not edit a shared application key.

Give the test user only the required operations on this specific key, with appropriate policy authorization. KMS combines key policy and identity/grant behavior; an IAM allow alone is not universally sufficient. Record the key ARN and optional alias `alias/roadmap-training-key`.

Create a tiny local UTF-8 plaintext file containing `dummy-training-secret`. It contains no real credential. Use explicit account verification and Region:

```powershell
$labProfile = 'roadmap'
$labRegion = 'ap-south-1'
aws sts get-caller-identity --profile $labProfile --region $labRegion
$cipherB64 = aws kms encrypt --key-id alias/roadmap-training-key --plaintext fileb://dummy.txt --encryption-context Purpose=RoadmapTraining --query CiphertextBlob --output text --profile $labProfile --region $labRegion
if ($LASTEXITCODE -ne 0) { throw 'Encryption failed' }
[IO.File]::WriteAllBytes((Join-Path $PWD 'cipher.bin'), [Convert]::FromBase64String($cipherB64.Trim()))
```

Check success before decoding output. The `fileb://` input means binary file bytes; do not confuse base64 text with the binary ciphertext blob expected by decrypt.

## 3. Test decryption and context mismatch

```powershell
$plainB64 = aws kms decrypt --ciphertext-blob fileb://cipher.bin --encryption-context Purpose=RoadmapTraining --query Plaintext --output text --profile $labProfile --region $labRegion
if ($LASTEXITCODE -ne 0) { throw 'Decryption failed' }
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($plainB64.Trim()))
```

Expected: the dummy text. Repeat using `Purpose=DifferentExercise`; the operation should fail because the authenticated context does not match. Context is not a secret storage field and can appear in logs; do not put passwords or sensitive personal data there.

Now design an IAM/key-policy condition permitting decrypt only for the intended context. Distinguish ciphertext-integrity binding from an authorization condition: matching context alone does not grant an unauthorized caller key access. Test a different unprivileged lab identity only if already available; do not create long-lived access keys for this purpose.

## 4. Explain envelope encryption and rotation

Draw plaintext data → data-key encryption → stored ciphertext plus protected data key. KMS protects the data key; the application/service handles its documented envelope workflow. Do not send large objects directly to an API intended for small payload encryption.

Add a key-rotation event. Existing ciphertext must remain decryptable under retained key material; rotation does not rewrite every old application object. If a data key itself is compromised, rotating a wrapping key does not automatically erase that exposure. Define the service-supported re-encryption procedure if requirements demand it.

For imported key material, assign responsibility for source material, expiration and recovery. For multi-Region keys, distinguish related key material from automatically replicated policies, aliases and encrypted application data. For asymmetric/HMAC uses, choose signing/verification or MAC requirements rather than applying symmetric-encryption assumptions universally.

## 5. Design secret rotation with two systems

Use a fictional database credential `portal-runtime`. In Secrets Manager define secret ownership, allowed runtime role, encryption key, rotation workflow and application refresh behavior. Rotation must update both the target database and the stored secret through a supported sequence; changing one side first without coordination can cause an outage.

Write four states: old credential valid, new credential created/tested, new secret version promoted, old credential retired. Record rollback behavior at each failure point. The application caches the secret for a bounded interval and refreshes on documented authentication failure without logging the value.

Compare Parameter Store for configuration and supported encrypted values. Specify which features matter: automatic rotation integrations, version/staging semantics, pricing, throughput and operational ownership. Do not assume a `SecureString` alone supplies a full database-rotation workflow.

Negative tests: runtime role cannot edit the secret; deployment logs do not print it; an expired/rotated credential triggers recovery; a missing secret does not cause fallback to a hardcoded password.

## 6. Design certificates and specialized HSM use

For the portal's HTTPS endpoint, list domain ownership validation, certificate hostname coverage, supported attachment Region/service, renewal and expiry alerting. ACM-managed renewal differs from the responsibilities for imported certificates. A valid certificate for one domain does not authenticate a different hostname reached through an accidental DNS record.

For internal service certificates, evaluate Private CA only when the issuance/trust model warrants its cost. Define root/intermediate hierarchy, issuance permissions, client trust distribution and revocation/expiry. Use a design exercise rather than deploying a CA just to obtain a screenshot.

For a requirement demanding dedicated HSM control, evaluate CloudHSM: cryptographic users, clustering, backup, integration libraries and operational recovery. The deliverable is an owner/availability diagram and a reason KMS does or does not meet the requirement. Dedicated hardware does not eliminate application key-management mistakes.

## 7. Protect the consuming application

Add parameterized SQL, per-object authorization, token validation, constrained upload types/sizes and appropriate browser headers to the application checklist. Dependencies and container images must be scanned and maintained; artifacts should be identifiable and trusted. A WAF or encryption setting does not fix a query assembled from untrusted string input.

Review where secrets could leak: source commits, environment dumps, AMI/container layers, user data, screenshots, exceptions and traces. Use a fabricated canary value in a local logging test and confirm it is redacted where expected.

## Verify and clean up

Pass with matching-context decrypt success, mismatched-context failure and a complete rotation/certificate runbook. For KMS denial, inspect caller/key Region, key state, policy/grants, operation and context. For invalid ciphertext, inspect binary/base64 handling before changing policy.

**Cleanup/cost:** remove local dummy files and only dedicated test secrets/resources. Before scheduling deletion of the dedicated KMS key, verify no retained data needs it; never delete a shared key. Record the supported pending-deletion/recovery window. Retained keys, secrets, certificates/CA/HSM and API operations have different cost dimensions.

Sources: [KMS guide](https://docs.aws.amazon.com/kms/latest/developerguide/), [key deletion](https://docs.aws.amazon.com/kms/latest/developerguide/deleting-keys.html), [Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html), [ACM](https://docs.aws.amazon.com/acm/latest/userguide/), [CloudHSM](https://docs.aws.amazon.com/cloudhsm/latest/userguide/).
