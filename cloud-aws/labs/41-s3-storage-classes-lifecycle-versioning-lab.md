# S3 Hands-On Lab — Storage Classes, Lifecycle, Versioning

10 exercises to build real muscle memory on S3. Use `us-east-1` (no LocationConstraint needed there) or add your region's LocationConstraint. Pick a globally unique bucket name — `s3-lab-<yourname>-<random>`.

## Setup

```bash
export BUCKET="s3-lab-satyam-$(date +%s)"
export REGION="ap-south-1"
aws s3api create-bucket --bucket $BUCKET --region $REGION \
  --create-bucket-configuration LocationConstraint=$REGION
```

---

## Exercise 1 — Upload objects and inspect storage class

**Goal:** Upload files and confirm they land in STANDARD.

```bash
echo "hot data" > app.log
aws s3 cp app.log s3://$BUCKET/logs/app.log
aws s3api head-object --bucket $BUCKET --key logs/app.log
```

**Expected:** `head-object` shows `"StorageClass": "STANDARD"`. No class shown in `ls` output — check with `head-object`.

**Why it matters:** Default class is always STANDARD. Interviewers expect you to know the default isn't "no class".

---

## Exercise 2 — Enable versioning and recover an overwrite

**Goal:** Overwrite a file and restore the original.

```bash
aws s3api put-bucket-versioning --bucket $BUCKET \
  --versioning-configuration Status=Enabled
echo "v1" > config.json && aws s3 cp config.json s3://$BUCKET/config.json
echo "v2-corrupt" > config.json && aws s3 cp config.json s3://$BUCKET/config.json
aws s3api list-object-versions --bucket $BUCKET --prefix config.json
# Copy the VersionId of v1, then restore it:
aws s3api get-object --bucket $BUCKET --key config.json --version-id <V1_ID> restored.json
aws s3 cp restored.json s3://$BUCKET/config.json
```

**Expected:** Two versions listed; after restore, a third version containing "v1".

**Why it matters:** This is the exact recovery flow for a corrupted Terraform state file.

---

## Exercise 3 — See what a delete really does with versioning on

**Goal:** Prove a delete is just a marker.

```bash
aws s3 rm s3://$BUCKET/config.json
aws s3api list-object-versions --bucket $BUCKET --prefix config.json
```

**Expected:** All three versions still listed, plus a `DeleteMarkers` entry on top. `aws s3 ls s3://$BUCKET/config.json` shows nothing.

**Why it matters:** "I deleted it, why is the bill the same?" — this exercise is the answer.

---

## Exercise 4 — Apply a lifecycle rule: Standard → IA → Glacier → expire

**Goal:** Age objects into cheaper classes automatically.

```bash
cat > lifecycle.json <<'EOF'
{
  "Rules": [{
    "ID": "logs-tiering",
    "Status": "Enabled",
    "Filter": {"Prefix": "logs/"},
    "Transitions": [
      {"Days": 30, "StorageClass": "STANDARD_IA"},
      {"Days": 90, "StorageClass": "GLACIER"}
    ],
    "NoncurrentVersionTransitions": [
      {"NoncurrentDays": 30, "StorageClass": "STANDARD_IA"}
    ],
    "NoncurrentVersionExpiration": {"NoncurrentDays": 90},
    "Expiration": {"Days": 365}
  }]
}
EOF
aws s3api put-bucket-lifecycle-configuration --bucket $BUCKET \
  --lifecycle-configuration file://lifecycle.json
aws s3api get-bucket-lifecycle-configuration --bucket $BUCKET
```

**Expected:** Rule returned by `get-bucket-lifecycle-configuration` with status Enabled.

**Why it matters:** Lifecycle is the main cost lever in S3. Forgetting the noncurrent rules is the classic cost leak.

---

## Exercise 5 — Compare storage class pricing in one command

**Goal:** See how much a terabyte costs in each class.

```bash
aws pricing get-products --service-code AmazonS3 --region-code $REGION \
  --filters "Type=TERM_MATCH,Field=storageClass,Value=Standard" \
  --max-items 1 --query 'PriceList[0]' --output text | head -c 400
```

**Expected:** A JSON blob with per-GB-month pricing. Repeat with `Value=Glacier Deep Archive` and compare.

**Why it matters:** Quoting rough price ratios (Deep Archive ~1/20th of Standard) in interviews signals real cost awareness.

---

## Exercise 6 — Turn on SSE-KMS with a Bucket Key

**Goal:** Encrypt with KMS and watch costs stay sane.

```bash
aws kms create-key --description "s3-lab-key" --query 'KeyMetadata.KeyId' --output text
# Save the key id, then:
aws s3api put-bucket-encryption --bucket $BUCKET \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "<KEY_ID>",
        "BucketKeyEnabled": true
      }
    }]
  }'
aws s3api get-bucket-encryption --bucket $BUCKET
```

**Expected:** Encryption config shows `aws:kms` with `BucketKeyEnabled: true`.

**Why it matters:** Bucket Key cuts KMS request costs by up to 99% — a detail interviewers reward.

---

## Exercise 7 — Deny all non-TLS traffic with a bucket policy

**Goal:** Lock the bucket down with an explicit deny.

```bash
cat > deny-http.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": ["arn:aws:s3:::$BUCKET", "arn:aws:s3:::$BUCKET/*"],
    "Condition": {"Bool": {"aws:SecureTransport": "false"}}
  }]
}
EOF
aws s3api put-bucket-policy --bucket $BUCKET --policy file://deny-http.json
```

**Expected:** Policy accepted. Try `curl http://...` style access — denied.

**Why it matters:** This exact deny statement is a standard compliance requirement and a favorite interview question.

---

## Exercise 8 — Generate a pre-signed URL and test expiry

**Goal:** Create a 5-minute download link.

```bash
aws s3 presign s3://$BUCKET/logs/app.log --expires-in 300
```

**Expected:** A long HTTPS URL. `curl` it within 5 minutes — downloads. After 5 minutes — `AccessDenied`.

**Why it matters:** This is how apps serve private S3 content without IAM credentials on the client.

---

## Exercise 9 — Find and clean up incomplete multipart uploads

**Goal:** Discover the silent bill inflator and kill it.

```bash
# Simulate an abandoned upload:
aws s3api create-multipart-upload --bucket $BUCKET --key bigfile.bin
aws s3api list-multipart-uploads --bucket $BUCKET
aws s3api abort-multipart-upload --bucket $BUCKET --key bigfile.bin \
  --upload-id <UPLOAD_ID>
# Real fix — add an abort rule to your lifecycle config:
# "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
```

**Expected:** Upload listed, then gone after abort. Storage that would have been billed forever is freed.

**Why it matters:** Abandoned multipart uploads are the #1 cause of "my S3 bill doubled and I changed nothing".

---

## Exercise 10 — Cleanup

**Goal:** Empty and delete the lab bucket cleanly.

```bash
aws s3 rm s3://$BUCKET --recursive
# Versioned objects and delete markers also need removal:
aws s3api list-object-versions --bucket $BUCKET --output text | \
  awk '{print $3, $5}' | while read ver key; do :; done
# Easiest reliable path — delete every version id:
for v in $(aws s3api list-object-versions --bucket $BUCKET \
  --query 'Versions[].VersionId' --output text); do
  aws s3api delete-object --bucket $BUCKET --key config.json --version-id $v
done
for m in $(aws s3api list-object-versions --bucket $BUCKET \
  --query 'DeleteMarkers[].VersionId' --output text); do
  aws s3api delete-object --bucket $BUCKET --key config.json --version-id $m
done
aws s3api delete-bucket --bucket $BUCKET
```

**Expected:** Bucket deleted, `aws s3 ls` no longer shows it.

**Why it matters:** Versioned buckets can't be deleted until versions and markers are gone — a trap in both exams and interviews.
