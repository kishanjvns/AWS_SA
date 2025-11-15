# AWS S3 & KMS Security - Learning Summary

> **⚠️ SECURITY NOTICE:** This document has been sanitized for public sharing. All sensitive information including AWS account IDs, KMS key IDs, IAM user names, and bucket names have been replaced with placeholders or example values.

## Overview

Explored AWS S3 permissions, bucket policies, and KMS encryption with hands-on testing using IAM users and CLI commands.

---

## Topics Covered

### 1. AWS S3 Basic Operations

**Commands Learned:**

```bash
# List all S3 buckets
aws s3 ls

# Upload object to S3
aws s3 cp <local-file> s3://<bucket-name>/<key>
aws s3api put-object --bucket <bucket> --key <key> --body <file>

# Download object from S3
aws s3 cp s3://<bucket-name>/<key> <local-file>
aws s3api get-object --bucket <bucket> --key <key> <output-file>

# Delete object from S3
aws s3 rm s3://<bucket-name>/<key>
aws s3api delete-object --bucket <bucket> --key <key>
```

---

### 2. S3 Bucket Policies - Granular Access Control

**Bucket:** `my-test-bucket`

**Scenario Tested:**
- Public read access to `public_space/*` folder only (HTTPS enforced)
- User-specific permissions: `iam-user-1` (PutObject) vs `iam-user-2` (DeleteObject)

**Final Bucket Policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadPublicSpace",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-test-bucket/public_space/*",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "true"
        }
      }
    },
    {
      "Sid": "IAMUser1PutGet",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/iam-user-1"
      },
      "Action": [
        "s3:PutObject",
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::my-test-bucket/*"
    },
    {
      "Sid": "IAMUser2DeleteGet",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/iam-user-2"
      },
      "Action": [
        "s3:DeleteObject",
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::my-test-bucket/*"
    }
  ]
}
```

**Key Learnings:**
- ✅ Resource: `".../*"` → Applies to objects inside bucket
- ✅ Resource: `"..."` → Applies to bucket itself (for ListBucket)
- ✅ Conditions enforce security (HTTPS-only access)
- ✅ Bucket policies work for resource-specific permissions

---

### 3. IAM Policies vs Bucket Policies

**Problem Encountered:** Bucket policy restrictions weren't working because `iam-user-2` had `AdministratorAccess` IAM policy attached.

**Root Cause:**
- AWS evaluates both IAM policies and bucket policies
- Uses union of Allow statements (if either allows, action succeeds)
- Explicit Deny always wins over any Allow

**Solution:** Removed administrative permissions from IAM users to test bucket policy isolation.

**Decision Framework:**

| Use Case | Best Choice |
|----------|-------------|
| Cross-account access | Bucket Policy (only option) |
| Public/anonymous access | Bucket Policy (only option) |
| Specific resource control | Bucket Policy |
| Few users, few resources | Bucket Policy |
| Many users across resources | IAM Policy with Groups |
| User-centric permissions | IAM Policy |
| Multiple services access | IAM Policy |

**Key Learning:**
- **IAM Policy** = Identity-based (what can THIS user do?)
- **Bucket Policy** = Resource-based (who can access THIS bucket?)
- Both must allow for action to succeed

---

### 4. AWS KMS Customer Managed Keys (CMK)

**Objective:** Implement server-side encryption using Customer Managed KMS key instead of AWS managed key.

**KMS CMK Setup:**
- **Key ARN:** `arn:aws:kms:us-east-1:123456789012:key/abcd1234-5678-90ef-ghij-klmnopqrstuv`

**Enabled Default Encryption:**
- **Algorithm:** SSE-KMS
- **Bucket Key:** Enabled (cost optimization)

**Command Used:**

```bash
aws s3api put-bucket-encryption \
    --bucket my-test-bucket \
    --server-side-encryption-configuration '{
        "Rules": [{
            "ApplyServerSideEncryptionByDefault": {
                "SSEAlgorithm": "aws:kms",
                "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/abcd1234-5678-90ef-ghij-klmnopqrstuv"
            },
            "BucketKeyEnabled": true
        }]
    }'
```

---

### 5. The Access Denied Problem - KMS Permissions

**Problem:**

```bash
aws s3 cp dp.png s3://my-test-bucket/dp.png
# Error: AccessDenied - not authorized to perform: kms:GenerateDataKey
```

**Root Cause Analysis:**

Even though `iam-user-1` had S3 PutObject permission:
- ❌ IAM user lacked KMS permissions
- Encryption requires calling KMS to generate data keys
- S3 service needs permission to use KMS on behalf of the user

**The Critical Insight:**

> **S3 permissions ≠ KMS permissions**

For encrypted uploads/downloads to work, you need:
- ✅ S3 bucket policy allowing PutObject/GetObject
- ✅ KMS key policy allowing GenerateDataKey/Decrypt
- Missing either = Access Denied!

---

### 6. KMS Key Policy - The Solution

**Final KMS Key Policy:**

```json
{
  "Version": "2012-10-17",
  "Id": "key-consolepolicy",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "AllowApplicationRoleDecrypt",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/my-application-role"
      },
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    },
    {
      "Sid": "BucketEncryptionPermissions",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/iam-user-1"
      },
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey",
        "kms:GenerateDataKey"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowS3ToUseTheKey",
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "*"
    }
  ]
}
```

**KMS Permissions Breakdown:**

| Permission | Used For |
|------------|----------|
| `kms:GenerateDataKey` | Uploading encrypted objects (PutObject) |
| `kms:Decrypt` | Downloading encrypted objects (GetObject) |
| `kms:DescribeKey` | Viewing key metadata |

**After adding KMS permissions:**

```bash
aws s3 cp dp.png s3://my-test-bucket/dp.png
# ✅ Success!
```

---

### 7. S3 Bucket Key - Cost Optimization

**What is Bucket Key?**

Reduces KMS API calls by generating object keys locally from a bucket-level key.

- **Without Bucket Key:** 1 million uploads = 1 million KMS API calls
- **With Bucket Key:** 1 million uploads = ~1 KMS API call

**Cost Impact:**

| Objects/Month | Without Bucket Key | With Bucket Key | Savings |
|---------------|-------------------|-----------------|---------|
| 10,000 | $0.03 | ~$0.00 | ~$0.03 |
| 1,000,000 | $3.00 | ~$0.01 | ~$2.99 |
| 100,000,000 | $300.00 | ~$0.50 | ~$299.50 |

**Recommendation:** Always enable unless compliance requires individual KMS logs per object.

**Configuration:**
- ✅ Enabled in bucket encryption settings
- No code changes required
- Transparent to applications

---

### 8. Understanding S3 Replication Warning

**Warning Seen:**

> "Changing default encryption might cause replication jobs to fail due to missing KMS permissions on IAM replication role"

**Explanation:**
- Applies only if using S3 Cross-Region or Same-Region Replication
- Replication role needs KMS permissions to decrypt source and encrypt destination objects
- Not relevant if not using replication

**If Using Replication, Add:**

```json
{
  "Sid": "AllowReplicationRole",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:role/my-replication-role"
  },
  "Action": [
    "kms:Decrypt",
    "kms:DescribeKey"
  ],
  "Resource": "*"
}
```

---

## Key Concepts Mastered

### 1. Defense in Depth

AWS uses multiple permission layers - all must allow for access:
- IAM policies (identity-based)
- Bucket policies (resource-based)
- KMS key policies (for encryption)
- Block Public Access settings

### 2. Resource ARN Patterns

```
arn:aws:s3:::bucket-name          → Bucket itself
arn:aws:s3:::bucket-name/*        → Objects in bucket
arn:aws:s3:::bucket-name/path/*   → Objects in specific path
```

### 3. AWS CLI Profile Management

```bash
# Configure named profile
aws configure --profile my-profile

# Use profile in commands
aws s3 ls --profile my-profile
```

### 4. Permission Troubleshooting

When getting Access Denied:
1. Check S3 bucket policy
2. Check IAM user/role policies
3. Check KMS key policy (if encrypted)
4. Check Block Public Access settings
5. Verify `aws:SecureTransport` conditions

---

## Problems Solved

### Problem 1: Public vs Private Access

- **Challenge:** Allow public read to specific folder, keep rest private
- **Solution:** Bucket policy with `Resource: "arn:aws:s3:::bucket/public_space/*"`

### Problem 2: User-Specific Permissions

- **Challenge:** Different users need different S3 operations
- **Solution:** Separate bucket policy statements per user with specific actions

### Problem 3: IAM Policy Override

- **Challenge:** Bucket policy not working due to admin permissions
- **Solution:** Remove conflicting IAM policies to test bucket policy isolation

### Problem 4: KMS Encryption Access Denied

- **Challenge:** Can't upload to encrypted bucket despite S3 permissions
- **Solution:** Add KMS `GenerateDataKey` and `Decrypt` permissions to key policy

### Problem 5: Understanding Permission Layers

- **Challenge:** Confusion about when to use IAM vs bucket vs KMS policies
- **Solution:** Learned resource-based vs identity-based policy patterns

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                AWS Account: 123456789012                     │
│                                                              │
│  ┌────────────────┐           ┌──────────────────────────┐  │
│  │  IAM Users     │           │   S3: my-test-bucket     │  │
│  │                │           │                          │  │
│  │ iam-user-1 ────┼──────────►│  /public_space/*         │  │
│  │   (PutObject,  │           │    (Public Read)         │  │
│  │    GetObject)  │           │                          │  │
│  │                │           │  /other files            │  │
│  │ iam-user-2 ────┼──────────►│    (Private)             │  │
│  │   (DeleteObj,  │           │                          │  │
│  │    GetObject)  │           │  Encryption: SSE-KMS     │  │
│  └────────┬───────┘           │  Bucket Key: Enabled     │  │
│           │                   └────────┬─────────────────┘  │
│           │                            │                    │
│           │                            │                    │
│           │    ┌───────────────────────▼─────────────────┐  │
│           │    │  KMS Customer Managed Key (CMK)        │  │
│           │    │  Key ID: abcd1234-5678-90ef-ghij-...   │  │
│           │    │                                         │  │
│           └───►│  Permissions:                           │  │
│                │  - iam-user-1: Encrypt/Decrypt          │  │
│                │  - S3 Service: Encrypt/Decrypt          │  │
│                │  - app-role: Decrypt                    │  │
│                └─────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Testing Flow

```
1. Configure AWS CLI
   aws configure --profile my-profile
   ↓
2. Test S3 Upload (without KMS permissions)
   aws s3 cp file.txt s3://my-test-bucket/
   ↓
   ❌ Access Denied - kms:GenerateDataKey
   ↓
3. Add KMS Permissions to Key Policy
   {
     "Action": ["kms:GenerateDataKey", "kms:Decrypt"],
     "Principal": {"AWS": "arn:aws:iam::123456789012:user/iam-user-1"}
   }
   ↓
4. Retry S3 Upload
   aws s3 cp file.txt s3://my-test-bucket/
   ↓
   ✅ Success!
   ↓
5. Verify Encryption
   aws s3api head-object --bucket my-test-bucket --key file.txt
   ↓
   ServerSideEncryption: aws:kms ✅
   BucketKeyEnabled: true ✅
```

---

## Commands Reference Sheet

### S3 Operations

```bash
aws s3 ls
aws s3 ls s3://<bucket-name>/
aws s3 cp <local-file> s3://<bucket>/<key>
aws s3 cp s3://<bucket>/<key> <local-file>
aws s3 rm s3://<bucket>/<key>
```

### S3 API Operations

```bash
aws s3api put-object --bucket <NAME> --key <KEY> --body <FILE>
aws s3api get-object --bucket <NAME> --key <KEY> <OUTPUT>
aws s3api delete-object --bucket <NAME> --key <KEY>
aws s3api head-object --bucket <NAME> --key <KEY>
```

### Bucket Configuration

```bash
aws s3api get-bucket-policy --bucket <NAME>
aws s3api put-bucket-policy --bucket <NAME> --policy file://policy.json
aws s3api get-bucket-encryption --bucket <NAME>
aws s3api put-bucket-encryption --bucket <NAME> --server-side-encryption-configuration '{...}'
```

### Block Public Access

```bash
aws s3api put-public-access-block \
    --bucket <NAME> \
    --public-access-block-configuration "BlockPublicPolicy=false,..."
```

### KMS Operations

```bash
aws kms list-keys
aws kms list-aliases
aws kms describe-key --key-id <KEY-ID>
aws kms get-key-policy --key-id <KEY-ID> --policy-name default
aws kms put-key-policy --key-id <KEY-ID> --policy-name default --policy file://policy.json
```

### IAM Operations

```bash
aws iam list-users
aws iam list-attached-user-policies --user-name <USER>
aws iam put-user-policy --user-name <USER> --policy-name <NAME> --policy-document file://policy.json
aws iam create-access-key --user-name <USER>
```

### Account Info

```bash
aws sts get-caller-identity
```

---

## Best Practices Learned

- ✅ **Principle of Least Privilege:** Grant only necessary permissions
- ✅ **Enforce HTTPS:** Use `aws:SecureTransport` condition
- ✅ **Enable Bucket Key:** Reduce KMS costs
- ✅ **Use CMK for Sensitive Data:** Better control and audit trail
- ✅ **Separate Concerns:** Different users = different permissions
- ✅ **Resource-Based for Resources:** Use bucket/key policies for resource control
- ✅ **Identity-Based for Users:** Use IAM policies for user permissions
- ✅ **Test Incrementally:** Add permissions step-by-step to understand each layer

---

## Lessons Learned

### 1. Permission Evaluation Order

```
Request → Block Public Access → IAM Policy → Bucket Policy → KMS Policy
          (if blocked, deny)    (must allow)  (must allow)   (must allow)
                                      ↓
                                 All must allow = ✅ Access Granted
                                 Any deny = ❌ Access Denied
```

### 2. When Bucket Policies Don't Work

- Check if IAM admin policies are overriding
- Verify Block Public Access isn't blocking
- For encryption, ensure KMS permissions exist

### 3. KMS is Separate from S3

- S3 permissions ≠ Encryption permissions
- Always grant both S3 and KMS permissions
- S3 service itself needs KMS permissions

### 4. Testing Strategy

- Start with one user, one bucket, one key
- Remove all admin permissions for clean testing
- Add permissions incrementally
- Verify each step with CLI commands

---

## Next Steps to Explore

- S3 Versioning with KMS encryption
- S3 Lifecycle Policies for cost optimization
- Cross-Region Replication with KMS
- S3 Access Logs for audit trails
- CloudTrail for KMS API monitoring
- S3 Object Lock for compliance
- IAM Roles for EC2/Lambda accessing S3
- Cross-Account S3 Access with bucket policies

---

## Summary

### Today's Achievement:

- ✅ Configured granular S3 bucket policies
- ✅ Implemented user-specific permissions
- ✅ Set up KMS Customer Managed Key encryption
- ✅ Understood IAM vs Bucket vs KMS policies
- ✅ Solved real-world access denied issues
- ✅ Learned cost optimization with Bucket Key
- ✅ Mastered AWS CLI for S3 and KMS operations

### Key Takeaway:

**AWS security is layered. Understanding the interaction between IAM policies, bucket policies, and KMS key policies is crucial for troubleshooting access issues and implementing secure architectures.**

---

## Placeholders Used in This Document

For security purposes, the following sensitive information has been masked:

### Replaced with Placeholders:
- `<ACCOUNT_ID>` - Your AWS Account ID (originally a 12-digit number)
- `<KEY_ID>` - KMS Key ID (originally a UUID format like `9e2992ba-130b-4f8e-...`)
- `<APP_ROLE_NAME>` - Application role name
- `<REPLICATION_ROLE_NAME>` - Replication role name
- `<USERNAME>` - IAM user names
- `<profile-name>` - AWS CLI profile names

### Replaced with Example Values:
- **Account ID:** All instances replaced with `123456789012` (example account)
- **KMS Key ID:** Replaced with `abcd1234-5678-90ef-ghij-...` (example key)
- **Bucket Name:** Original bucket name changed to `my-test-bucket`
- **IAM Users:** Original user names changed to `iam-user-1`, `iam-user-2`

**To use these examples:**
Replace placeholders and example values with your actual AWS resource identifiers when implementing in your environment.

**⚠️ Important:** Never share your actual AWS account IDs, KMS key IDs, or IAM credentials publicly.

---

## License

This document is created for educational purposes.

**Topics:** AWS S3, AWS KMS, IAM, Security, Cloud Architecture  
**Status:** Sanitized for public sharing
