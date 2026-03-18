# Security Group Monitor™ - Standalone Edition

Automated AWS Security Group compliance monitoring and remediation. Detects and automatically revokes open CIDR rules (0.0.0.0/0 and ::/0) in real-time.

This is the standalone (non-Marketplace) version. It is 100% serverless — no EC2 instances required.

---

## What It Does

- Monitors all security groups in your AWS account in real-time
- Detects open ingress rules allowing traffic from 0.0.0.0/0 (IPv4) or ::/0 (IPv6)
- Automatically revokes dangerous open rules (optional)
- Sends email alerts via SNS when violations are found
- Runs an initial scan on deployment to catch existing violations
- Supports tag-based whitelisting (tag a security group with `Name=testing` to skip it)

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    AWS Account                                │
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌───────────────┐  │
│  │  CloudTrail   │    │  EventBridge  │    │    Lambda     │  │
│  │  (Captures    │───▶│  (Detects SG  │───▶│  (Scans &    │  │
│  │   API calls)  │    │   changes)    │    │   Remediates) │  │
│  └──────────────┘    └──────────────┘    └───────┬───────┘  │
│                                                   │          │
│                                    ┌──────────────┼─────┐   │
│                                    ▼              ▼     │   │
│                           ┌──────────────┐  ┌────────┐  │   │
│                           │  SNS (Email   │  │ EC2 API│  │   │
│                           │  Alerts)      │  │(Revoke)│  │   │
│                           └──────────────┘  └────────┘  │   │
│                                                         │   │
└──────────────────────────────────────────────────────────────┘
```

### Flow

1. User creates or modifies a security group
2. CloudTrail captures the API call
3. EventBridge detects the event and triggers Lambda
4. Lambda inspects the security group for open CIDR rules
5. If violations found:
   - Sends email alert via SNS
   - Automatically revokes the open rules (if auto-remediate enabled)
6. If security group has tag `Name=testing`, it is whitelisted and skipped

### Monitored Events

- `AuthorizeSecurityGroupIngress` - New inbound rules added
- `AuthorizeSecurityGroupEgress` - New outbound rules added
- `CreateSecurityGroup` - New security group created
- `ModifySecurityGroupRules` - Existing rules modified
- `DeleteTags` - Tag removed (re-scans if whitelist tag removed)

---

## Deployment

### Prerequisites

- AWS Account
- Valid email address for alerts
- CloudFormation permissions

### Parameters

| Parameter | Description | Default | Required |
|-----------|-------------|---------|----------|
| AdminEmail | Email for security alerts | - | Yes |
| AutoRemediate | Auto-revoke open rules | true | Yes |
| RunInitialScan | Scan existing security groups on deploy | true | Yes |

### Deploy via AWS Console

1. Go to CloudFormation Console: https://console.aws.amazon.com/cloudformation
2. Click "Create stack" → "With new resources (standard)"
3. Select "Upload a template file"
4. Upload `security-group-monitor.yaml`
5. Click "Next"
6. Fill in:
   - Stack name: `security-group-monitor` (or any name)
   - AdminEmail: Your email address
   - AutoRemediate: `true` (recommended)
   - RunInitialScan: `true` (recommended)
7. Click "Next" → "Next"
8. Check "I acknowledge that AWS CloudFormation might create IAM resources with custom names"
9. Click "Submit"
10. Wait for stack to reach "CREATE_COMPLETE"
11. Check your email and confirm the SNS subscription

---

## Resources Created

| Resource | Type | Purpose |
|----------|------|---------|
| SecurityGroupMonitorFunction | Lambda Function | Real-time monitoring & remediation |
| InitialScanFunction | Lambda Function | Scans all existing security groups on deploy |
| TriggerInitialScanFunction | Lambda Function | Triggers initial scan (CloudFormation custom resource) |
| SecurityMonitorRole | IAM Role | Permissions for monitoring Lambda |
| TriggerScanRole | IAM Role | Permissions for trigger Lambda |
| SecurityAlertTopic | SNS Topic | Email alert notifications |
| SecurityGroupEventRule | EventBridge Rule | Triggers Lambda on SG changes |
| SecurityMonitorTrail | CloudTrail | Captures API events |
| TrailBucket | S3 Bucket | Stores CloudTrail logs (7-day retention) |
| TrailBucketPolicy | S3 Bucket Policy | Allows CloudTrail to write logs |
| SecurityMonitorLogGroup | CloudWatch Logs | Main Lambda logs (7-day retention) |
| InitialScanLogGroup | CloudWatch Logs | Initial scan logs (7-day retention) |
| TriggerScanLogGroup | CloudWatch Logs | Trigger function logs (7-day retention) |

All resources are tagged with:
- `Name=do-not-delete`
- `ManagedBy=CloudFormation`
- `Purpose=SecurityGroupMonitoring`

---

## Features

### Auto-Remediation
When enabled, Lambda automatically revokes any ingress rule allowing traffic from 0.0.0.0/0 or ::/0. Happens in real-time within seconds of the rule being created.

### Tag-Based Whitelisting
Tag any security group with `Name=testing` (case-insensitive) to exclude it from monitoring and remediation.

If the `Name=testing` tag is later removed, the monitor re-scans that security group and remediates any open rules.

### Initial Scan
On deployment, scans all existing security groups in the account using pagination. Catches pre-existing violations created before the monitor was deployed.

### Email Alerts
All violations trigger email notifications. Alerts include:
- Security group ID and name
- VPC ID
- Open rules detected (protocol, ports, CIDR)
- Remediation status (revoked, whitelisted, or failed)

---

## Estimated Monthly Cost

| Service | Monthly Cost | Notes |
|---------|--------------|-------|
| S3 (CloudTrail logs) | ~$0.50 | 7-day lifecycle |
| CloudWatch Logs | ~$0.10 | 7-day retention |
| Lambda | $0.00 | Free tier |
| CloudTrail | $0.00 | First trail free |
| SNS | $0.00 | First 1,000 emails free |
| EventBridge | $0.00 | Free for AWS events |
| **Total** | **~$0.60/month** | |

This is a fully serverless solution — no EC2 instances, no ongoing compute costs.

---

## Differences from Marketplace Edition

| Feature | Standalone (This) | Marketplace Edition |
|---------|-------------------|---------------------|
| EC2 Instance | None | Required (license server) |
| Monthly Cost | ~$0.60 | ~$5.14+ |
| AWS Marketplace | Not listed | Listed for sale |
| Billing | Free (self-deploy) | Paid subscription |
| VPC/Subnet Required | No | Yes (for EC2) |
| Parameters | 3 | 8 |

Use this standalone version for personal/internal use. Use the Marketplace edition (`security-group-monitor-marketplace.yaml`) for commercial distribution.

---

## Cleanup / Deletion

### Via Console

1. **Empty the S3 bucket first:**
   - Go to S3 Console: https://console.aws.amazon.com/s3
   - Find bucket: `security-monitor-trail-[ACCOUNT-ID]-[REGION]`
   - Click "Empty" → Type `permanently delete` → Click "Empty"

2. **Delete the CloudFormation stack:**
   - Go to CloudFormation Console: https://console.aws.amazon.com/cloudformation
   - Select your stack → Click "Delete" → Confirm

---

## Testing

### Test 1: Create an Open Security Group

1. Go to EC2 Console → Security Groups → Create Security Group
2. Add inbound rule: Type=All Traffic, Source=0.0.0.0/0
3. Save
4. Within seconds:
   - You should receive an email alert
   - The open rule should be automatically revoked (if auto-remediate is on)

### Test 2: Whitelist a Security Group

1. Create a security group
2. Add tag: Key=`Name`, Value=`testing`
3. Add inbound rule: Type=All Traffic, Source=0.0.0.0/0
4. You should receive an alert, but the rule should NOT be revoked
5. Remove the `Name=testing` tag
6. The monitor re-scans and revokes the open rule

### Test 3: Manual Lambda Invocation

1. Go to Lambda Console
2. Find `SecurityGroupMonitorFunction`
3. Click "Test"
4. Use empty event: `{}`
5. This triggers a full scan of all security groups

---

## Outputs

After deployment, CloudFormation provides these outputs:

| Output | Description |
|--------|-------------|
| LambdaFunctionArn | ARN of the monitoring Lambda function |
| SNSTopicArn | ARN of the SNS alert topic |
| CloudTrailName | Name of the CloudTrail trail |
| TrailBucketName | Name of the S3 bucket for logs |
| InitialScanFunctionArn | ARN of the initial scan Lambda (if enabled) |

---

## Troubleshooting

### Not Receiving Email Alerts
- Check your email for the SNS subscription confirmation
- Look in spam/junk folder
- Verify the email address in CloudFormation parameters

### Rules Not Being Revoked
- Check `AutoRemediate` parameter is set to `true`
- Check if the security group has `Name=testing` tag (whitelisted)
- Check Lambda CloudWatch Logs for errors

### EventBridge Not Triggering
- Verify CloudTrail is logging (check trail status)
- Ensure EventBridge rule is enabled
- Check Lambda permissions

### Initial Scan Didn't Run
- Verify `RunInitialScan` parameter is set to `true`
- Check TriggerInitialScanFunction CloudWatch Logs
- Check InitialScanFunction CloudWatch Logs

---

## Support & Contact

- **Email:** [mathebula000@gmail.com]
- **WhatsApp:** [+27797749367]

For issues, feature requests, or general inquiries, reach out via email or WhatsApp.

---

## License

© 2026 Security Group Monitor™. All rights reserved.

Security Group Monitor is a trademark. Unauthorized reproduction, distribution, or modification of this software is strictly prohibited.
