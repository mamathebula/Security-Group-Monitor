# Deletion Protection Guide

## Does `Name=do-not-delete` Tag Work?

**NO.** Tags are just labels. AWS doesn't enforce them.

```yaml
Tags:
  - Key: Name
    Value: do-not-delete  # ← This does NOTHING to prevent deletion
```

Anyone with delete permissions can still delete the resource.

---

## How to ACTUALLY Prevent Deletion

### 1. EC2 Termination Protection

**Add to your template:**
```yaml
LicenseServer:
  Type: AWS::EC2::Instance
  Properties:
    ImageId: !Ref ImageId
    InstanceType: t3.medium
    DisableApiTermination: true  # ← Prevents termination via API/Console
```

**What it does:**
- ✅ Prevents accidental termination in console
- ✅ Prevents termination via AWS CLI
- ✅ Must be explicitly disabled before deletion
- ❌ Can still be disabled by someone with permissions

**To delete:**
```bash
# Must first disable protection
aws ec2 modify-instance-attribute \
  --instance-id i-xxxxx \
  --no-disable-api-termination

# Then can terminate
aws ec2 terminate-instances --instance-ids i-xxxxx
```

---

### 2. CloudFormation Stack Protection

**Enable when deploying:**
```bash
aws cloudformation create-stack \
  --stack-name security-monitor \
  --template-body file://template.yaml \
  --enable-termination-protection  # ← Protects entire stack
```

**Or in Console:**
- Create stack
- Stack options → Termination protection → Enable

**What it does:**
- ✅ Prevents stack deletion
- ✅ Prevents all resources in stack from being deleted
- ✅ Must be explicitly disabled before deletion

**To delete:**
```bash
# Must first disable protection
aws cloudformation update-termination-protection \
  --stack-name security-monitor \
  --no-enable-termination-protection

# Then can delete
aws cloudformation delete-stack --stack-name security-monitor
```

---

### 3. S3 Bucket Deletion Protection

**Add to S3 bucket:**
```yaml
TrailBucket:
  Type: AWS::S3::Bucket
  DeletionPolicy: Retain  # ← Keeps bucket even if stack is deleted
  Properties:
    BucketName: !Sub 'security-monitor-trail-${AWS::AccountId}-${AWS::Region}'
```

**What it does:**
- ✅ Bucket survives stack deletion
- ✅ Data is preserved
- ❌ Bucket can still be deleted manually

---

### 4. IAM Policy (Deny Deletion)

**Create a policy that denies deletion:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": [
        "ec2:TerminateInstances",
        "cloudformation:DeleteStack",
        "s3:DeleteBucket"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Name": "do-not-delete"
        }
      }
    }
  ]
}
```

**What it does:**
- ✅ Actually enforces the tag
- ✅ Prevents deletion by anyone without override permissions
- ✅ Works across all resources with the tag

**Attach to:**
- IAM users
- IAM roles
- Service Control Policies (SCPs)

---

### 5. AWS Backup (Best for Data)

**For S3, RDS, EBS:**
```yaml
BackupPlan:
  Type: AWS::Backup::BackupPlan
  Properties:
    BackupPlan:
      BackupPlanName: security-monitor-backup
      BackupPlanRule:
        - RuleName: DailyBackup
          TargetBackupVault: !Ref BackupVault
          ScheduleExpression: "cron(0 2 * * ? *)"
          Lifecycle:
            DeleteAfterDays: 30
```

**What it does:**
- ✅ Automatic backups
- ✅ Can restore even if deleted
- ✅ Protects against accidental deletion

---

### 6. Resource Access Manager (RAM)

**Share resources to prevent deletion:**
- Share with another account
- Original account can't delete while shared
- Requires coordination to delete

---

## Recommended Protection Strategy

### For Your Security Monitor Product:

**1. EC2 Instance (License Server)**
```yaml
LicenseServer:
  Type: AWS::EC2::Instance
  Properties:
    DisableApiTermination: true  # ← Add this
```

**2. CloudFormation Stack**
```bash
# Deploy with protection
aws cloudformation create-stack \
  --enable-termination-protection
```

**3. S3 Bucket (CloudTrail Logs)**
```yaml
TrailBucket:
  Type: AWS::S3::Bucket
  DeletionPolicy: Retain  # ← Add this
```

**4. IAM Policy (Optional but Recommended)**
```json
{
  "Effect": "Deny",
  "Action": [
    "ec2:TerminateInstances",
    "cloudformation:DeleteStack"
  ],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "aws:ResourceTag/ManagedBy": "CloudFormation",
      "aws:ResourceTag/Purpose": "SecurityGroupMonitoring"
    }
  }
}
```

---

## What Each Protection Level Gives You

| Protection Method | Prevents Accidental Deletion | Prevents Malicious Deletion | Survives Stack Deletion |
|-------------------|------------------------------|----------------------------|------------------------|
| Tags only | ❌ No | ❌ No | ❌ No |
| EC2 Termination Protection | ✅ Yes | ⚠️ Partial | ❌ No |
| Stack Termination Protection | ✅ Yes | ⚠️ Partial | N/A |
| DeletionPolicy: Retain | ❌ No | ❌ No | ✅ Yes |
| IAM Deny Policy | ✅ Yes | ✅ Yes | ⚠️ Partial |
| AWS Backup | ⚠️ Recoverable | ⚠️ Recoverable | ✅ Yes |

---

## Best Practice: Layer Multiple Protections

**For critical resources:**
1. ✅ EC2 Termination Protection
2. ✅ Stack Termination Protection
3. ✅ DeletionPolicy: Retain (for data)
4. ✅ IAM Deny Policy
5. ✅ Tags (for visibility)
6. ✅ AWS Backup (for recovery)

**For your product:**
```yaml
# 1. EC2 Protection
LicenseServer:
  Properties:
    DisableApiTermination: true
    Tags:
      - Key: Name
        Value: do-not-delete  # Visual warning
      - Key: Protected
        Value: "true"

# 2. S3 Protection
TrailBucket:
  DeletionPolicy: Retain
  Properties:
    Tags:
      - Key: Name
        Value: do-not-delete

# 3. Deploy with stack protection
# aws cloudformation create-stack --enable-termination-protection
```

---

## How to Update Your Template

### Add EC2 Termination Protection:

Find this section in your template:
```yaml
LicenseServer:
  Type: AWS::EC2::Instance
  Properties:
    ImageId: !Ref ImageId
    InstanceType: t3.medium
    # Add this line:
    DisableApiTermination: true
```

### Add S3 Retention:

Find this section:
```yaml
TrailBucket:
  Type: AWS::S3::Bucket
  # Add this line:
  DeletionPolicy: Retain
  Properties:
    BucketName: !Sub 'security-monitor-trail-${AWS::AccountId}-${AWS::Region}'
```

---

## Testing Deletion Protection

### Test EC2 Protection:
```bash
# Try to terminate (should fail)
aws ec2 terminate-instances --instance-ids i-xxxxx

# Expected error:
# An error occurred (OperationNotPermitted) when calling the TerminateInstances operation: 
# The instance 'i-xxxxx' may not be terminated. Modify its 'disableApiTermination' instance attribute and try again.
```

### Test Stack Protection:
```bash
# Try to delete stack (should fail)
aws cloudformation delete-stack --stack-name security-monitor

# Expected error:
# An error occurred (ValidationError) when calling the DeleteStack operation: 
# Stack [security-monitor] cannot be deleted while TerminationProtection is enabled
```

---

## Summary

**Tags alone:** Do nothing
**Real protection:** Requires specific AWS features
**Best approach:** Layer multiple protections
**Your product:** Should use EC2 + Stack + S3 protection

Want me to update your template with these protections?
