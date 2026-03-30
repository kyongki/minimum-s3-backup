# Minimal Cost, Maximum Protection: Building S3 Ransomware Defense and Point-in-Time Recovery with Native AWS Services

*A practical guide to building a cost-effective S3 data protection strategy using native AWS services*

---

## Introduction

Customers trust Amazon S3 with their most valuable data — driven by its 11 nines of durability and proven operational reliability. Naturally, they want to protect that data from ransomware, accidental deletion, and operator errors through backup. However, many hesitate to act when faced with the sheer volume of data and the associated backup costs.

The consequences of inaction can be severe. Some organizations managing hundreds of terabytes find that per-job backup pricing makes adoption impractical, leaving critical data exposed with no structured recovery strategy. Others discover the gap only after an incident — when operator mistakes or malicious actions cause irreversible data loss, and there is simply nothing to restore from.

AWS Backup for S3 is the most comprehensive solution, offering centralized management, continuous backup, and seamless point-in-time recovery. However, at scale — say, **500 TB across 500 million objects** — the cost can be substantial, and per-object charges alone may reach significant amounts.

For customers who gave up on backup due to cost, this architecture offers a practical path forward. By combining native S3 capabilities — **Versioning**, **Cross-Account Replication**, **Object Lock**, **Lifecycle to Glacier Deep Archive**, **S3 Metadata**, and **S3 Batch Operations** — you can protect your data at an estimated **60–80% lower cost** compared to AWS Backup, making backup feasible even for the largest and most cost-sensitive workloads. While this approach requires more operational effort than AWS Backup, it provides essential data protection at roughly **$600/month** for a 500 TB environment.

## Cost Comparison at a Glance

| Component | AWS Backup | This Architecture |
|-----------|-----------|-------------------|
| Backup storage (500 TB) | ~$11,500/mo (periodic backup) | ~$500/mo (Deep Archive) |
| Per-object charges (backup/restore) | Varies by backup type — see [AWS Backup pricing](https://aws.amazon.com/backup/pricing/) | **$0** |
| Object Lock / Versioning | — | **$0** (free features) |
| S3 Metadata (PITR) | — | ~$50/mo |
| Data transfer (same-region) | — | **$0** |
| **Monthly total (additional)** | **~$12,000+** (storage alone) | **~$600** |

> **Note on AWS Backup pricing**: AWS Backup for S3 charges differently for periodic vs. continuous backup, and restore operations incur additional per-object fees. At 500 million objects, these per-object costs can be substantial. Always consult the [latest pricing page](https://aws.amazon.com/backup/pricing/) for your specific backup configuration.

## Architecture Overview

The architecture has two layers: **protection** (prevent and survive attacks) and **recovery** (search and restore to any point in time).

```
┌──────────────────────────────────────────────────────────────────┐
│  Account A (Production)                                          │
│  ┌────────────────────────────────┐                              │
│  │  Source Bucket                  │                              │
│  │  - Versioning: Enabled         │── Same-Region ──┐            │
│  │  - IAM Deny Policies           │   Replication    │            │
│  │  - Lifecycle: noncurrent mgmt  │                  │            │
│  │  - S3 Metadata: Journal +      │                  │            │
│  │    Live Inventory (Iceberg)    │                  │            │
│  └────────────────────────────────┘                  │            │
│  ┌────────────────────────────────┐                  │            │
│  │  Ops Bucket (A)                │                  │            │
│  │  - PITR manifests & reports    │                  │            │
│  └────────────────────────────────┘                  │            │
│  CloudTrail + GuardDuty S3 Protection                │            │
└──────────────────────────────────────────────────────│────────────┘
                                                       │
┌──────────────────────────────────────────────────────│────────────┐
│  Account B (Backup — Isolated)                       │            │
│  ┌────────────────────────────────┐                  │            │
│  │  Destination Bucket            │◄─────────────────┘            │
│  │  - Object Lock: Compliance 180d│                               │
│  │  - Storage: Deep Archive       │                               │
│  │  - Bucket Policy: Deny Delete  │                               │
│  │  - S3 Metadata: Journal +      │                               │
│  │    Live Inventory (Iceberg)    │                               │
│  └────────────────────────────────┘                               │
│  ┌────────────────────────────────┐                               │
│  │  Ops Bucket (B)                │                               │
│  │  - PITR manifests & reports    │                               │
│  └────────────────────────────────┘                               │
│  Object Lock Compliance: immutable for 180 days                  │
└───────────────────────────────────────────────────────────────────┘
```

### Threat-to-Recovery Matrix

| Threat | Protection Layer | Recovery Path | RTO |
|--------|-----------------|---------------|-----|
| Accidental overwrite | Source versioning | Restore previous version from source | Immediate |
| Accidental delete | Source versioning (delete marker) | Remove delete marker | Immediate |
| Ransomware (mass overwrite) | Source versioning + IAM deny | PITR from source versions via Batch Ops | Hours |
| Account A takeover | Account B isolation + Compliance Lock | PITR from Account B via Batch Ops (restore + copy) | 12–48 hours |

### Why Not Just Object Lock on the Source Bucket?

A natural question arises: **if Object Lock (Compliance mode) prevents even root from deleting objects, why not just enable it on the source bucket and skip cross-account backup entirely?**

Object Lock on the source bucket does provide meaningful protection — Compliance mode ensures locked object versions cannot be deleted during the retention period, regardless of IAM permissions. However, it has critical gaps when the **AWS account itself** is compromised:

| Threat Scenario | Source Object Lock Only | Cross-Account Backup |
|-----------------|------------------------|---------------------|
| IAM compromise → delete existing objects | **Protected** (Compliance mode blocks deletion) | **Protected** |
| IAM compromise → upload malicious objects | Not blocked (new objects are accepted) | PITR from backup restores clean state |
| **AWS account takeover (root credentials)** | **Risk**: account closure suspends access (resources deleted after 90-day grace period) | **Protected** (separate account) |
| Root compromise → Bucket Policy deny all access | **Risk**: data becomes inaccessible | **Protected** |
| Root compromise → disable CloudTrail, tamper logs | **Risk**: forensic evidence lost | Backup account retains independent audit trail |
| Insider threat (admin collusion) | Single account = single trust boundary | **Isolated** across trust boundaries |

**The key gap**: Object Lock protects *objects within a bucket*, but it cannot protect against **account-level actions** — closing the account, modifying billing settings to trigger suspension, or applying deny-all bucket policies. If an attacker gains root access to the account, the data may be technically intact but operationally unreachable.

Cross-account backup addresses this by creating a **separate trust boundary**:

- **Account isolation**: Compromising Account A gives zero access to Account B's IAM, SCPs, or resources
- **Object Lock immutability**: Compliance mode on Account B blocks all deletes with no exceptions — even root cannot delete or shorten retention on locked objects. Once enabled, Object Lock cannot be removed from the bucket
- **Independent metadata**: Account B maintains its own S3 Metadata journal, enabling PITR even if Account A's metadata is destroyed
- **Compliance alignment**: Many regulatory frameworks (financial services, healthcare) explicitly require backup copies in a separate account or environment

**Recommendation**: Use Object Lock on the source bucket as a **first line of defense** against everyday threats (accidental deletes, IAM-level attacks). Use cross-account backup as the **last line of defense** against account-level compromise — the scenario where everything else has failed.

> **Why this walkthrough does not enable Object Lock on the source bucket**: Object Lock on a production bucket adds operational constraints — locked object versions cannot be deleted during the retention period, which complicates data lifecycle management such as storage cleanup and compliance-driven data removal. (Note: Object Lock does not block overwrites — a `PutObject` to the same key creates a new version while the locked old version is preserved as noncurrent.) This walkthrough uses **IAM deny policies + Versioning** on the source bucket instead, which provides similar protection (block unauthorized deletes, preserve all versions) with greater operational flexibility. If your workload can tolerate retention constraints on deletion, enabling Object Lock on the source bucket is a worthwhile addition.

## Prerequisites

Before starting the walkthrough, ensure you have:

- **Two AWS accounts** (Account A: production, Account B: backup) — they do not need to be in the same AWS Organization
- **Both accounts in the same Region** (e.g., `us-east-1`) to avoid data transfer costs
- **AWS CLI v2.27.51 or later** configured with appropriate credentials for both accounts (S3 Metadata V2 API requires this version)
- **Permissions**: IAM admin access for both accounts

```bash
# Set up CLI profiles for both accounts
aws configure --profile account-a
aws configure --profile account-b

# Variables used throughout this walkthrough
export ACCOUNT_A=111111111111
export ACCOUNT_B=222222222222
export REGION=us-east-1
export SOURCE_BUCKET=my-source-bucket
export DEST_BUCKET=my-backup-bucket
export OPS_BUCKET_A=my-ops-a
export OPS_BUCKET_B=my-ops-b
export RESTORE_BUCKET_A=my-restore-bucket
export RESTORE_BUCKET_B=my-restore-bucket-b
```

---

## Step 1: Configure the Source Bucket (Account A)

### 1.1 Enable Versioning

Versioning is the foundation of the entire architecture. Every overwrite or delete preserves the previous version, making recovery possible.

```bash
aws s3api put-bucket-versioning \
  --bucket ${SOURCE_BUCKET} \
  --versioning-configuration Status=Enabled \
  --profile account-a
```

Verify:

```bash
aws s3api get-bucket-versioning \
  --bucket ${SOURCE_BUCKET} \
  --profile account-a

# Expected output:
# { "Status": "Enabled" }
```

### 1.2 Configure Lifecycle for Noncurrent Versions

Without lifecycle rules, old versions accumulate indefinitely. This policy keeps costs under control while maintaining a recovery window:

```bash
cat > /tmp/lifecycle.json << 'EOF'
{
  "Rules": [
    {
      "ID": "manage-noncurrent-versions",
      "Status": "Enabled",
      "Filter": {},
      "NoncurrentVersionTransitions": [
        {
          "NoncurrentDays": 7,
          "StorageClass": "GLACIER_IR"
        }
      ],
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 90,
        "NewerNoncurrentVersions": 3
      }
    }
  ]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
  --bucket ${SOURCE_BUCKET} \
  --lifecycle-configuration file:///tmp/lifecycle.json \
  --profile account-a
```

This configuration:
- Keeps noncurrent versions accessible for **7 days**, then transitions to Glacier IR
- Retains the **3 most recent** noncurrent versions per object
- Deletes noncurrent versions older than **90 days**

> **Note:** Only noncurrent versions are transitioned to Glacier IR — these are created only when objects are overwritten or deleted. Current (latest) versions remain in their original storage class. The actual Glacier IR storage volume depends on your data change rate, not the total bucket size.

---

## Step 2: Configure the Destination Bucket (Account B)

### 2.1 Create the Bucket with Object Lock

Object Lock has traditionally required enabling at bucket creation time. However, AWS now supports enabling Object Lock on existing buckets — you can do this via the S3 console, CLI, or API using `put-object-lock-configuration`. Note that Object Lock requires versioning (which must be enabled first or will be enabled as part of the process), and default retention settings apply only to new object versions; existing versions are not retroactively locked.

For this walkthrough, we create a new bucket:

```bash
# For us-east-1, omit --create-bucket-configuration
aws s3api create-bucket \
  --bucket ${DEST_BUCKET} \
  --region ${REGION} \
  --create-bucket-configuration LocationConstraint=${REGION} \
  --object-lock-enabled-for-bucket \
  --profile account-b
```

### 2.2 Set Object Lock to Compliance Mode

Compliance mode ensures that **no one — not even the root user** — can delete or modify objects during the retention period. This is the critical defense against account takeover.

> **Why 180 days?** Glacier Deep Archive has a **180-day minimum storage charge** — objects deleted before 180 days are still billed for the full period. Setting Object Lock retention to 180 days aligns with this minimum and provides a longer protection window.

```bash
aws s3api put-object-lock-configuration \
  --bucket ${DEST_BUCKET} \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "COMPLIANCE",
        "Days": 180
      }
    }
  }' \
  --profile account-b
```

### 2.3 Why Glacier Deep Archive?

The destination bucket is a **last-resort backup** — used only when Account A is fully compromised. In that scenario, waiting 12–48 hours for restore is acceptable, and the cost savings are substantial.

| Factor | Glacier Instant Retrieval | Deep Archive (Bulk, 48h) | Deep Archive (Standard, 12h) |
|--------|--------------------------|-------------------------|------------------------------|
| Monthly storage (500 TB) | $2,000 | $495 | $495 |
| Full restore request cost (500M obj) | $5,000 | $12,500 | $50,000 |
| Full restore retrieval cost (500 TB) | $15,000 | $1,250 | $10,000 |
| **Total recovery cost** | **$20,000** | **$13,750** | **$60,000** |
| Restore time | Milliseconds | 48 hours | 12 hours |

**Bulk retrieval** (48 hours) is the best option for full-scale DR — it is actually **cheaper than Glacier IR** for both storage and recovery. For smaller, targeted restores where faster turnaround matters, **Standard retrieval** (12 hours) is available at higher per-request cost.

Deep Archive saves **$1,505/month** ($18,060/year) on storage alone. Combined with Bulk retrieval, a full DR restore costs **$6,250 less** than Glacier IR. The only trade-offs are restore latency (12–48 hours) and a two-step recovery process (restore → copy), which we cover in Step 6.4.

### 2.4 Create Ops Buckets (for Manifests and Reports)

These buckets store Batch Operations completion reports and PITR manifests. We create them now because they are referenced in later steps:

```bash
# Account A ops bucket (for us-east-1, omit --create-bucket-configuration)
aws s3api create-bucket \
  --bucket ${OPS_BUCKET_A} \
  --region ${REGION} \
  --create-bucket-configuration LocationConstraint=${REGION} \
  --profile account-a

# Account B ops bucket
aws s3api create-bucket \
  --bucket ${OPS_BUCKET_B} \
  --region ${REGION} \
  --create-bucket-configuration LocationConstraint=${REGION} \
  --profile account-b
```

---

## Step 3: Set Up Cross-Account Same-Region Replication

Same-region replication (SRR) between accounts in the same region incurs **zero data transfer cost** — a key advantage over cross-region replication, which would add ~$10,000/month at this scale.

### 3.1 Create the Replication IAM Role (Account A)

```bash
# Trust policy
cat > /tmp/replication-trust.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "s3.amazonaws.com",
          "batchoperations.s3.amazonaws.com"
        ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name s3-replication-role \
  --assume-role-policy-document file:///tmp/replication-trust.json \
  --profile account-a

# Permission policy
cat > /tmp/replication-perms.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetReplicationConfiguration",
        "s3:PutInventoryConfiguration",
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::${SOURCE_BUCKET}"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObjectVersionForReplication",
        "s3:GetObjectVersionAcl",
        "s3:GetObjectVersionTagging",
        "s3:InitiateReplication"
      ],
      "Resource": "arn:aws:s3:::${SOURCE_BUCKET}/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ReplicateObject",
        "s3:ReplicateDelete",
        "s3:ReplicateTags",
        "s3:ObjectOwnerOverrideToBucketOwner"
      ],
      "Resource": "arn:aws:s3:::${DEST_BUCKET}/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::${OPS_BUCKET_A}/*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name s3-replication-role \
  --policy-name s3-replication-policy \
  --policy-document file:///tmp/replication-perms.json \
  --profile account-a
```

### 3.2 Apply the Destination Bucket Policy (Account B)

Belt and suspenders — even without Object Lock, this policy allows replication from Account A while blocking all delete operations:

```bash
cat > /tmp/dest-bucket-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReplicationFromAccountA",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::${ACCOUNT_A}:role/s3-replication-role"
      },
      "Action": [
        "s3:ReplicateObject",
        "s3:ReplicateDelete",
        "s3:ReplicateTags",
        "s3:ObjectOwnerOverrideToBucketOwner",
        "s3:GetBucketVersioning"
      ],
      "Resource": [
        "arn:aws:s3:::${DEST_BUCKET}",
        "arn:aws:s3:::${DEST_BUCKET}/*"
      ]
    },
    {
      "Sid": "DenyAllDelete",
      "Effect": "Deny",
      "Principal": "*",
      "Action": [
        "s3:DeleteObject",
        "s3:DeleteObjectVersion",
        "s3:DeleteBucket"
      ],
      "Resource": [
        "arn:aws:s3:::${DEST_BUCKET}",
        "arn:aws:s3:::${DEST_BUCKET}/*"
      ]
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket ${DEST_BUCKET} \
  --policy file:///tmp/dest-bucket-policy.json \
  --profile account-b
```

> **Why apply this after Step 3.1?** The bucket policy references the replication role's ARN as a principal. AWS validates that the principal exists when the policy is applied, so the role must be created first.

### 3.3 Configure the Replication Rule

```bash
cat > /tmp/replication-config.json << EOF
{
  "Role": "arn:aws:iam::${ACCOUNT_A}:role/s3-replication-role",
  "Rules": [
    {
      "ID": "cross-account-backup",
      "Status": "Enabled",
      "Priority": 1,
      "Filter": {},
      "Destination": {
        "Bucket": "arn:aws:s3:::${DEST_BUCKET}",
        "Account": "${ACCOUNT_B}",
        "StorageClass": "DEEP_ARCHIVE",
        "AccessControlTranslation": {
          "Owner": "Destination"
        },
        "ReplicationTime": {
          "Status": "Enabled",
          "Time": { "Minutes": 15 }
        },
        "Metrics": {
          "Status": "Enabled",
          "EventThreshold": { "Minutes": 15 }
        }
      },
      "DeleteMarkerReplication": {
        "Status": "Disabled"
      }
    }
  ]
}
EOF

aws s3api put-bucket-replication \
  --bucket ${SOURCE_BUCKET} \
  --replication-configuration file:///tmp/replication-config.json \
  --profile account-a
```

> **Key setting: `DeleteMarkerReplication: Disabled`** — When an object is deleted in Account A, the delete marker is NOT replicated to Account B. This means even if an attacker deletes everything in the source, the backup copies remain fully intact.

### 3.4 Replicate Existing Objects with S3 Batch Replication

The replication rule above only applies to new objects. To replicate the existing 500 million objects, use S3 Batch Replication:

```bash
aws s3control create-job \
  --account-id ${ACCOUNT_A} \
  --operation '{"S3ReplicateObject":{}}' \
  --manifest-generator '{
    "S3JobManifestGenerator": {
      "ExpectedBucketOwner": "'${ACCOUNT_A}'",
      "SourceBucket": "arn:aws:s3:::'${SOURCE_BUCKET}'",
      "EnableManifestOutput": false,
      "Filter": {
        "EligibleForReplication": true,
        "ObjectReplicationStatuses": ["NONE", "FAILED"]
      }
    }
  }' \
  --priority 10 \
  --role-arn arn:aws:iam::${ACCOUNT_A}:role/s3-replication-role \
  --report '{
    "Bucket": "arn:aws:s3:::'${OPS_BUCKET_A}'",
    "Prefix": "batch-replication-report",
    "Format": "Report_CSV_20180820",
    "Enabled": true,
    "ReportScope": "AllTasks"
  }' \
  --confirmation-required \
  --region ${REGION} \
  --profile account-a
```

After the job is created, confirm it to start execution:

```bash
# Wait for the job to reach "Suspended" status (it transitions from New → Preparing → Suspended)
aws s3control describe-job \
  --account-id ${ACCOUNT_A} \
  --job-id <JOB_ID> \
  --query 'Job.Status' \
  --region ${REGION} \
  --profile account-a

# Once status is "Suspended", confirm the job to start execution
aws s3control update-job-status \
  --account-id ${ACCOUNT_A} \
  --job-id <JOB_ID> \
  --requested-job-status Ready \
  --region ${REGION} \
  --profile account-a
```

---

## Step 4: Lock Down with IAM and Bucket Policies

### 4.1 IAM: Deny Dangerous Actions (Account A)

Apply this as an inline deny policy (or attach as a managed policy) to all non-admin roles. This prevents unauthorized users from disabling versioning, modifying bucket policies, or changing replication settings:

```bash
cat > /tmp/deny-dangerous.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDangerousS3Actions",
      "Effect": "Deny",
      "Action": [
        "s3:PutBucketVersioning",
        "s3:DeleteBucket",
        "s3:PutBucketPolicy",
        "s3:DeleteBucketPolicy",
        "s3:PutReplicationConfiguration",
        "s3:PutLifecycleConfiguration"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name app-role \
  --policy-name deny-dangerous-s3 \
  --policy-document file:///tmp/deny-dangerous.json \
  --profile account-a
```

> **Scope**: Apply this policy to every IAM role that does not need administrative S3 control — application roles, CI/CD roles, developer roles, etc. Only a dedicated admin break-glass role should be exempt.

### 4.2 Why Object Lock Alone Protects Account B

Account B's destination bucket is already protected by three layers configured in Steps 2 and 3, **none of which require AWS Organizations**:

| Protection | What it blocks | Can root bypass? |
|-----------|---------------|-----------------|
| **Object Lock Compliance 180d** | Delete or modify any locked object version | **No** — Compliance mode is absolute |
| **Object Lock (bucket-level)** | Disabling Object Lock on the bucket | **No** — once enabled, cannot be removed |
| **Bucket Policy deny delete** | All `DeleteObject` / `DeleteBucket` calls | Yes, but Object Lock still blocks actual deletion |

The key insight: **Object Lock Compliance mode is the strongest protection available in S3** — it cannot be bypassed by root, IAM policies, or bucket policy changes. Even if an attacker gains root access to Account B, they cannot delete or modify locked object versions. This makes additional Organization-level controls (SCPs) a welcome layer of defense-in-depth but not a prerequisite.

### 4.3 Enable Detection: GuardDuty + CloudTrail

```bash
# Enable GuardDuty with S3 Protection (Account A)
# If GuardDuty is already enabled, get the existing detector ID and update it instead:
#   DETECTOR_ID=$(aws guardduty list-detectors --query 'DetectorIds[0]' --output text --profile account-a)
#   aws guardduty update-detector --detector-id ${DETECTOR_ID} \
#     --features '[{"Name":"S3_DATA_EVENTS","Status":"ENABLED"}]' --profile account-a
aws guardduty create-detector \
  --enable \
  --features '[{"Name":"S3_DATA_EVENTS","Status":"ENABLED"}]' \
  --profile account-a

# CloudTrail is typically already enabled for management events.
# For S3 data event monitoring (optional — costs scale with API call volume):
# Find your trail name and use one you own (skip organization-managed trails):
#   aws cloudtrail describe-trails --query 'trailList[?IsOrganizationTrail==`false`].Name' --profile account-a
# If you don't have your own trail, create one first:
#   aws cloudtrail create-trail --name my-s3-trail \
#     --s3-bucket-name my-cloudtrail-logs --profile account-a
#   aws cloudtrail start-logging --name my-s3-trail --profile account-a
TRAIL_NAME="my-s3-trail"  # Replace with your actual trail name
aws cloudtrail put-event-selectors \
  --trail-name ${TRAIL_NAME} \
  --event-selectors '[{
    "ReadWriteType": "WriteOnly",
    "IncludeManagementEvents": true,
    "DataResources": [{
      "Type": "AWS::S3::Object",
      "Values": ["arn:aws:s3:::'${SOURCE_BUCKET}'/"]
    }]
  }]' \
  --profile account-a
```

### 4.4 (Optional) Strengthen with SCPs via AWS Organizations

If both accounts belong to the same AWS Organization, you can add **Service Control Policies (SCPs)** for defense-in-depth. SCPs operate at the Organization level and cannot be bypassed by IAM — not even by the account's root user.

> **Note**: SCPs do not support environment variables. Replace the placeholder values (`<SOURCE_BUCKET>`, `<ACCOUNT_A>`, `<DEST_BUCKET>`, `<ACCOUNT_B>`) with your actual values before applying.

**SCP for Account A** — prevent versioning changes and bulk deletes:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyVersioningChange",
      "Effect": "Deny",
      "Action": "s3:PutBucketVersioning",
      "Resource": "arn:aws:s3:::<SOURCE_BUCKET>",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalArn": "arn:aws:iam::<ACCOUNT_A>:role/admin-break-glass"
        }
      }
    },
    {
      "Sid": "DenyBulkDelete",
      "Effect": "Deny",
      "Action": [
        "s3:DeleteObject",
        "s3:DeleteObjectVersion"
      ],
      "Resource": "arn:aws:s3:::<SOURCE_BUCKET>/*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalArn": [
            "arn:aws:iam::<ACCOUNT_A>:role/admin-break-glass",
            "arn:aws:iam::<ACCOUNT_A>:role/app-authorized-role"
          ]
        }
      }
    }
  ]
}
```

**SCP for Account B** — absolute deny on deletes and Object Lock changes:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAllS3Delete",
      "Effect": "Deny",
      "Action": [
        "s3:DeleteObject",
        "s3:DeleteObjectVersion",
        "s3:DeleteBucket"
      ],
      "Resource": [
        "arn:aws:s3:::<DEST_BUCKET>",
        "arn:aws:s3:::<DEST_BUCKET>/*"
      ]
    },
    {
      "Sid": "DenyObjectLockChanges",
      "Effect": "Deny",
      "Action": [
        "s3:PutObjectLockConfiguration",
        "s3:PutObjectRetention",
        "s3:PutObjectLegalHold",
        "s3:PutBucketVersioning"
      ],
      "Resource": [
        "arn:aws:s3:::<DEST_BUCKET>",
        "arn:aws:s3:::<DEST_BUCKET>/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalArn": "arn:aws:iam::<ACCOUNT_B>:role/break-glass-role"
        }
      }
    }
  ]
}
```

Apply using `aws organizations create-policy` and `attach-policy` from the Organization management account.

> **When is this worth adding?** SCPs are most valuable for Account A, where IAM deny policies can be overridden by root or IAM admins. For Account B, Object Lock Compliance mode already provides root-proof protection — SCPs add an extra layer but are not essential.

---

## Step 5: Set Up Point-in-Time Recovery with S3 Metadata and Athena

S3 does not have native PITR like DynamoDB. We build it using **S3 Metadata** (near real-time change tracking via Apache Iceberg tables), **Athena** (SQL queries to find the right versions), and **S3 Batch Operations** (bulk restore).

S3 Metadata provides two table types:
- **Journal table**: Records every object change (CREATE, UPDATE_METADATA, DELETE) in **near real-time**
- **Live inventory table**: Maintains the **current state** of all objects, refreshed within ~1 hour

### 5.1 Ops Buckets

The ops buckets (`${OPS_BUCKET_A}` and `${OPS_BUCKET_B}`) were already created in **Step 2.4**. S3 Metadata tables are stored automatically in AWS managed table buckets — no additional bucket is needed for metadata.

### 5.2 Enable S3 Metadata (Both Accounts)

Enable S3 Metadata with both **journal table** (change tracking) and **live inventory table** (current state snapshot) on each bucket. Set journal record expiration to 90 days to match the recovery window.

> **Note:** The `create-bucket-metadata-configuration` command (V2 API) requires **AWS CLI v2.27.51 or later**. If you receive an `Invalid choice` error, check your version with `aws --version` and [upgrade to the latest AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

**Account A — Source Bucket:**

```bash
cat > /tmp/metadata-config.json << 'EOF'
{
  "JournalTableConfiguration": {
    "RecordExpiration": {
      "Expiration": "ENABLED",
      "Days": 90
    }
  },
  "InventoryTableConfiguration": {
    "ConfigurationState": "ENABLED"
  }
}
EOF

aws s3api create-bucket-metadata-configuration \
  --bucket ${SOURCE_BUCKET} \
  --metadata-configuration file:///tmp/metadata-config.json \
  --region ${REGION} \
  --profile account-a
```

**Account B — Destination Bucket:**

```bash
aws s3api create-bucket-metadata-configuration \
  --bucket ${DEST_BUCKET} \
  --metadata-configuration file:///tmp/metadata-config.json \
  --region ${REGION} \
  --profile account-b
```

Verify:

```bash
aws s3api get-bucket-metadata-configuration \
  --bucket ${SOURCE_BUCKET} \
  --region ${REGION} \
  --profile account-a
```

> **Why S3 Metadata on both accounts?** If Account A is compromised, the attacker can delete Account A's metadata tables. Account B's metadata provides an independent record for PITR even in the worst-case scenario.

> **Note**: When the live inventory table is first enabled, it goes through a **backfilling** process — Amazon S3 scans the bucket to retrieve initial metadata for all existing objects. This takes **15 minutes to several hours** depending on bucket size (much faster than S3 Inventory's 48-hour wait). After backfilling completes, updates are reflected within ~1 hour. Backfilling incurs a one-time cost of **$0.30 per million objects** — for 500 million objects, that is ~$150 per bucket ($300 total for both accounts).

### 5.3 Integrate with Athena

S3 Metadata tables are stored as Apache Iceberg tables in an AWS managed table bucket (`aws-s3`). To query them with Athena, integrate the table bucket with AWS analytics services.

> **Important**: Perform these steps in **both Account A and Account B**. Account B's analytics integration is required for the disaster recovery scenario in Step 6.4, where you query Account B's metadata tables directly from Account B's Athena.

1. Open the **S3 console** → **Table buckets** → select the `aws-s3` table bucket
2. Under **Integrations**, choose **Enable integration with AWS analytics services**
3. In **Lake Formation**, grant `SELECT` permissions on the metadata table namespace to the IAM user/role that will run Athena queries

After integration, query metadata tables in Athena using:
- **Catalog**: `s3tablescatalog/aws-s3`
- **Database**: `b_<bucket-name>` (periods in bucket names become underscores)
- **Tables**: `journal` (change log) or `inventory` (current state)

Verify with a test query:

```sql
-- In Athena, set Catalog to: s3tablescatalog/aws-s3
-- Set Database to: b_my-source-bucket

SELECT record_type, count(*) as cnt
FROM "s3tablescatalog/aws-s3"."b_my-source-bucket"."journal"
WHERE record_timestamp > current_date - interval '1' day
GROUP BY record_type;
```

### 5.4 PITR Query: Find All Objects at a Specific Point in Time

This is the core query. Given a target timestamp, it reconstructs the **complete bucket state at that point in time** by combining the **inventory table** (baseline of all objects) with **journal entries** (recent changes). This hybrid approach is necessary because the journal only records changes after S3 Metadata was enabled — pre-existing objects that were never modified would be missing from a journal-only query.

```sql
-- Target: restore to 2024-07-15 14:00:00 UTC
-- Hybrid approach: inventory (baseline) + journal (recent changes)

-- Step 1: Determine the boundary between inventory and journal data
WITH inventory_time_cte AS (
    SELECT COALESCE(inventory_time_from_property, inventory_time_default)
           AS inventory_time
    FROM (
        SELECT *
        FROM (VALUES (TIMESTAMP '2024-12-01 00:00')) AS T(inventory_time_default)
        LEFT JOIN (
            SELECT from_unixtime(CAST(value AS BIGINT) / 1000.0)
                   AS inventory_time_from_property
            FROM "s3tablescatalog/aws-s3"."b_my-source-bucket"."journal$properties"
            WHERE key = 'aws.s3metadata.oldest-uncoalesced-record-timestamp'
            LIMIT 1
        ) ON TRUE
    )
),

-- Step 2: Build working set from inventory (baseline) + journal (delta)
working_set AS (
    -- Baseline: all objects from inventory snapshot
    SELECT bucket, key, sequence_number, version_id, is_delete_marker,
           last_modified_date, size, storage_class,
           CAST(NULL AS varchar) AS record_type
    FROM "s3tablescatalog/aws-s3"."b_my-source-bucket"."inventory" i
    WHERE i.last_modified_date <= TIMESTAMP '2024-07-15 14:00:00'

    UNION ALL

    -- Delta: journal entries since inventory boundary (with 15-min overlap buffer)
    SELECT bucket, key, sequence_number, version_id, is_delete_marker,
           COALESCE(last_modified_date, record_timestamp) AS last_modified_date,
           size, storage_class, record_type
    FROM "s3tablescatalog/aws-s3"."b_my-source-bucket"."journal" j
    CROSS JOIN inventory_time_cte t
    WHERE j.last_modified_date > (t.inventory_time - interval '15' minute)
      AND j.last_modified_date <= TIMESTAMP '2024-07-15 14:00:00'
),

-- Step 3: For each (key, version_id), find the tip of the version stack
version_stacks AS (
    SELECT *,
      LEAD(sequence_number, 1) OVER (
        PARTITION BY bucket, key, coalesce(version_id, '')
        ORDER BY sequence_number ASC
      ) AS next_sequence_number
    FROM working_set
),
version_tips AS (
    SELECT * FROM version_stacks WHERE next_sequence_number IS NULL
),

-- Step 4: Exclude permanently deleted versions (keep delete markers and live versions)
existing_versions AS (
    SELECT * FROM version_tips
    WHERE record_type IS NULL        -- inventory rows: always keep
       OR record_type != 'DELETE'    -- journal non-delete events: keep
       OR is_delete_marker = TRUE    -- journal delete markers (soft deletes): keep
    -- Filters out: journal permanent deletes (record_type='DELETE', is_delete_marker=FALSE)
),

-- Step 5: For each key, find the latest existing version
with_is_latest AS (
    SELECT *,
      sequence_number = MAX(sequence_number) OVER (
        PARTITION BY bucket, key
      ) AS is_latest_version
    FROM existing_versions
)

-- Step 6: Pick the latest version per key, exclude delete markers
SELECT
  bucket,
  key,
  version_id,
  size,
  storage_class
FROM with_is_latest
WHERE is_latest_version = TRUE
  AND COALESCE(is_delete_marker, FALSE) = FALSE;
```

**How to read this query:**
- **Step 1** — `journal$properties` provides the `oldest-uncoalesced-record-timestamp`, marking where the inventory snapshot ends and journal records begin. The hardcoded `2024-12-01 00:00` is only a fallback default — the query automatically reads the actual boundary from `journal$properties` via `COALESCE()`, so in most cases you do not need to change this value. If `journal$properties` does not yet contain the timestamp (e.g., the inventory backfill is still in progress), replace the fallback with the approximate time you enabled S3 Metadata on the bucket.
- **Step 2** — Inventory provides the baseline of **all objects** (including those that predate S3 Metadata). Journal adds only recent changes since the inventory boundary, with a 15-minute overlap buffer to prevent gaps.
- **Step 3** — `PARTITION BY bucket, key, coalesce(version_id, '')` builds a "version stack" for each object version ([AWS documentation pattern](https://docs.aws.amazon.com/AmazonS3/latest/userguide/metadata-tables-example-queries.html)). `LEAD()` finds the next event; NULL means this row is the tip (most recent event).
- **Step 4** — Remove permanently deleted versions (`record_type='DELETE'` with `is_delete_marker=FALSE` in journal) so that Lifecycle noncurrent version expiration doesn't mask the current version. Inventory rows (`record_type IS NULL`) are always kept.
- **Step 5–6** — Among surviving versions, pick the latest per key and exclude delete markers (object was deleted at target time).

> **Why inventory + journal?** The journal only records changes **after S3 Metadata was enabled**. Pre-existing objects that were never modified have no journal entry. The inventory table includes all objects via backfill, making it essential for complete PITR. Additionally, journal records expire (90 days in this architecture), so older CREATE events are eventually lost. The inventory table provides the stable baseline that the journal alone cannot.

> **Performance benefit**: For a 500M-object bucket, this approach reads the bulk of state from the pre-aggregated inventory and scans only the recent journal delta, reducing Athena costs by **90%+** compared to a full journal scan.

### 5.5 Variant: Include Deleted Objects (Restore Them Too)

If ransomware deleted objects before overwriting, you may want to restore those as well. This query extends 5.4 by also recovering objects whose latest version is a delete marker, finding their most recent non-deleted version:

```sql
-- Same hybrid working_set as 5.4 (inventory baseline + journal delta)
WITH inventory_time_cte AS (
    SELECT COALESCE(inventory_time_from_property, inventory_time_default)
           AS inventory_time
    FROM (
        SELECT *
        FROM (VALUES (TIMESTAMP '2024-12-01 00:00')) AS T(inventory_time_default)
        LEFT JOIN (
            SELECT from_unixtime(CAST(value AS BIGINT) / 1000.0)
                   AS inventory_time_from_property
            FROM "s3tablescatalog/aws-s3"."b_my-source-bucket"."journal$properties"
            WHERE key = 'aws.s3metadata.oldest-uncoalesced-record-timestamp'
            LIMIT 1
        ) ON TRUE
    )
),
working_set AS (
    SELECT bucket, key, sequence_number, version_id, is_delete_marker,
           last_modified_date, size,
           CAST(NULL AS varchar) AS record_type
    FROM "s3tablescatalog/aws-s3"."b_my-source-bucket"."inventory" i
    WHERE i.last_modified_date <= TIMESTAMP '2024-07-15 14:00:00'
    UNION ALL
    SELECT bucket, key, sequence_number, version_id, is_delete_marker,
           COALESCE(last_modified_date, record_timestamp) AS last_modified_date, size,
           record_type
    FROM "s3tablescatalog/aws-s3"."b_my-source-bucket"."journal" j
    CROSS JOIN inventory_time_cte t
    WHERE j.last_modified_date > (t.inventory_time - interval '15' minute)
      AND j.last_modified_date <= TIMESTAMP '2024-07-15 14:00:00'
),
version_stacks AS (
    SELECT *,
      LEAD(sequence_number, 1) OVER (
        PARTITION BY bucket, key, coalesce(version_id, '')
        ORDER BY sequence_number ASC
      ) AS next_sequence_number
    FROM working_set
),
version_tips AS (
    SELECT * FROM version_stacks WHERE next_sequence_number IS NULL
),
existing_versions AS (
    SELECT * FROM version_tips
    WHERE record_type IS NULL
       OR record_type != 'DELETE'
       OR is_delete_marker = TRUE
),
with_is_latest AS (
    SELECT *,
      sequence_number = MAX(sequence_number) OVER (PARTITION BY bucket, key) AS is_latest_version
    FROM existing_versions
),
-- Objects alive at target time
alive AS (
    SELECT bucket, key, version_id, size
    FROM with_is_latest
    WHERE is_latest_version = TRUE
      AND COALESCE(is_delete_marker, FALSE) = FALSE
),
-- Objects deleted at target time — recover the most recent non-delete-marker version
deleted_keys AS (
    SELECT key FROM with_is_latest
    WHERE is_latest_version = TRUE
      AND is_delete_marker = TRUE
),
deleted_restore AS (
    SELECT e.bucket, e.key, e.version_id, e.size,
      ROW_NUMBER() OVER (PARTITION BY e.key ORDER BY e.sequence_number DESC) AS restore_rn
    FROM existing_versions e
    WHERE e.key IN (SELECT key FROM deleted_keys)
      AND COALESCE(e.is_delete_marker, FALSE) = FALSE
)
SELECT bucket, key, version_id, size FROM alive
UNION ALL
SELECT bucket, key, version_id, size FROM deleted_restore WHERE restore_rn = 1;
```

### 5.6 Current State Query (Live Inventory Table)

For quick verification of the current bucket state — e.g., confirming how many objects exist right now — use the live inventory table:

```sql
SELECT storage_class, count(*) as object_count, sum(size) as total_bytes
FROM "s3tablescatalog/aws-s3"."b_my-source-bucket"."inventory"
WHERE NOT is_delete_marker
GROUP BY storage_class;
```

The live inventory table is refreshed within ~1 hour and does not require any manual setup beyond enabling it in Step 5.2.

---

## Step 6: Execute a Point-in-Time Recovery

When an incident occurs, follow this runbook to restore data to a specific point in time.

> **Before you begin**: The examples below use `2024-07-15 14:00:00` as the PITR target timestamp and `2024-07-15T14` in S3 paths (e.g., `pitr-manifests/2024-07-15T14/`). When restoring to a different point in time, update **both** the `TIMESTAMP` values in the SQL query **and** the S3 paths in the bash commands to match your target date.

### 6.1 Generate the PITR Manifest

Run the PITR query in Athena and use the automatically generated CSV result file as the manifest. Athena saves every SELECT result as a CSV file in the workgroup's query result location.

> **Note**: S3 Metadata tables live in `s3tablescatalog`, which only supports Iceberg formats (AVRO, ORC, PARQUET) — CTAS with `TEXTFILE` format is not supported when querying from this catalog. Instead, we run a plain `SELECT` and post-process the result CSV.

```sql
-- Run in Athena with Catalog: s3tablescatalog/aws-s3, Database: b_my-source-bucket
WITH inventory_time_cte AS (
    SELECT COALESCE(inventory_time_from_property, inventory_time_default)
           AS inventory_time
    FROM (
        SELECT *
        FROM (VALUES (TIMESTAMP '2024-12-01 00:00')) AS T(inventory_time_default)
        LEFT JOIN (
            SELECT from_unixtime(CAST(value AS BIGINT) / 1000.0)
                   AS inventory_time_from_property
            FROM "s3tablescatalog/aws-s3"."b_my-source-bucket"."journal$properties"
            WHERE key = 'aws.s3metadata.oldest-uncoalesced-record-timestamp'
            LIMIT 1
        ) ON TRUE
    )
),
working_set AS (
    SELECT bucket, key, sequence_number, version_id, is_delete_marker,
           last_modified_date,
           CAST(NULL AS varchar) AS record_type
    FROM "s3tablescatalog/aws-s3"."b_my-source-bucket"."inventory" i
    WHERE i.last_modified_date <= TIMESTAMP '2024-07-15 14:00:00'
    UNION ALL
    SELECT bucket, key, sequence_number, version_id, is_delete_marker,
           COALESCE(last_modified_date, record_timestamp) AS last_modified_date,
           record_type
    FROM "s3tablescatalog/aws-s3"."b_my-source-bucket"."journal" j
    CROSS JOIN inventory_time_cte t
    WHERE j.last_modified_date > (t.inventory_time - interval '15' minute)
      AND j.last_modified_date <= TIMESTAMP '2024-07-15 14:00:00'
),
version_stacks AS (
    SELECT *,
      LEAD(sequence_number, 1) OVER (
        PARTITION BY bucket, key, coalesce(version_id, '')
        ORDER BY sequence_number ASC
      ) AS next_sequence_number
    FROM working_set
),
version_tips AS (
    SELECT * FROM version_stacks WHERE next_sequence_number IS NULL
),
existing_versions AS (
    SELECT * FROM version_tips
    WHERE record_type IS NULL
       OR record_type != 'DELETE'
       OR is_delete_marker = TRUE
),
with_is_latest AS (
    SELECT *,
      sequence_number = MAX(sequence_number) OVER (
        PARTITION BY bucket, key
      ) AS is_latest_version
    FROM existing_versions
)
SELECT bucket, key, version_id
FROM with_is_latest
WHERE is_latest_version = TRUE
  AND COALESCE(is_delete_marker, FALSE) = FALSE;
```

After the query completes, download the result CSV from the Athena query result location, strip the header row, and upload it as the manifest:

```bash
# Copy the result S3 path from: Athena console > Recent queries > select your query > Output location
# The path varies by workgroup settings (e.g., Unsaved/2024/07/15/<query-id>.csv or query-results/<query-id>.csv)
QUERY_RESULT="s3://${OPS_BUCKET_A}/Unsaved/2024/07/15/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.csv"  # ← replace with actual Output location

aws s3 cp "${QUERY_RESULT}" /tmp/pitr-result.csv --profile account-a
tail -n +2 /tmp/pitr-result.csv > /tmp/pitr-manifest.csv    # strip header row
aws s3 cp /tmp/pitr-manifest.csv s3://${OPS_BUCKET_A}/pitr-manifests/2024-07-15T14/manifest.csv --profile account-a
```

> **Comma in S3 keys**: Athena's CSV output quotes field values, so commas within S3 keys are handled correctly. However, if your keys contain both commas and double quotes, verify the manifest file before proceeding.

### 6.2 Create the Batch Operations Restore Job

Create a restore bucket first, then run the Batch Operations job to copy the specific versions:

```bash
# Create a restore bucket (for us-east-1, omit --create-bucket-configuration)
aws s3api create-bucket \
  --bucket ${RESTORE_BUCKET_A} \
  --region ${REGION} \
  --create-bucket-configuration LocationConstraint=${REGION} \
  --profile account-a

# Create the Batch Operations IAM role
cat > /tmp/batch-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "batchoperations.s3.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name s3-batch-restore-role \
  --assume-role-policy-document file:///tmp/batch-trust.json \
  --profile account-a

cat > /tmp/batch-perms.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:GetObjectTagging",
        "s3:GetObjectVersionTagging"
      ],
      "Resource": "arn:aws:s3:::${SOURCE_BUCKET}/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectTagging"
      ],
      "Resource": "arn:aws:s3:::${RESTORE_BUCKET_A}/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion"
      ],
      "Resource": "arn:aws:s3:::${OPS_BUCKET_A}/pitr-manifests/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::${OPS_BUCKET_A}/batch-reports/*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name s3-batch-restore-role \
  --policy-name batch-restore-perms \
  --policy-document file:///tmp/batch-perms.json \
  --profile account-a
```

Now create the Batch Operations job:

```bash
# Get the manifest file ETag
MANIFEST_ETAG=$(aws s3api head-object \
  --bucket ${OPS_BUCKET_A} \
  --key pitr-manifests/2024-07-15T14/manifest.csv \
  --query ETag --output text \
  --profile account-a | tr -d '"')

# Create the Batch Operations job
aws s3control create-job \
  --account-id ${ACCOUNT_A} \
  --operation '{
    "S3PutObjectCopy": {
      "TargetResource": "arn:aws:s3:::'${RESTORE_BUCKET_A}'",
      "StorageClass": "STANDARD"
    }
  }' \
  --manifest '{
    "Spec": {
      "Format": "S3BatchOperations_CSV_20180820",
      "Fields": ["Bucket", "Key", "VersionId"]
    },
    "Location": {
      "ObjectArn": "arn:aws:s3:::'${OPS_BUCKET_A}'/pitr-manifests/2024-07-15T14/manifest.csv",
      "ETag": "'${MANIFEST_ETAG}'"
    }
  }' \
  --report '{
    "Bucket": "arn:aws:s3:::'${OPS_BUCKET_A}'",
    "Prefix": "batch-reports/pitr-2024-07-15",
    "Format": "Report_CSV_20180820",
    "Enabled": true,
    "ReportScope": "AllTasks"
  }' \
  --priority 10 \
  --role-arn arn:aws:iam::${ACCOUNT_A}:role/s3-batch-restore-role \
  --description "PITR restore to 2024-07-15T14:00:00" \
  --confirmation-required \
  --region ${REGION} \
  --profile account-a
```

### 6.3 Confirm and Monitor the Restore Job

Since we used `--confirmation-required`, the job starts in a **Suspended** state. Confirm it to begin execution:

```bash
# Get the job ID from the create-job output
JOB_ID="your-job-id-here"

# Wait for "Suspended" status, then confirm the job
aws s3control describe-job \
  --account-id ${ACCOUNT_A} \
  --job-id ${JOB_ID} \
  --query 'Job.Status' \
  --region ${REGION} \
  --profile account-a

# Confirm the job to start execution
aws s3control update-job-status \
  --account-id ${ACCOUNT_A} \
  --job-id ${JOB_ID} \
  --requested-job-status Ready \
  --region ${REGION} \
  --profile account-a

# Monitor progress:
aws s3control describe-job \
  --account-id ${ACCOUNT_A} \
  --job-id ${JOB_ID} \
  --query 'Job.{Status:Status,Succeeded:ProgressSummary.NumberOfTasksSucceeded,Failed:ProgressSummary.NumberOfTasksFailed,Total:ProgressSummary.TotalNumberOfTasks}' \
  --region ${REGION} \
  --profile account-a
```

### 6.4 Disaster Recovery: Restore from Account B

If Account A is compromised, run the same process from Account B using Account B's metadata.

**Pre-requisite**: Create a restore bucket and the Batch Operations IAM role in Account B (same structure as Account A's role in Step 6.2, but referencing Account B's buckets):

```bash
# Create a restore bucket in Account B (for us-east-1, omit --create-bucket-configuration)
aws s3api create-bucket \
  --bucket ${RESTORE_BUCKET_B} \
  --region ${REGION} \
  --create-bucket-configuration LocationConstraint=${REGION} \
  --profile account-b

# Create Batch Ops role in Account B (same trust policy as Account A)
aws iam create-role \
  --role-name s3-batch-restore-role \
  --assume-role-policy-document file:///tmp/batch-trust.json \
  --profile account-b

cat > /tmp/batch-perms-b.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:GetObjectVersion", "s3:RestoreObject"],
      "Resource": "arn:aws:s3:::${DEST_BUCKET}/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:PutObjectTagging"],
      "Resource": "arn:aws:s3:::${RESTORE_BUCKET_B}/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:GetObjectVersion"],
      "Resource": "arn:aws:s3:::${OPS_BUCKET_B}/pitr-manifests/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::${OPS_BUCKET_B}/batch-reports/*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name s3-batch-restore-role \
  --policy-name batch-restore-perms \
  --policy-document file:///tmp/batch-perms-b.json \
  --profile account-b
```

Generate the PITR manifest from Account B's journal (same SELECT approach as Step 6.1):

```sql
-- Run in Account B's Athena
-- Catalog: s3tablescatalog/aws-s3, Database: b_my-backup-bucket
WITH inventory_time_cte AS (
    SELECT COALESCE(inventory_time_from_property, inventory_time_default)
           AS inventory_time
    FROM (
        SELECT *
        FROM (VALUES (TIMESTAMP '2024-12-01 00:00')) AS T(inventory_time_default)
        LEFT JOIN (
            SELECT from_unixtime(CAST(value AS BIGINT) / 1000.0)
                   AS inventory_time_from_property
            FROM "s3tablescatalog/aws-s3"."b_my-backup-bucket"."journal$properties"
            WHERE key = 'aws.s3metadata.oldest-uncoalesced-record-timestamp'
            LIMIT 1
        ) ON TRUE
    )
),
working_set AS (
    SELECT bucket, key, sequence_number, version_id, is_delete_marker,
           last_modified_date,
           CAST(NULL AS varchar) AS record_type
    FROM "s3tablescatalog/aws-s3"."b_my-backup-bucket"."inventory" i
    WHERE i.last_modified_date <= TIMESTAMP '2024-07-15 14:00:00'
    UNION ALL
    SELECT bucket, key, sequence_number, version_id, is_delete_marker,
           COALESCE(last_modified_date, record_timestamp) AS last_modified_date,
           record_type
    FROM "s3tablescatalog/aws-s3"."b_my-backup-bucket"."journal" j
    CROSS JOIN inventory_time_cte t
    WHERE j.last_modified_date > (t.inventory_time - interval '15' minute)
      AND j.last_modified_date <= TIMESTAMP '2024-07-15 14:00:00'
),
version_stacks AS (
    SELECT *,
      LEAD(sequence_number, 1) OVER (
        PARTITION BY bucket, key, coalesce(version_id, '')
        ORDER BY sequence_number ASC
      ) AS next_sequence_number
    FROM working_set
),
version_tips AS (
    SELECT * FROM version_stacks WHERE next_sequence_number IS NULL
),
existing_versions AS (
    SELECT * FROM version_tips
    WHERE record_type IS NULL
       OR record_type != 'DELETE'
       OR is_delete_marker = TRUE
),
with_is_latest AS (
    SELECT *,
      sequence_number = MAX(sequence_number) OVER (
        PARTITION BY bucket, key
      ) AS is_latest_version
    FROM existing_versions
)
SELECT bucket, key, version_id
FROM with_is_latest
WHERE is_latest_version = TRUE
  AND COALESCE(is_delete_marker, FALSE) = FALSE;
```

Since the destination uses **Glacier Deep Archive**, objects must be restored before copying. This requires a two-step Batch Operations process.

First, download the Athena query result, strip the header, and upload as the manifest (same process as Step 6.1):

```bash
# Copy the result S3 path from: Athena console > Recent queries > select your query > Output location
QUERY_RESULT_B="s3://${OPS_BUCKET_B}/Unsaved/2024/07/15/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.csv"  # ← replace with actual Output location
aws s3 cp "${QUERY_RESULT_B}" /tmp/pitr-result-dr.csv --profile account-b
tail -n +2 /tmp/pitr-result-dr.csv > /tmp/pitr-manifest-dr.csv
aws s3 cp /tmp/pitr-manifest-dr.csv s3://${OPS_BUCKET_B}/pitr-manifests/dr-2024-07-15/manifest.csv --profile account-b

# Get the manifest ETag
MANIFEST_ETAG=$(aws s3api head-object \
  --bucket ${OPS_BUCKET_B} \
  --key pitr-manifests/dr-2024-07-15/manifest.csv \
  --query ETag --output text \
  --profile account-b | tr -d '"')
```

**Step 1: Initiate restore (Bulk = 48h, Standard = 12h)**

```bash
# Create the Batch Operations restore-from-archive job
aws s3control create-job \
  --account-id ${ACCOUNT_B} \
  --operation '{
    "S3InitiateRestoreObject": {
      "ExpirationInDays": 7,
      "GlacierJobTier": "BULK"
    }
  }' \
  --manifest '{
    "Spec": {
      "Format": "S3BatchOperations_CSV_20180820",
      "Fields": ["Bucket", "Key", "VersionId"]
    },
    "Location": {
      "ObjectArn": "arn:aws:s3:::'${OPS_BUCKET_B}'/pitr-manifests/dr-2024-07-15/manifest.csv",
      "ETag": "'${MANIFEST_ETAG}'"
    }
  }' \
  --report '{
    "Bucket": "arn:aws:s3:::'${OPS_BUCKET_B}'",
    "Prefix": "batch-reports/restore-archive",
    "Format": "Report_CSV_20180820",
    "Enabled": true,
    "ReportScope": "AllTasks"
  }' \
  --priority 10 \
  --role-arn arn:aws:iam::${ACCOUNT_B}:role/s3-batch-restore-role \
  --description "Restore from Deep Archive - BULK 48h" \
  --confirmation-required \
  --region ${REGION} \
  --profile account-b
```

Confirm the job to start execution, then monitor until all restores complete (same process as Step 6.3):

```bash
aws s3control update-job-status \
  --account-id ${ACCOUNT_B} \
  --job-id <RESTORE_JOB_ID> \
  --requested-job-status Ready \
  --region ${REGION} \
  --profile account-b
```

> **Choosing retrieval tier**: Use `BULK` (48 hours) for full-scale DR — it is the cheapest option and actually costs less than Glacier IR. Use `STANDARD` (12 hours) for smaller, time-sensitive restores where faster access justifies the higher per-request cost.

> **Important — two-phase restore process**: The Batch Operations job for `S3InitiateRestoreObject` completes when all restore **requests** have been submitted — this does NOT mean the objects are ready to access. The actual Glacier Deep Archive restore takes **12 hours (Standard)** or **48 hours (Bulk)** after the job completes. You must wait for the restore to finish before proceeding to Step 2.

**Monitor Glacier restore progress:**

S3 sends an `s3:ObjectRestore:Completed` event via EventBridge for **each** object that finishes restoring. Set up an EventBridge rule to count completions and receive a notification when all objects are ready.

**Option A: EventBridge + CloudWatch counter (recommended for full visibility)**

Create an EventBridge rule that counts `ObjectRestore:Completed` events via a CloudWatch metric. When the metric count matches the total number of objects in the manifest, all restores are complete.

```bash
# 1. Enable EventBridge notifications on the backup bucket
aws s3api put-bucket-notification-configuration \
  --bucket ${DEST_BUCKET} \
  --notification-configuration '{"EventBridgeConfiguration": {}}' \
  --profile account-b

# 2. Create EventBridge rule to count restore completions
aws events put-rule \
  --name "s3-restore-completed" \
  --event-pattern '{
    "source": ["aws.s3"],
    "detail-type": ["Object Restore Completed"],
    "detail": {"bucket": {"name": ["'${DEST_BUCKET}'"]}}
  }' \
  --region ${REGION} \
  --profile account-b

# 3. (Optional) Add an SNS target to receive email notification per object
# aws events put-targets --rule "s3-restore-completed" --targets '[{"Id":"sns","Arn":"arn:aws:sns:'${REGION}':'${ACCOUNT_B}':restore-notify"}]'
```

Monitor the restore count in **CloudWatch** → **Metrics** → **Events** → **TriggeredRules** for the `s3-restore-completed` rule, and compare it with the total manifest line count:

```bash
# Restore-completed count (TriggeredRules metric for the last 1 hour)
# macOS: replace -d '1 hour ago' with -v-1H
aws cloudwatch get-metric-statistics \
  --namespace "AWS/Events" \
  --metric-name "TriggeredRules" \
  --dimensions Name=RuleName,Value=s3-restore-completed \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 3600 \
  --statistics Sum \
  --region ${REGION} \
  --profile account-b \
  --query 'Datapoints[0].Sum' --output text

# Total objects to restore
wc -l < /tmp/pitr-manifest-dr.csv
```

> **Note:** The EventBridge rule must be created **before** the restore completes. If the rule is created after restores have already finished, the `TriggeredRules` metric will return `None` because past events are not captured retroactively. In that case, use Option B below to verify restore status.

**Option B: Spot-check multiple samples from the manifest**

For a quick check without setting up EventBridge, verify objects from the beginning, middle, and end of the manifest:

```bash
TOTAL=$(wc -l < /tmp/pitr-manifest-dr.csv)
for LINE in 1 $((TOTAL/2)) $TOTAL; do
  KEY=$(sed -n "${LINE}p" /tmp/pitr-manifest-dr.csv | cut -d',' -f2 | tr -d '"')
  VER=$(sed -n "${LINE}p" /tmp/pitr-manifest-dr.csv | cut -d',' -f3 | tr -d '"')
  STATUS=$(aws s3api head-object --bucket ${DEST_BUCKET} --key "${KEY}" \
    --version-id "${VER}" --query 'Restore' --output text --profile account-b 2>/dev/null)
  echo "Line ${LINE}/${TOTAL}: ${STATUS}"
done
```

- `ongoing-request="true"` → restore still in progress, **wait and retry later**
- `ongoing-request="false", expiry-date="..."` → restore complete

When all samples show `ongoing-request="false"`, proceed to Step 2. For large restores (millions of objects), Option A provides definitive confirmation.

**Step 2: Copy restored objects to a new bucket (after restore completes)**

```bash
# Re-fetch the manifest ETag (in case the shell session has changed since Step 1)
MANIFEST_ETAG=$(aws s3api head-object \
  --bucket ${OPS_BUCKET_B} \
  --key pitr-manifests/dr-2024-07-15/manifest.csv \
  --query ETag --output text \
  --profile account-b | tr -d '"')

aws s3control create-job \
  --account-id ${ACCOUNT_B} \
  --operation '{
    "S3PutObjectCopy": {
      "TargetResource": "arn:aws:s3:::'${RESTORE_BUCKET_B}'",
      "StorageClass": "STANDARD"
    }
  }' \
  --manifest '{
    "Spec": {
      "Format": "S3BatchOperations_CSV_20180820",
      "Fields": ["Bucket", "Key", "VersionId"]
    },
    "Location": {
      "ObjectArn": "arn:aws:s3:::'${OPS_BUCKET_B}'/pitr-manifests/dr-2024-07-15/manifest.csv",
      "ETag": "'${MANIFEST_ETAG}'"
    }
  }' \
  --report '{
    "Bucket": "arn:aws:s3:::'${OPS_BUCKET_B}'",
    "Prefix": "batch-reports/copy-restored",
    "Format": "Report_CSV_20180820",
    "Enabled": true,
    "ReportScope": "AllTasks"
  }' \
  --priority 10 \
  --role-arn arn:aws:iam::${ACCOUNT_B}:role/s3-batch-restore-role \
  --description "Copy restored objects to restore bucket" \
  --confirmation-required \
  --region ${REGION} \
  --profile account-b
```

Confirm and monitor this job as well:

```bash
aws s3control update-job-status \
  --account-id ${ACCOUNT_B} \
  --job-id <COPY_JOB_ID> \
  --requested-job-status Ready \
  --region ${REGION} \
  --profile account-b
```

> **Verify DR completion**: Open the **S3 console** → **Batch Operations** → select the copy job. When the status shows **Complete** and the **Successful** count matches the **Total** count, the DR recovery is finished. Verify the restored objects in the `${RESTORE_BUCKET_B}` bucket.

You can also verify from the CLI:

```bash
# Check copy job status (replace <COPY_JOB_ID> with the job ID returned above)
aws s3control describe-job \
  --account-id ${ACCOUNT_B} \
  --job-id <COPY_JOB_ID> \
  --region ${REGION} \
  --profile account-b \
  --query 'Job.{Status:Status,Total:ProgressSummary.TotalNumberOfTasks,Succeeded:ProgressSummary.NumberOfTasksSucceeded,Failed:ProgressSummary.NumberOfTasksFailed}'

# (Optional) Verify objects in the restore bucket
# For large-scale buckets (hundreds of millions of objects), this command may
# incur significant LIST request costs and take a long time to complete.
# In that case, rely on the describe-job output above instead.
# aws s3 ls s3://${RESTORE_BUCKET_B}/ --recursive --summarize --profile account-b
```

When `Status` is `Complete` and `Succeeded` matches `Total`, the DR recovery is finished.

---

## Recovery Cost Estimates

**From Source (Account A)** — noncurrent versions in Glacier IR, immediate access:

| Recovery Scale | Batch Ops | GET Requests (Glacier IR) | Data Retrieval (Glacier IR) | Athena | Total |
|---------------|-----------|--------------------------|----------------------------|--------|-------|
| 100 objects (accidental) | ~$0 | ~$0 | ~$0 | $0.03 | **~$0** |
| 1M objects (partial) | $1.25 | $10 | ~$300 | $0.03 | **~$311** |
| 500M objects (full DR) | $500 | $5,000 | $15,000 | $0.03 | **~$20,500** |

**From Destination (Account B)** — Deep Archive, restore required:

| Recovery Scale | Batch Ops (2 jobs) | Restore + Retrieval (Bulk, 48h) | Restore + Retrieval (Standard, 12h) | Athena | Total (Bulk) |
|---------------|-------------------|--------------------------------|-------------------------------------|--------|-------------|
| 100 objects | ~$1 | ~$0 | ~$0 | $0.03 | **~$1** |
| 1M objects (10 TB) | $2.50 | ~$50 | ~$300 | $0.03 | **~$53** |
| 500M objects (full DR) | $1,001 | $13,750 | $60,000 | $0.03 | **~$14,751** |

---

## PITR Precision and Limitations

| Factor | Detail |
|--------|--------|
| **Recovery granularity** | Any timestamp — journal records changes in near real-time |
| **Initial setup time** | Live inventory backfill: 15 minutes to several hours (vs. 48 hours for S3 Inventory) |
| **Journal record retention** | Configurable — set to 90 days in this architecture (minimum 7 days) |
| **RPO (Recovery Point Objective)** | **Near real-time** — journal captures every object mutation as it happens |

> **Note**: For PITR queries spanning a long time range with high object churn, journal table scans can be expensive. Always include a `record_timestamp` range filter to limit the scan window. For current state queries, use the live inventory table instead of scanning the full journal.

---

## Implementation Priority

| Phase | Actions | Timeline | Cost Impact |
|-------|---------|----------|-------------|
| **Immediate** | Enable Versioning, IAM deny policies for versioning protection | Day 1 | $0 |
| **Week 1** | Create Account B, configure destination bucket with Object Lock | Week 1 | $0 until replication starts |
| **Week 2** | Set up cross-account SRR, enable S3 Metadata on both accounts | Week 2 | ~$600/mo begins |
| **Week 3** | Run Batch Replication for existing objects, create Athena tables | Week 3 | One-time replication request costs |
| **Week 4** | Test PITR restore (partial), document runbook, enable GuardDuty | Week 4 | ~$15 for test restore |

---

## Final Monthly Cost Summary

| Component | Monthly Cost |
|-----------|-------------|
| Destination storage (Deep Archive, 500 TB) | $495 |
| S3 Metadata journal — Account A (~0.1–1% daily churn) | $5–45 |
| S3 Metadata journal — Account B (~0.1–1% daily churn) | $5–45 |
| S3 Metadata live inventory (< 1B objects) | $0 |
| Data transfer (same-region SRR) | $0 |
| Object Lock / Versioning / IAM deny policies | $0 |
| CloudTrail (management events) + GuardDuty | ~$10 |
| **Total additional cost** | **~$515–595/mo** |

---

## Conclusion

By combining native AWS services — S3 Versioning, Cross-Account Same-Region Replication, Object Lock (Compliance mode), IAM deny policies, S3 Metadata, Athena, and S3 Batch Operations — we built a data protection architecture that:

1. **Defends against ransomware**: Object versioning preserves originals even when attackers overwrite data; IAM deny policies prevent version deletion
2. **Survives account takeover**: Cross-account isolation with Compliance mode Object Lock means even root cannot delete backup copies
3. **Prevents accidental damage**: Versioning + IAM deny policies create multiple safety nets (optionally strengthened with SCPs via AWS Organizations)
4. **Enables point-in-time recovery**: S3 Metadata journal tables provide near real-time, SQL-queryable change history to identify exact versions at any timestamp, and S3 Batch Operations restores them at scale

While AWS Backup for S3 remains the most feature-rich and operationally simple option, this architecture provides essential data protection at a fraction of the cost — ideal for organizations where AWS Backup pricing is prohibitive at scale. The destination uses Glacier Deep Archive for maximum storage savings, with Bulk retrieval (48 hours) for full-scale DR or Standard retrieval (12 hours) for targeted restores.

The key insight: **separate the protection layer (replication + locks) from the recovery layer (metadata + query + batch restore)**. The protection layer runs continuously at low cost. The recovery layer is mostly dormant — S3 Metadata journal costs scale with actual change volume, and the expensive operations (Athena queries, Batch Operations) only run when you actually need to recover.

---

*About the author: This architecture was designed for organizations managing large-scale S3 environments that need baseline data protection when managed backup service costs are prohibitive.*
