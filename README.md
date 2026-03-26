# 최소 비용, 최대 보호: AWS 네이티브 서비스로 S3 랜섬웨어 방어와 시점 복구 구축하기

*AWS 네이티브 서비스만으로 비용 효율적인 S3 데이터 보호 전략을 구축하는 실습 가이드*

---

## 소개

Amazon S3 환경에 **500TB의 데이터와 5억 개의 오브젝트**를 관리하고 있다고 가정해봅시다. **랜섬웨어**, **계정 탈취**, **실수로 인한 변경** 세 가지 위협으로부터 데이터를 보호해야 하며, 특정 시점으로의 복구(PITR) 기능도 필요합니다.

AWS Backup for S3는 중앙 집중식 관리, 연속 백업, 원활한 시점 복구를 제공하는 가장 포괄적인 솔루션입니다. 하지만 이 규모에서는 비용이 상당할 수 있습니다 — 5억 개 오브젝트의 오브젝트별 과금만으로도 큰 금액에 달합니다.

AWS Backup 비용이 부담되는 조직을 위한 대안은 없을까요? 이 글에서는 S3 Versioning, Cross-Account Replication, Object Lock, S3 Metadata, Athena, S3 Batch Operations 등 **AWS 네이티브 서비스를 활용한 최소 비용 데이터 보호 아키텍처**를 소개합니다. AWS Backup에 비해 기능이 제한적이고 운영 부담이 더 크지만, 동일한 500TB 환경에서 약 **월 $600**으로 필수적인 데이터 보호를 제공합니다.

## 비용 비교 요약

| 항목 | AWS Backup | 본 아키텍처 |
|------|-----------|------------|
| 백업 스토리지 (500TB) | ~$11,500/월 (periodic backup) | ~$500/월 (Deep Archive) |
| 오브젝트별 과금 (백업/복원) | 백업 유형에 따라 상이 — [AWS Backup 요금](https://aws.amazon.com/backup/pricing/) 참조 | **$0** |
| Object Lock / Versioning | — | **$0** (무료 기능) |
| S3 Metadata (PITR) | — | ~$50/월 |
| 데이터 전송 (동일 리전) | — | **$0** |
| **월 추가 비용 합계** | **~$12,000+** (스토리지만) | **~$600** |

> **AWS Backup 요금 참고**: AWS Backup for S3는 정기 백업과 연속 백업의 과금 방식이 다르며, 복원 시 오브젝트별 추가 요금이 발생합니다. 5억 개 오브젝트 규모에서는 이 비용이 상당할 수 있습니다. 구체적인 백업 구성에 따라 [최신 요금 페이지](https://aws.amazon.com/backup/pricing/)를 반드시 확인하세요.

## 아키텍처 개요

아키텍처는 **보호 계층**(공격 방어 및 생존)과 **복구 계층**(시점 검색 및 복원) 두 가지로 구성됩니다.

```
┌──────────────────────────────────────────────────────────────────┐
│  Account A (운영)                                                 │
│  ┌────────────────────────────────┐                              │
│  │  Source Bucket                  │                              │
│  │  - Versioning: Enabled         │── Same-Region ──┐            │
│  │  - IAM Deny Policies           │   Replication    │            │
│  │  - Lifecycle: noncurrent 관리   │                  │            │
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
│  Account B (백업 — 격리)                              │            │
│  ┌────────────────────────────────┐                  │            │
│  │  Destination Bucket            │◄─────────────────┘            │
│  │  - Object Lock: Compliance 180일│                              │
│  │  - Storage: Deep Archive       │                               │
│  │  - Bucket Policy: 삭제 거부     │                               │
│  │  - S3 Metadata: Journal +      │                               │
│  │    Live Inventory (Iceberg)    │                               │
│  └────────────────────────────────┘                               │
│  ┌────────────────────────────────┐                               │
│  │  Ops Bucket (B)                │                               │
│  │  - PITR manifests & reports    │                               │
│  └────────────────────────────────┘                               │
│  Object Lock Compliance: 180일간 변경 불가                          │
└───────────────────────────────────────────────────────────────────┘
```

### 위협별 복구 매트릭스

| 위협 | 보호 계층 | 복구 경로 | RTO |
|------|----------|----------|-----|
| 실수로 덮어쓰기 | Source versioning | Source에서 이전 버전 복원 | 즉시 |
| 실수로 삭제 | Source versioning (delete marker) | Delete marker 제거 | 즉시 |
| 랜섬웨어 (대량 덮어쓰기) | Source versioning + IAM deny | Batch Ops로 Source 버전 PITR 복구 | 수 시간 |
| Account A 탈취 | Account B 격리 + Compliance Lock | Batch Ops로 Account B에서 PITR 복구 (복원 + 복사) | 12–48시간 |

### Source 버킷에 Object Lock만 걸면 안 되나?

자연스러운 질문이 생깁니다: **Object Lock(Compliance 모드)이 root도 삭제 못하게 막는다면, Source 버킷에만 걸고 Cross-Account 백업은 생략해도 되지 않을까?**

Source 버킷의 Object Lock은 분명 의미 있는 보호를 제공합니다 — Compliance 모드는 IAM 권한과 무관하게 보존 기간 동안 잠긴 오브젝트 버전의 삭제를 차단합니다. 하지만 **AWS 계정 자체가 탈취**되면 치명적인 한계가 드러납니다:

| 위협 시나리오 | Source Object Lock만 | Cross-Account 백업 |
|--------------|---------------------|-------------------|
| IAM 탈취 → 기존 오브젝트 삭제 시도 | **방어됨** (Compliance 모드가 삭제 차단) | **방어됨** |
| IAM 탈취 → 악성 오브젝트 업로드 | 차단 불가 (새 오브젝트는 수용됨) | 백업에서 PITR로 정상 상태 복구 |
| **AWS 계정 탈취 (root 자격증명)** | **위험**: 계정 폐쇄 시 접근 차단 (90일 유예 기간 후 리소스 삭제) | **방어됨** (별도 계정) |
| root 탈취 → Bucket Policy로 전체 접근 차단 | **위험**: 데이터 접근 불가 | **방어됨** |
| root 탈취 → CloudTrail 비활성화, 로그 조작 | **위험**: 포렌식 증거 소실 | 백업 계정에 독립적 감사 기록 유지 |
| 내부자 위협 (관리자 공모) | 단일 계정 = 단일 신뢰 경계 | 신뢰 경계가 **격리**됨 |

**핵심 한계**: Object Lock은 *버킷 내 오브젝트*를 보호하지만, **계정 수준 작업**으로부터는 보호할 수 없습니다 — 계정 폐쇄, 결제 설정 변경으로 인한 정지, deny-all Bucket Policy 적용 등. 공격자가 계정의 root 접근 권한을 획득하면, 데이터는 기술적으로 온전하더라도 운영적으로 접근 불가능해질 수 있습니다.

Cross-Account 백업은 **별도의 신뢰 경계**를 만들어 이 문제를 해결합니다:

- **계정 격리**: Account A가 침해되어도 Account B의 IAM, SCP, 리소스에 대한 접근 권한은 전혀 없음
- **Object Lock 불변성**: Account B의 Compliance 모드가 예외 없이 모든 삭제를 차단 — root도 잠긴 객체의 삭제나 보존 기간 단축 불가. 한번 활성화된 Object Lock은 버킷에서 제거할 수 없음
- **독립적 메타데이터**: Account B가 자체 S3 Metadata journal을 유지하여, Account A의 메타데이터가 파괴되어도 PITR 가능
- **컴플라이언스 충족**: 금융, 의료 등 많은 규제 프레임워크가 별도 계정 또는 환경에 백업 복사본을 명시적으로 요구

**권장 사항**: Source 버킷의 Object Lock은 일상적인 위협(실수 삭제, IAM 수준 공격)에 대한 **1차 방어선**으로 활용하세요. Cross-Account 백업은 계정 수준 침해 — 다른 모든 방어가 실패한 최악의 시나리오 — 에 대한 **최후의 방어선**으로 활용하세요.

> **본 워크스루에서 Source 버킷에 Object Lock을 설정하지 않는 이유**: 운영 버킷에 Object Lock을 적용하면 보존 기간 동안 잠긴 오브젝트 버전을 삭제할 수 없어 운영상 제약이 생깁니다 — 스토리지 정리, 컴플라이언스 기반 데이터 삭제 등 데이터 생명주기 관리가 복잡해집니다. (참고: Object Lock은 덮어쓰기를 차단하지 않습니다 — 같은 키에 `PutObject`하면 새 버전이 생성되고, 잠긴 기존 버전은 noncurrent로 보존됩니다.) 본 워크스루에서는 Source 버킷에 **IAM deny 정책 + Versioning** 조합을 사용하여, 유사한 보호 효과(비인가 삭제 차단, 모든 버전 보존)를 더 높은 운영 유연성과 함께 제공합니다. 워크로드가 삭제에 대한 보존 제약을 수용할 수 있다면, Source 버킷에도 Object Lock을 추가하는 것을 권장합니다.

## 사전 준비

실습을 시작하기 전에 다음을 준비하세요:

- **두 개의 AWS 계정** (Account A: 운영, Account B: 백업) — 동일 AWS Organization에 속하지 않아도 됨
- **두 계정 모두 동일 리전** (예: `us-east-1`) — 데이터 전송 비용 방지
- **AWS CLI v2.27.51 이상**: 두 계정의 자격 증명으로 구성 (S3 Metadata V2 API에 필요)
- **권한**: 두 계정의 IAM 관리자 권한

```bash
# 두 계정의 CLI 프로파일 설정
aws configure --profile account-a
aws configure --profile account-b

# 본 실습에서 사용하는 변수
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

## Step 1: Source 버킷 구성 (Account A)

### 1.1 Versioning 활성화

Versioning은 전체 아키텍처의 기반입니다. 덮어쓰기나 삭제 시 이전 버전이 보존되어 복구가 가능해집니다.

```bash
aws s3api put-bucket-versioning \
  --bucket ${SOURCE_BUCKET} \
  --versioning-configuration Status=Enabled \
  --profile account-a
```

확인:

```bash
aws s3api get-bucket-versioning \
  --bucket ${SOURCE_BUCKET} \
  --profile account-a

# 예상 출력:
# { "Status": "Enabled" }
```

### 1.2 Noncurrent 버전 Lifecycle 설정

Lifecycle 규칙 없이는 이전 버전이 무한히 누적됩니다. 이 정책으로 복구 기간을 유지하면서 비용을 제어합니다:

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

이 설정의 동작:
- Noncurrent 버전을 **7일간** 즉시 접근 가능하게 유지한 후 Glacier IR로 전환
- 오브젝트당 **최근 3개** noncurrent 버전 유지
- **90일** 이후 noncurrent 버전 삭제

---

## Step 2: Destination 버킷 구성 (Account B)

### 2.1 Object Lock이 활성화된 버킷 생성

Object Lock은 전통적으로 버킷 생성 시에만 활성화할 수 있었습니다. 하지만 현재 AWS는 기존 버킷에서도 Object Lock 활성화를 지원합니다 — S3 콘솔, CLI, 또는 `put-object-lock-configuration` API로 가능합니다. Object Lock은 Versioning을 필요로 합니다 (먼저 활성화하거나 활성화 과정에서 함께 켜짐). 기본 보존 설정은 새 오브젝트 버전에만 적용되며, 기존 버전에는 소급 적용되지 않습니다.

본 실습에서는 새 버킷을 생성합니다:

```bash
# us-east-1의 경우 --create-bucket-configuration 생략
aws s3api create-bucket \
  --bucket ${DEST_BUCKET} \
  --region ${REGION} \
  --create-bucket-configuration LocationConstraint=${REGION} \
  --object-lock-enabled-for-bucket \
  --profile account-b
```

### 2.2 Object Lock을 Compliance 모드로 설정

Compliance 모드는 보존 기간 동안 **root 사용자를 포함하여 그 누구도** 오브젝트를 삭제하거나 수정할 수 없도록 보장합니다. 이것이 계정 탈취 방어의 핵심입니다.

> **왜 180일인가?** Glacier Deep Archive는 **180일 최소 보관 과금**이 적용됩니다 — 180일 이전에 삭제해도 180일치 요금이 부과됩니다. Object Lock 보존 기간을 180일로 설정하여 이 최소 과금 기간에 맞추고, 더 긴 보호 기간을 확보합니다.

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

### 2.3 Glacier Deep Archive를 선택한 이유

Destination 버킷은 **최후의 보루** — Account A가 완전히 탈취된 경우에만 사용합니다. 이런 시나리오에서 12–48시간 복원 대기는 충분히 수용 가능하며, 비용 절감 효과가 큽니다.

| 항목 | Glacier Instant Retrieval | Deep Archive (Bulk, 48시간) | Deep Archive (Standard, 12시간) |
|------|--------------------------|---------------------------|-------------------------------|
| 월간 스토리지 (500TB) | $2,000 | $495 | $495 |
| 전체 복원 요청 비용 (5억 obj) | $5,000 | $12,500 | $50,000 |
| 전체 복원 데이터 검색 비용 (500TB) | $15,000 | $1,250 | $10,000 |
| **전체 복구 총 비용** | **$20,000** | **$13,750** | **$60,000** |
| 복원 시간 | 밀리초 | 48시간 | 12시간 |

**Bulk 검색**(48시간)은 전체 규모 DR에 최적입니다 — 스토리지와 복구 비용 모두 Glacier IR보다 **오히려 저렴**합니다. 더 빠른 복원이 필요한 소규모 개별 복구에는 **Standard 검색**(12시간)을 사용할 수 있으며, 요청당 비용이 더 높습니다.

Deep Archive는 스토리지만으로 **월 $1,505 절약** (연 $18,060)됩니다. Bulk 검색과 결합하면 전체 DR 복구 비용도 Glacier IR보다 **$6,250 저렴**합니다. 유일한 trade-off는 복원 지연 시간(12–48시간)과 2단계 복구 프로세스(복원 → 복사)이며, 이는 Step 6.4에서 다룹니다.

### 2.4 Ops 버킷 생성 (Manifest 및 리포트용)

이후 단계에서 참조되는 Batch Operations 완료 리포트 및 PITR manifest 저장용 버킷을 미리 생성합니다:

```bash
# Account A ops 버킷 (us-east-1의 경우 --create-bucket-configuration 생략)
aws s3api create-bucket \
  --bucket ${OPS_BUCKET_A} \
  --region ${REGION} \
  --create-bucket-configuration LocationConstraint=${REGION} \
  --profile account-a

# Account B ops 버킷
aws s3api create-bucket \
  --bucket ${OPS_BUCKET_B} \
  --region ${REGION} \
  --create-bucket-configuration LocationConstraint=${REGION} \
  --profile account-b
```

---

## Step 3: Cross-Account 동일 리전 복제 설정

동일 리전의 계정 간 복제(SRR)는 **데이터 전송 비용이 $0**입니다 — 이 규모에서 월 ~$10,000이 추가되는 교차 리전 복제(CRR) 대비 핵심 이점입니다.

### 3.1 복제용 IAM Role 생성 (Account A)

```bash
# 신뢰 정책
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

# 권한 정책
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

### 3.2 Destination Bucket Policy 적용 (Account B)

이중 보안을 위해 — Account A의 복제를 허용하면서 모든 삭제 작업을 차단합니다:

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

> **Step 3.1 이후에 적용하는 이유**: Bucket policy가 복제 role의 ARN을 Principal로 참조합니다. AWS는 정책 적용 시 해당 principal의 존재를 검증하므로, role을 먼저 생성해야 합니다.

### 3.3 복제 규칙 설정

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

> **핵심 설정: `DeleteMarkerReplication: Disabled`** — Account A에서 오브젝트를 삭제해도 delete marker가 Account B로 복제되지 않습니다. 공격자가 Source의 모든 것을 삭제하더라도 백업 복사본은 온전하게 유지됩니다.

### 3.4 기존 오브젝트를 S3 Batch Replication으로 복제

위 복제 규칙은 새 오브젝트에만 적용됩니다. 기존 5억 개 오브젝트를 복제하려면 S3 Batch Replication을 사용합니다:

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

Job이 생성되면 실행을 확인합니다:

```bash
# Job이 "Suspended" 상태가 될 때까지 대기 (New → Preparing → Suspended 순서로 전환)
aws s3control describe-job \
  --account-id ${ACCOUNT_A} \
  --job-id <JOB_ID> \
  --query 'Job.Status' \
  --region ${REGION} \
  --profile account-a

# "Suspended" 상태 확인 후 실행 승인
aws s3control update-job-status \
  --account-id ${ACCOUNT_A} \
  --job-id <JOB_ID> \
  --requested-job-status Ready \
  --region ${REGION} \
  --profile account-a
```

---

## Step 4: IAM과 Bucket Policy로 잠금

### 4.1 IAM: 위험한 작업 거부 (Account A)

관리자가 아닌 모든 Role에 인라인 deny 정책(또는 관리형 정책으로 연결)으로 적용합니다. Versioning 비활성화, 버킷 정책 변경, 복제 설정 변경 등을 차단합니다:

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

> **적용 범위**: S3 관리 권한이 필요 없는 모든 IAM role에 적용하세요 — 애플리케이션 role, CI/CD role, 개발자 role 등. 전용 관리자 break-glass role만 예외로 합니다.

### 4.2 Object Lock만으로 Account B가 보호되는 이유

Account B의 Destination 버킷은 Step 2와 3에서 구성한 세 가지 계층으로 이미 보호되며, **어느 것도 AWS Organizations를 필요로 하지 않습니다**:

| 보호 메커니즘 | 차단 대상 | root가 우회 가능? |
|-------------|----------|-----------------|
| **Object Lock Compliance 180일** | 잠긴 오브젝트 버전의 삭제 또는 수정 | **불가** — Compliance 모드는 절대적 |
| **Object Lock (버킷 수준)** | 버킷에서 Object Lock 비활성화 | **불가** — 한번 활성화하면 제거 불가능 |
| **Bucket Policy deny delete** | 모든 `DeleteObject` / `DeleteBucket` 호출 | 가능하지만, Object Lock이 여전히 실제 삭제를 차단 |

핵심 인사이트: **Object Lock Compliance 모드는 S3에서 가장 강력한 보호**입니다 — root, IAM 정책, 버킷 정책 변경으로도 우회할 수 없습니다. 공격자가 Account B의 root 접근 권한을 얻어도 잠긴 오브젝트 버전은 삭제하거나 수정할 수 없습니다. 따라서 Organization 수준 제어(SCP)는 defense-in-depth로서 유용하지만 필수 조건은 아닙니다.

### 4.3 탐지 활성화: GuardDuty + CloudTrail

```bash
# GuardDuty S3 Protection 활성화 (Account A)
# GuardDuty가 이미 활성화된 경우, 기존 detector ID를 가져와서 업데이트합니다:
#   DETECTOR_ID=$(aws guardduty list-detectors --query 'DetectorIds[0]' --output text --profile account-a)
#   aws guardduty update-detector --detector-id ${DETECTOR_ID} \
#     --features '[{"Name":"S3_DATA_EVENTS","Status":"ENABLED"}]' --profile account-a
aws guardduty create-detector \
  --enable \
  --features '[{"Name":"S3_DATA_EVENTS","Status":"ENABLED"}]' \
  --profile account-a

# CloudTrail은 일반적으로 관리 이벤트에 대해 이미 활성화되어 있습니다.
# S3 데이터 이벤트 모니터링 (선택 — API 호출량에 비례하여 비용 발생):
# 본인 소유의 trail 이름을 확인합니다 (Organization 관리 trail 제외):
#   aws cloudtrail describe-trails --query 'trailList[?IsOrganizationTrail==`false`].Name' --profile account-a
# 본인 소유 trail이 없으면 먼저 생성합니다:
#   aws cloudtrail create-trail --name my-s3-trail \
#     --s3-bucket-name my-cloudtrail-logs --profile account-a
#   aws cloudtrail start-logging --name my-s3-trail --profile account-a
TRAIL_NAME="my-s3-trail"  # 실제 trail 이름으로 교체
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

### 4.4 (선택) AWS Organizations의 SCP로 보호 강화

두 계정이 동일 AWS Organization에 속해 있다면 **Service Control Policy(SCP)**를 추가하여 defense-in-depth를 구현할 수 있습니다. SCP는 Organization 수준에서 동작하며 IAM으로 우회할 수 없습니다 — 계정의 root 사용자도 포함됩니다.

> **참고**: SCP는 환경 변수를 지원하지 않습니다. 적용 전에 플레이스홀더(`<SOURCE_BUCKET>`, `<ACCOUNT_A>`, `<DEST_BUCKET>`, `<ACCOUNT_B>`)를 실제 값으로 교체하세요.

**Account A용 SCP** — Versioning 변경 및 대량 삭제 차단:

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

**Account B용 SCP** — 삭제 및 Object Lock 변경 절대 차단:

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

Organization 관리 계정에서 `aws organizations create-policy`와 `attach-policy`로 적용합니다.

> **언제 추가할 가치가 있는가?** SCP는 IAM deny 정책을 root나 IAM 관리자가 우회할 수 있는 Account A에서 가장 유용합니다. Account B는 Object Lock Compliance 모드가 이미 root도 우회 불가능한 보호를 제공하므로 — SCP는 추가 보호 계층이지만 필수는 아닙니다.

---

## Step 5: S3 Metadata와 Athena로 시점 복구(PITR) 구축

S3에는 DynamoDB처럼 네이티브 PITR이 없습니다. **S3 Metadata**(Apache Iceberg 테이블 기반 near real-time 변경 추적), **Athena**(버전 검색 SQL 쿼리), **S3 Batch Operations**(대량 복원)를 조합하여 구축합니다.

S3 Metadata는 두 가지 테이블을 제공합니다:
- **Journal table**: 모든 오브젝트 변경(CREATE, UPDATE_METADATA, DELETE)을 **near real-time**으로 기록
- **Live inventory table**: 전체 오브젝트의 **현재 상태**를 유지하며, 변경 후 ~1시간 내 반영

### 5.1 Ops 버킷

Ops 버킷(`${OPS_BUCKET_A}`, `${OPS_BUCKET_B}`)은 **Step 2.4**에서 이미 생성했습니다. S3 Metadata 테이블은 AWS 관리형 테이블 버킷에 자동 저장되므로 메타데이터용 별도 버킷은 필요 없습니다.

### 5.2 S3 Metadata 활성화 (양쪽 계정)

**Journal table**(변경 추적)과 **Live inventory table**(현재 상태 스냅샷)을 각 버킷에 활성화합니다. 복구 기간에 맞춰 journal 레코드 만료를 90일로 설정합니다.

> **참고:** `create-bucket-metadata-configuration` 명령어(V2 API)는 **AWS CLI v2.27.51 이상**이 필요합니다. `Invalid choice` 에러가 발생하면 `aws --version`으로 버전을 확인하고 [최신 AWS CLI v2로 업그레이드](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)하세요.

**Account A — Source 버킷:**

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

**Account B — Destination 버킷:**

```bash
aws s3api create-bucket-metadata-configuration \
  --bucket ${DEST_BUCKET} \
  --metadata-configuration file:///tmp/metadata-config.json \
  --region ${REGION} \
  --profile account-b
```

확인:

```bash
aws s3api get-bucket-metadata-configuration \
  --bucket ${SOURCE_BUCKET} \
  --region ${REGION} \
  --profile account-a
```

> **왜 양쪽 계정 모두에 S3 Metadata를 설정하는가?** Account A가 탈취되면 공격자가 Account A의 메타데이터 테이블도 삭제할 수 있습니다. Account B의 메타데이터는 최악의 시나리오에서도 PITR을 위한 독립적인 기록을 제공합니다.

> **참고**: Live inventory 테이블을 처음 활성화하면 **backfilling** 프로세스가 진행됩니다 — Amazon S3가 버킷의 모든 기존 오브젝트 메타데이터를 스캔합니다. 버킷 크기에 따라 **15분에서 수 시간** 소요됩니다 (S3 Inventory의 48시간 대기보다 훨씬 빠름). Backfilling 완료 후 업데이트는 ~1시간 내에 반영됩니다. Backfilling에는 **오브젝트 100만 개당 $0.30**의 일회성 비용이 발생합니다 — 5억 개 오브젝트 기준으로 버킷당 ~$150 (양쪽 계정 합산 $300)입니다.

### 5.3 Athena 연동

S3 Metadata 테이블은 AWS 관리형 테이블 버킷(`aws-s3`)에 Apache Iceberg 테이블로 저장됩니다. Athena로 쿼리하려면 테이블 버킷을 AWS 분석 서비스와 통합합니다.

> **중요**: 이 단계를 **Account A와 Account B 모두**에서 수행하세요. Account B의 analytics integration은 Step 6.4의 재해 복구 시나리오에서 Account B의 메타데이터 테이블을 Account B의 Athena에서 직접 쿼리할 때 필요합니다.

1. **S3 콘솔** → **Table buckets** → `aws-s3` 테이블 버킷 선택
2. **Integrations**에서 **Enable integration with AWS analytics services** 선택
3. **Lake Formation**에서 Athena 쿼리를 실행할 IAM 사용자/역할에 메타데이터 테이블 네임스페이스에 대한 `SELECT` 권한 부여

연동 후 Athena에서 다음 형식으로 쿼리합니다:
- **Catalog**: `s3tablescatalog/aws-s3`
- **Database**: `b_<bucket-name>` (버킷 이름의 마침표는 밑줄로 변환)
- **Tables**: `journal` (변경 로그) 또는 `inventory` (현재 상태)

연동 확인 테스트 쿼리:

```sql
-- Athena에서 Catalog를 s3tablescatalog/aws-s3로 설정
-- Database를 b_my-source-bucket으로 설정

SELECT record_type, count(*) as cnt
FROM "s3tablescatalog/aws-s3"."b_my-source-bucket"."journal"
WHERE record_timestamp > current_date - interval '1' day
GROUP BY record_type;
```

### 5.4 PITR 쿼리: 특정 시점의 모든 오브젝트 조회

핵심 쿼리입니다. 대상 타임스탬프를 기준으로 **inventory 테이블**(기준선)과 **journal 항목**(최근 변경분)을 결합하여 **해당 시점의 전체 버킷 상태를 재구성**합니다. 이 하이브리드 접근법이 필요한 이유는 journal이 S3 Metadata 활성화 이후의 변경만 기록하기 때문입니다 — 한 번도 수정되지 않은 기존 오브젝트는 journal-only 쿼리에서 누락됩니다.

```sql
-- 대상: 2024-07-15 14:00:00 UTC로 복구
-- 하이브리드 접근법: inventory (기준선) + journal (최근 변경분)

-- Step 1: inventory와 journal 데이터의 경계 시점 결정
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

-- Step 2: inventory (기준선) + journal (델타)로 작업 세트 구성
working_set AS (
    -- 기준선: inventory 스냅샷의 모든 오브젝트
    SELECT bucket, key, sequence_number, version_id, is_delete_marker,
           last_modified_date, size, storage_class,
           CAST(NULL AS varchar) AS record_type
    FROM "s3tablescatalog/aws-s3"."b_my-source-bucket"."inventory" i
    WHERE i.last_modified_date <= TIMESTAMP '2024-07-15 14:00:00'

    UNION ALL

    -- 델타: inventory 경계 이후의 journal 항목 (15분 오버랩 버퍼 포함)
    SELECT bucket, key, sequence_number, version_id, is_delete_marker,
           COALESCE(last_modified_date, record_timestamp) AS last_modified_date,
           size, storage_class, record_type
    FROM "s3tablescatalog/aws-s3"."b_my-source-bucket"."journal" j
    CROSS JOIN inventory_time_cte t
    WHERE j.last_modified_date > (t.inventory_time - interval '15' minute)
      AND j.last_modified_date <= TIMESTAMP '2024-07-15 14:00:00'
),

-- Step 3: 각 (key, version_id)별 version stack의 tip(최신 이벤트) 찾기
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

-- Step 4: 영구 삭제된 버전 제외 (delete marker와 live 버전은 유지)
existing_versions AS (
    SELECT * FROM version_tips
    WHERE record_type IS NULL        -- inventory 행: 항상 유지
       OR record_type != 'DELETE'    -- journal 비삭제 이벤트: 유지
       OR is_delete_marker = TRUE    -- journal delete marker (소프트 삭제): 유지
    -- 필터 대상: journal 영구 삭제 (record_type='DELETE', is_delete_marker=FALSE)
),

-- Step 5: 각 key별 최신 현존 버전 식별
with_is_latest AS (
    SELECT *,
      sequence_number = MAX(sequence_number) OVER (
        PARTITION BY bucket, key
      ) AS is_latest_version
    FROM existing_versions
)

-- Step 6: key별 최신 버전 선택, delete marker 제외
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

**쿼리 해석:**
- **Step 1** — `journal$properties`의 `oldest-uncoalesced-record-timestamp`가 inventory 스냅샷 종료 시점과 journal 레코드 시작 시점의 경계를 제공합니다. 쿼리에 하드코딩된 `2024-12-01 00:00`은 fallback 기본값일 뿐입니다 — `COALESCE()`를 통해 `journal$properties`에서 실제 경계 시점을 자동으로 읽어오므로, 대부분의 경우 이 값을 변경할 필요가 없습니다. 만약 `journal$properties`에 아직 타임스탬프가 없는 경우(예: inventory backfill이 진행 중), fallback 값을 해당 버킷에 S3 Metadata를 활성화한 대략적인 시점으로 변경하세요.
- **Step 2** — Inventory가 **모든 오브젝트**(S3 Metadata 이전 오브젝트 포함)의 기준선을 제공합니다. Journal은 inventory 경계 이후의 최근 변경분만 추가하며, 15분 오버랩 버퍼로 gap을 방지합니다.
- **Step 3** — `PARTITION BY bucket, key, coalesce(version_id, '')`로 각 오브젝트 버전별 "version stack"을 구축합니다([AWS 문서 패턴](https://docs.aws.amazon.com/AmazonS3/latest/userguide/metadata-tables-example-queries.html)). `LEAD()`로 다음 이벤트를 찾고, NULL이면 해당 버전의 최신 이벤트(tip)입니다.
- **Step 4** — 영구 삭제된 버전(journal의 `record_type='DELETE'` + `is_delete_marker=FALSE`)을 제거합니다. Inventory 행(`record_type IS NULL`)은 항상 유지됩니다. 이렇게 하면 Lifecycle noncurrent version expiration이 현재 버전을 가리는 문제를 방지합니다.
- **Step 5–6** — 살아남은 버전 중 각 key별 최신 버전을 선택하고, delete marker(해당 시점에 삭제된 오브젝트)를 제외합니다.

> **왜 inventory + journal인가?** Journal은 S3 Metadata **활성화 이후**의 변경만 기록합니다. 한 번도 수정되지 않은 기존 오브젝트에는 journal 항목이 없습니다. Inventory 테이블은 백필을 통해 모든 오브젝트를 포함하므로, 완전한 PITR에 필수적입니다. 또한 journal 레코드는 만료됩니다(이 아키텍처에서 90일). 오래된 CREATE 이벤트는 결국 사라지므로, journal만으로는 제공할 수 없는 안정적인 기준선을 inventory 테이블이 제공합니다.

> **성능 이점**: 5억 개 오브젝트 버킷에서 이 접근법은 대부분의 상태를 사전 집계된 inventory에서 읽고 최근 journal 델타만 스캔하므로, 전체 journal 스캔 대비 Athena 비용을 **90% 이상 절감**합니다.

### 5.5 변형: 삭제된 오브젝트도 포함하여 복구

랜섬웨어가 덮어쓰기 전에 오브젝트를 삭제한 경우, 삭제된 오브젝트도 복구할 수 있습니다. 이 쿼리는 5.4를 확장하여, 최신 버전이 delete marker인 오브젝트도 가장 최근의 비삭제 버전을 찾아 복구합니다:

```sql
-- 5.4와 동일한 하이브리드 working_set (inventory 기준선 + journal 델타)
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
-- 대상 시점에 살아있는 오브젝트
alive AS (
    SELECT bucket, key, version_id, size
    FROM with_is_latest
    WHERE is_latest_version = TRUE
      AND COALESCE(is_delete_marker, FALSE) = FALSE
),
-- 대상 시점에 삭제된 오브젝트 — 가장 최근의 비삭제 버전으로 복구
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

### 5.6 현재 상태 쿼리 (Live Inventory 테이블)

현재 버킷 상태를 빠르게 확인할 때 — 예를 들어 현재 오브젝트 수를 확인할 때 — live inventory 테이블을 사용합니다:

```sql
SELECT storage_class, count(*) as object_count, sum(size) as total_bytes
FROM "s3tablescatalog/aws-s3"."b_my-source-bucket"."inventory"
WHERE NOT is_delete_marker
GROUP BY storage_class;
```

Live inventory 테이블은 ~1시간 내에 갱신되며, Step 5.2에서 활성화한 것 외에 별도 설정이 필요 없습니다.

---

## Step 6: 시점 복구 실행

인시던트 발생 시, 이 런북을 따라 특정 시점으로 데이터를 복원합니다.

> **시작 전 확인**: 아래 예시는 PITR 대상 시점으로 `2024-07-15 14:00:00`을, S3 경로에 `2024-07-15T14`(예: `pitr-manifests/2024-07-15T14/`)을 사용합니다. 다른 시점으로 복원할 경우, SQL 쿼리의 `TIMESTAMP` 값 **과** bash 명령어의 S3 경로를 **모두** 대상 날짜에 맞게 변경하세요.

### 6.1 PITR Manifest 생성

Athena에서 PITR 쿼리를 실행하고, 자동 생성되는 CSV 결과 파일을 manifest로 사용합니다. Athena는 모든 SELECT 결과를 워크그룹의 쿼리 결과 위치에 CSV 파일로 저장합니다.

> **참고**: S3 Metadata 테이블은 `s3tablescatalog`에 있으며, 이 카탈로그는 Iceberg 포맷(AVRO, ORC, PARQUET)만 지원합니다 — `TEXTFILE` 포맷의 CTAS는 지원되지 않습니다. 대신 일반 `SELECT`를 실행하고 결과 CSV를 후처리합니다.

```sql
-- Athena에서 실행: Catalog: s3tablescatalog/aws-s3, Database: b_my-source-bucket
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

쿼리 완료 후, Athena 쿼리 결과 위치에서 결과 CSV를 다운로드하고 헤더 행을 제거한 뒤 manifest로 업로드합니다:

```bash
# 결과 S3 경로 복사: Athena 콘솔 > Recent queries > 쿼리 선택 > Output location
# 경로는 워크그룹 설정에 따라 다릅니다 (예: Unsaved/2024/07/15/<query-id>.csv 또는 query-results/<query-id>.csv)
QUERY_RESULT="s3://${OPS_BUCKET_A}/Unsaved/2024/07/15/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.csv"  # ← 실제 Output location으로 변경

aws s3 cp "${QUERY_RESULT}" /tmp/pitr-result.csv --profile account-a
tail -n +2 /tmp/pitr-result.csv > /tmp/pitr-manifest.csv    # 헤더 행 제거
aws s3 cp /tmp/pitr-manifest.csv s3://${OPS_BUCKET_A}/pitr-manifests/2024-07-15T14/manifest.csv --profile account-a
```

> **S3 키의 콤마**: Athena의 CSV 출력은 필드 값을 따옴표로 감싸므로, S3 키 내 콤마는 정상 처리됩니다. 다만 키에 콤마와 큰따옴표가 모두 포함된 경우, 진행 전에 manifest 파일을 확인하세요.

### 6.2 Batch Operations 복원 Job 생성

먼저 복원용 버킷을 생성한 후, Batch Operations job으로 특정 버전을 복사합니다:

```bash
# 복원 버킷 생성 (us-east-1의 경우 --create-bucket-configuration 생략)
aws s3api create-bucket \
  --bucket ${RESTORE_BUCKET_A} \
  --region ${REGION} \
  --create-bucket-configuration LocationConstraint=${REGION} \
  --profile account-a

# Batch Operations IAM role 생성
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

Batch Operations job을 생성합니다:

```bash
# Manifest 파일의 ETag 조회
MANIFEST_ETAG=$(aws s3api head-object \
  --bucket ${OPS_BUCKET_A} \
  --key pitr-manifests/2024-07-15T14/manifest.csv \
  --query ETag --output text \
  --profile account-a | tr -d '"')

# Batch Operations job 생성
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

### 6.3 복원 Job 확인 및 모니터링

`--confirmation-required`를 사용했으므로 job은 **Suspended** 상태로 생성됩니다. 실행을 시작하려면 확인이 필요합니다:

```bash
# create-job 출력에서 job ID를 받기
JOB_ID="your-job-id-here"

# "Suspended" 상태 대기 후 실행 승인
aws s3control describe-job \
  --account-id ${ACCOUNT_A} \
  --job-id ${JOB_ID} \
  --query 'Job.Status' \
  --region ${REGION} \
  --profile account-a

# Job 실행 확인
aws s3control update-job-status \
  --account-id ${ACCOUNT_A} \
  --job-id ${JOB_ID} \
  --requested-job-status Ready \
  --region ${REGION} \
  --profile account-a

# 진행 상황 모니터링:
aws s3control describe-job \
  --account-id ${ACCOUNT_A} \
  --job-id ${JOB_ID} \
  --query 'Job.{Status:Status,Succeeded:ProgressSummary.NumberOfTasksSucceeded,Failed:ProgressSummary.NumberOfTasksFailed,Total:ProgressSummary.TotalNumberOfTasks}' \
  --region ${REGION} \
  --profile account-a
```

### 6.4 재해 복구: Account B에서 복원

Account A가 탈취된 경우, Account B의 메타데이터를 사용하여 동일한 프로세스를 실행합니다:

**사전 준비**: Account B에 복원 버킷과 Batch Operations IAM role을 생성합니다 (Step 6.2의 Account A role과 동일 구조, Account B 버킷 참조):

```bash
# Account B에 복원 버킷 생성 (us-east-1의 경우 --create-bucket-configuration 생략)
aws s3api create-bucket \
  --bucket ${RESTORE_BUCKET_B} \
  --region ${REGION} \
  --create-bucket-configuration LocationConstraint=${REGION} \
  --profile account-b

# Account B에 Batch Ops role 생성 (Account A와 동일한 trust policy)
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

Account B의 journal에서 PITR manifest를 생성합니다 (Step 6.1과 동일한 SELECT 방식):

```sql
-- Account B의 Athena에서 실행
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

Destination이 **Glacier Deep Archive**를 사용하므로 복사 전에 먼저 오브젝트를 복원해야 합니다. 2단계 Batch Operations 프로세스가 필요합니다.

먼저 Athena 쿼리 결과를 다운로드하고 헤더를 제거한 뒤 manifest로 업로드합니다 (Step 6.1과 동일한 프로세스):

```bash
# 결과 S3 경로 복사: Athena 콘솔 > Recent queries > 쿼리 선택 > Output location
QUERY_RESULT_B="s3://${OPS_BUCKET_B}/Unsaved/2024/07/15/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.csv"  # ← 실제 Output location으로 변경
aws s3 cp "${QUERY_RESULT_B}" /tmp/pitr-result-dr.csv --profile account-b
tail -n +2 /tmp/pitr-result-dr.csv > /tmp/pitr-manifest-dr.csv
aws s3 cp /tmp/pitr-manifest-dr.csv s3://${OPS_BUCKET_B}/pitr-manifests/dr-2024-07-15/manifest.csv --profile account-b

# Manifest ETag 조회
MANIFEST_ETAG=$(aws s3api head-object \
  --bucket ${OPS_BUCKET_B} \
  --key pitr-manifests/dr-2024-07-15/manifest.csv \
  --query ETag --output text \
  --profile account-b | tr -d '"')
```

**Step 1: 아카이브 복원 시작 (Bulk = 48시간, Standard = 12시간)**

```bash
# Batch Operations 아카이브 복원 job 생성
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

Job을 확인하여 실행을 시작하고, 모든 복원이 완료될 때까지 모니터링합니다 (Step 6.3과 동일):

```bash
aws s3control update-job-status \
  --account-id ${ACCOUNT_B} \
  --job-id <RESTORE_JOB_ID> \
  --requested-job-status Ready \
  --region ${REGION} \
  --profile account-b
```

> **검색 티어 선택**: 전체 규모 DR에는 `BULK`(48시간)을 사용하세요 — 가장 저렴하며 Glacier IR보다도 비용이 낮습니다. 빠른 접근이 필요한 소규모 개별 복원에는 `STANDARD`(12시간)를 사용할 수 있으며, 요청당 비용이 더 높습니다.

> **중요 — 2단계 복원 프로세스**: `S3InitiateRestoreObject` Batch Operations 작업은 모든 복원 **요청 제출**이 완료되면 Complete로 표시됩니다 — 이것은 오브젝트에 접근할 수 있다는 의미가 **아닙니다**. 실제 Glacier Deep Archive 복원은 작업 완료 후 **12시간(Standard)** 또는 **48시간(Bulk)**이 소요됩니다. 복원이 완료될 때까지 기다린 후 Step 2를 진행해야 합니다.

**Glacier 복원 진행 상태 모니터링:**

S3는 각 오브젝트의 복원이 완료될 때마다 EventBridge를 통해 `s3:ObjectRestore:Completed` 이벤트를 전송합니다. EventBridge 규칙을 설정하여 완료 수를 카운트하고 모든 오브젝트가 준비되면 알림을 받을 수 있습니다.

**옵션 A: EventBridge + CloudWatch 카운터 (전체 오브젝트 확인에 권장)**

`ObjectRestore:Completed` 이벤트를 CloudWatch 메트릭으로 카운트하는 EventBridge 규칙을 생성합니다. 메트릭 카운트가 manifest의 총 오브젝트 수와 일치하면 모든 복원이 완료된 것입니다.

```bash
# 1. 백업 버킷에 EventBridge 알림 활성화
aws s3api put-bucket-notification-configuration \
  --bucket ${DEST_BUCKET} \
  --notification-configuration '{"EventBridgeConfiguration": {}}' \
  --profile account-b

# 2. 복원 완료를 카운트하는 EventBridge 규칙 생성
aws events put-rule \
  --name "s3-restore-completed" \
  --event-pattern '{
    "source": ["aws.s3"],
    "detail-type": ["Object Restore Completed"],
    "detail": {"bucket": {"name": ["'${DEST_BUCKET}'"]}}
  }' \
  --region ${REGION} \
  --profile account-b

# 3. (선택) SNS 타겟을 추가하여 오브젝트별 이메일 알림 수신
# aws events put-targets --rule "s3-restore-completed" --targets '[{"Id":"sns","Arn":"arn:aws:sns:'${REGION}':'${ACCOUNT_B}':restore-notify"}]'
```

**CloudWatch** → **Metrics** → **Events** → **TriggeredRules**에서 `s3-restore-completed` 규칙의 카운트를 모니터링하고, manifest의 총 라인 수와 비교합니다:

```bash
# 복원 대상 총 오브젝트 수
wc -l < /tmp/pitr-manifest-dr.csv
```

**옵션 B: manifest의 여러 지점에서 샘플 확인 (간편 방법)**

EventBridge 설정 없이 빠르게 확인하려면, manifest의 처음, 중간, 마지막 오브젝트를 확인합니다:

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

- `ongoing-request="true"` → 복원 진행 중, **대기 후 재확인**
- `ongoing-request="false", expiry-date="..."` → 복원 완료

모든 샘플이 `ongoing-request="false"`를 표시하면 Step 2를 진행합니다. 대규모 복원(수백만 오브젝트)의 경우, 옵션 A가 확실한 확인 방법입니다.

**Step 2: 복원된 오브젝트를 새 버킷에 복사 (복원 완료 후)**

```bash
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

이 Job도 동일하게 확인하고 모니터링합니다:

```bash
aws s3control update-job-status \
  --account-id ${ACCOUNT_B} \
  --job-id <COPY_JOB_ID> \
  --requested-job-status Ready \
  --region ${REGION} \
  --profile account-b
```

> **DR 완료 확인**: **S3 콘솔** → **Batch Operations** → 복사 작업을 선택합니다. 상태가 **Complete**이고 **Successful** 수가 **Total** 수와 일치하면 DR 복구가 완료된 것입니다. `${RESTORE_BUCKET_B}` 버킷에서 복원된 오브젝트를 확인하세요.

---

## 복구 비용 추정

**Source에서 복구 (Account A)** — noncurrent 버전은 Glacier IR, 즉시 접근 가능:

| 복구 규모 | Batch Ops | GET 요청 (Glacier IR) | 데이터 검색 (Glacier IR) | Athena | 합계 |
|-----------|-----------|---------------------|------------------------|--------|------|
| 100개 오브젝트 (실수) | ~$0 | ~$0 | ~$0 | $0.03 | **~$0** |
| 100만 오브젝트 (부분) | $1.25 | $10 | ~$300 | $0.03 | **~$311** |
| 5억 오브젝트 (전체 DR) | $500 | $5,000 | $15,000 | $0.03 | **~$20,500** |

**Destination에서 복구 (Account B)** — Deep Archive, 복원 필요:

| 복구 규모 | Batch Ops (2 jobs) | 복원 + 검색 (Bulk, 48시간) | 복원 + 검색 (Standard, 12시간) | Athena | 합계 (Bulk) |
|-----------|-------------------|--------------------------|-------------------------------|--------|------------|
| 100개 오브젝트 | ~$1 | ~$0 | ~$0 | $0.03 | **~$1** |
| 100만 오브젝트 (10TB) | $2.50 | ~$50 | ~$300 | $0.03 | **~$53** |
| 5억 오브젝트 (전체 DR) | $1,001 | $13,750 | $60,000 | $0.03 | **~$14,751** |

---

## PITR 정밀도와 한계

| 항목 | 상세 |
|------|------|
| **복구 정밀도** | 모든 타임스탬프 — journal이 near real-time으로 변경 기록 |
| **초기 설정 시간** | Live inventory backfill: 15분~수 시간 (S3 Inventory의 48시간 대비) |
| **Journal 레코드 보존** | 설정 가능 — 본 아키텍처에서 90일 (최소 7일) |
| **RPO (복구 시점 목표)** | **Near real-time** — journal이 모든 오브젝트 변경을 발생 즉시 캡처 |

> **참고**: 오브젝트 변경이 많은 장기간 PITR 쿼리는 journal 테이블 스캔 비용이 높을 수 있습니다. 항상 `record_timestamp` 범위 필터를 포함하여 스캔 범위를 제한하세요. 현재 상태 쿼리는 전체 journal 스캔 대신 live inventory 테이블을 사용하세요.

---

## 구현 우선순위

| 단계 | 작업 | 일정 | 비용 영향 |
|------|------|------|----------|
| **즉시** | Versioning 활성화, IAM deny 정책으로 Versioning 보호 | Day 1 | $0 |
| **1주차** | Account B 생성, Object Lock이 적용된 Destination 버킷 구성 | 1주차 | 복제 시작 전까지 $0 |
| **2주차** | Cross-account SRR 설정, 양쪽 계정 S3 Metadata 활성화 | 2주차 | ~$600/월 시작 |
| **3주차** | 기존 오브젝트 Batch Replication 실행, Athena 테이블 생성 | 3주차 | 일회성 복제 요청 비용 |
| **4주차** | PITR 복원 테스트 (부분), 런북 문서화, GuardDuty 활성화 | 4주차 | 테스트 복원 ~$15 |

---

## 최종 월간 비용 요약

| 항목 | 월 비용 |
|------|--------|
| Destination 스토리지 (Deep Archive, 500TB) | $495 |
| S3 Metadata journal — Account A (~0.1–1% 일별 변경률) | $5–45 |
| S3 Metadata journal — Account B (~0.1–1% 일별 변경률) | $5–45 |
| S3 Metadata live inventory (10억 미만 무료) | $0 |
| 데이터 전송 (동일 리전 SRR) | $0 |
| Object Lock / Versioning / IAM deny 정책 | $0 |
| CloudTrail (관리 이벤트) + GuardDuty | ~$10 |
| **총 추가 비용** | **~$515–595/월** |

---

## 결론

S3 Versioning, Cross-Account 동일 리전 복제, Object Lock(Compliance 모드), IAM deny 정책, S3 Metadata, Athena, S3 Batch Operations 등 AWS 네이티브 서비스를 조합하여 다음을 달성하는 데이터 보호 아키텍처를 구축했습니다:

1. **랜섬웨어 방어**: 공격자가 데이터를 덮어써도 Object Versioning이 원본을 보존; IAM deny 정책이 버전 삭제 차단
2. **계정 탈취 생존**: Cross-account 격리와 Compliance 모드 Object Lock으로 root도 백업 복사본 삭제 불가
3. **실수 방지**: Versioning + IAM deny 정책으로 다중 안전망 구성 (AWS Organizations가 있다면 SCP로 추가 강화 가능)
4. **시점 복구 지원**: S3 Metadata journal 테이블이 near real-time으로 모든 변경을 추적하고, Athena로 SQL 검색하며, S3 Batch Operations로 대규모 복원

AWS Backup for S3가 가장 기능이 풍부하고 운영이 간편한 옵션이지만, 이 아키텍처는 대규모 환경에서 AWS Backup 비용이 부담될 때 최소한의 비용으로 필수적인 데이터 보호를 제공합니다. Destination은 Glacier Deep Archive를 사용하여 스토리지 비용을 극대화하며, 전체 규모 DR에는 Bulk 검색(48시간), 개별 복원에는 Standard 검색(12시간)을 활용합니다.

핵심 인사이트: **보호 계층(복제 + 잠금)과 복구 계층(메타데이터 + 쿼리 + 배치 복원)을 분리**하는 것입니다. 보호 계층은 저비용으로 지속 운영되고, 복구 계층은 대부분 휴면 상태입니다 — S3 Metadata journal 비용은 실제 변경량에 비례하며, 비용이 큰 작업(Athena 쿼리, Batch Operations)은 실제 복구가 필요할 때만 실행됩니다.

---

*저자 소개: 이 아키텍처는 관리형 백업 서비스 비용이 부담되는 대규모 S3 환경에서 기본적인 데이터 보호가 필요한 조직을 위해 설계되었습니다.*
