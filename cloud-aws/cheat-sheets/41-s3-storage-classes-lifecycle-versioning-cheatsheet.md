# S3 Cheat Sheet — Storage Classes, Lifecycle, Versioning

## Buckets

```bash
aws s3api create-bucket --bucket my-bucket --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1
aws s3 ls s3://my-bucket
aws s3api get-bucket-location --bucket my-bucket
```

Names globally unique, bucket lives in one region. 100 buckets/account default (soft limit).

## Objects

```bash
aws s3 cp file.txt s3://my-bucket/logs/file.txt
aws s3 sync ./dist s3://my-bucket/static --delete
aws s3api head-object --bucket my-bucket --key logs/file.txt   # metadata + storage class
aws s3api list-objects-v2 --bucket my-bucket --prefix logs/
```

Object = key + data + metadata. Max 5 TB per object; 5 GB single PUT, multipart above.

## Storage classes (hot → cold)

| Class | Use for | Retrieval |
|---|---|---|
| Standard | Hot data | ms |
| Intelligent-Tiering | Unpredictable access | ms, auto-moves |
| Standard-IA | Backups, DR | ms, higher per-request cost |
| One Zone-IA | Recreatable data | ms, single AZ |
| Glacier Instant | Archives needing ms access | ms |
| Glacier Flexible | Archives, minutes–hours | min–hrs |
| Glacier Deep Archive | Compliance archives | up to 12 hrs |

```bash
aws s3 cp file.txt s3://my-bucket/file.txt --storage-class STANDARD_IA
```

## Lifecycle

```bash
aws s3api put-bucket-lifecycle-configuration --bucket my-bucket \
  --lifecycle-configuration file://lifecycle.json
aws s3api get-bucket-lifecycle-configuration --bucket my-bucket
```

Rules: Transitions (age → cheaper class), Expiration (age → delete), NoncurrentVersionExpiration, AbortIncompleteMultipartUpload. Minimum billable sizes: Standard-IA 128 KB / 30 days, Glacier 40 KB / 90 days.

## Versioning

```bash
aws s3api put-bucket-versioning --bucket my-bucket \
  --versioning-configuration Status=Enabled
aws s3api get-bucket-versioning --bucket my-bucket
aws s3api list-object-versions --bucket my-bucket --prefix file.txt
aws s3api delete-object --bucket my-bucket --key file.txt --version-id <id>
```

Delete = delete marker on top. Enable MFA Delete for state buckets.

## Encryption

- **SSE-S3** — Amazon-managed keys, default, zero effort.
- **SSE-KMS** — your KMS key, CloudTrail audit, use **Bucket Key** (cuts KMS costs ~99%).
- **SSE-C** — you supply the key in headers.
- **Client-side** — encrypt before upload, S3 never sees plaintext.

```bash
aws s3api put-bucket-encryption --bucket my-bucket \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"aws:kms","KMSMasterKeyID":"<id>","BucketKeyEnabled":true}}]}'
```

## Access control

- Block Public Access: account-level, keep all four ON.
- Bucket policy for cross-account / service access; IAM policy for users.
- Deny non-TLS: `Condition: {"Bool": {"aws:SecureTransport": "false"}}`.
- Pre-signed URL: `aws s3 presign s3://b/key --expires-in 900`.

## Gotchas to memorize

- Overlapping-prefix throttles: 5,500 reads/sec **per prefix** — randomize prefixes.
- S3 is eventually consistent for nothing now — **strong read-after-write consistency** since Dec 2020.
- Data transfer **in** is free; transfer **out** and cross-region replication cost.
- Versioned deletes add markers — the bill stays until versions are removed.
- `aws s3 rm` on a versioned bucket hides, never frees.
