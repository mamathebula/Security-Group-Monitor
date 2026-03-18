# Security Group Monitor

Automated AWS Security Group compliance monitoring and remediation. Detects and automatically revokes open CIDR rules (0.0.0.0/0 and ::/0) in real-time.

Available on AWS Marketplace as a CloudFormation template with AMI-based licensing.

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
│  ┌─────────────┐    ┌──────────────┐    ┌───────────────┐   │
│  │ EC2 License  │    │  CloudTrail   │    │  EventBridge   │   │
│  │ Server       │    │  (API Events) │───▶│  (Rules)       │   │
│  │ (Billing)    │    └──────────────┘    └───────┬───────┘   │
│  └─────────────┘                                 │           │
│                                                  ▼           │
│                                         ┌───────────────┐   │
│                                         │    Lambda      │   │
│                                         │  (Monitor &    │   │
│                                         │   Remediate)   │   │
│                                         └───────┬───────┘   │
│                                                  │           │
│                              ┌───────────────────┼──────┐   │
│                              ▼                   ▼      │   │
│                     ┌──────────────┐    ┌────────────┐  │   │
│                     │  SNS (Email   │    │ EC2 API    │  │   │
│                     │  Alerts)      │    │ (Revoke    │  │   │
│                     └──────────────┘    │  Rules)    │  │   │
│                                         └────────────┘  │   │
│                                                         │   │
└──────────────────────────────────────────────────────────────┘
```

### How It Works

1. A user creates or modifies a security group
2. CloudTrail captures the API call
3. EventBridge detects the event and triggers the Lambda function
4. Lambda inspects the security group for open CIDR rules (0.0.0.0/0 or ::/0)
5. If violations found:
   - Sends email alert via SNS
   - Automatically revokes the open rules (if auto-remediate is enabled)
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

- AWS Account with appropriate permissions
- Valid email address for alerts
- VPC and Subnet for the license server
- AWS Marketplace subscription (for Marketplace version)

### Parameters

| Parameter | Description | Default | Required |
|-----------|-------------|---------|----------|
| ImageId | AMI ID for license server | - | Yes |
| VpcId | VPC for license server | - | Yes |
| SubnetId | Subnet for license server | - | Yes |
| KeyName | SSH Key Pair (optional) | - | No |
| InstanceType | License server instance type | t3.micro | Yes |
| AdminEmail | Email for security alerts | - | Yes |
| AutoRemediate | Auto-revoke open rules | true | Yes |
| RunInitialScan | Scan existing security groups on deploy | true | Yes |

### Deploy via AWS Console

1. Go to AWS CloudFormation Console
2. Click "Create stack" → "With new resources"
3. Upload `security-group-monitor-marketplace.yaml`
4. Fill in parameters
5. Click "Next" → "Next" → Check "I acknowledge..." → "Submit"
6. Wait for stack to reach "CREATE_COMPLETE"
7. Confirm the SNS email subscription (check your inbox)

---

## Resources Created

| Resource | Type | Purpose |
|----------|------|---------|
| LicenseServer | EC2 Instance | AWS Marketplace billing/licensing |
| LicenseServerSecurityGroup | Security Group | Network access for license server |
| LicenseServerRole | IAM Role | Permissions for license server |
| LicenseServerInstanceProfile | Instance Profile | Attaches role to EC2 |
| SecurityGroupMonitorFunction | Lambda Function | Real-time monitoring & remediation |
| InitialScanFunction | Lambda Function | Scans all existing security groups |
| TriggerInitialScanFunction | Lambda Function | Triggers initial scan on deploy |
| SecurityMonitorRole | IAM Role | Permissions for Lambda functions |
| TriggerScanRole | IAM Role | Permissions for trigger Lambda |
| SecurityAlertTopic | SNS Topic | Email alert notifications |
| SecurityGroupEventRule | EventBridge Rule | Triggers Lambda on SG changes |
| SecurityMonitorTrail | CloudTrail | Captures API events |
| TrailBucket | S3 Bucket | Stores CloudTrail logs |
| SecurityMonitorLogGroup | CloudWatch Logs | Lambda function logs |
| InitialScanLogGroup | CloudWatch Logs | Initial scan logs |
| TriggerScanLogGroup | CloudWatch Logs | Trigger function logs |

All resources are tagged with:
- `Name=do-not-delete`
- `ManagedBy=CloudFormation`
- `Purpose=SecurityGroupMonitoring`

---

## Features

### Auto-Remediation
When enabled, the Lambda function automatically revokes any ingress rule that allows traffic from 0.0.0.0/0 or ::/0. This happens in real-time within seconds of the rule being created.

### Tag-Based Whitelisting
Tag any security group with `Name=testing` (case-insensitive) to exclude it from monitoring and remediation. Useful for development and testing environments.

If the `Name=testing` tag is removed from a whitelisted security group, the monitor will re-scan it and remediate any open rules.

### Initial Scan
On deployment, the template can run an initial scan of all existing security groups in the account. This catches any pre-existing violations that were created before the monitor was deployed.

### Email Alerts
All violations trigger email notifications via SNS. Alerts include:
- Security group ID and name
- VPC ID
- Open rules detected (protocol, ports, CIDR)
- Remediation status (revoked, whitelisted, or failed)

---

## Alert Examples

### Violation Detected and Remediated
```
Security Group Compliance Alert
================================
Timestamp: 2026-03-18 14:30:00 UTC
Action: REMEDIATED
Total Violations: 1

Details:
--------
1. Security Group: web-server-sg (sg-0123456789abcdef0)
   VPC: vpc-0123456789abcdef0
   Open Rules Detected:
     - Protocol: tcp, Ports: 22-22, CIDR: 0.0.0.0/0
   ✅ Status: OPEN RULES REVOKED
```

### Whitelisted Security Group
```
Security Group Compliance Alert
================================
Timestamp: 2026-03-18 14:30:00 UTC
Action: DETECTED (Whitelisted)
Total Violations: 1

Details:
--------
1. Security Group: test-sg (sg-0123456789abcdef0)
   VPC: vpc-0123456789abcdef0
   Open Rules Detected:
     - Protocol: -1, Ports: All-All, CIDR: 0.0.0.0/0
   ⚪ Status: WHITELISTED (Tag: Name=testing) - Not remediated
```

---

## Estimated Monthly Cost

| Instance Type | EC2 Cost | Other Services | Total |
|---------------|----------|----------------|-------|
| t3.nano | $3.80 | $1.34 | $5.14 |
| t3.micro | $7.59 | $1.34 | $8.93 |
| t3.medium | $30.37 | $1.34 | $31.71 |

Other services include: S3 (~$0.50), EBS (~$0.64), CloudWatch Logs (~$0.10), Lambda ($0.00 free tier), SNS ($0.00 free tier), EventBridge ($0.00), CloudTrail ($0.00 first trail).

See `COST_BREAKDOWN.md` for detailed cost analysis.

---

## Cleanup / Deletion

### Quick Method (Console)
1. Go to S3 Console → Empty the `security-monitor-trail-*` bucket
2. Go to CloudFormation Console → Select stack → Delete

### If Deletion Fails
See `CONSOLE_CLEANUP_GUIDE.md` for step-by-step manual cleanup instructions.

---

## Important Notes

### EC2 Instance Must Keep Running
The EC2 license server is required by AWS Marketplace for billing. Do NOT stop or terminate it. See `EC2_INSTANCE_REQUIREMENT.md` for details.

### CloudTrail Requirement
The template creates a CloudTrail trail to capture security group API events. This is required for EventBridge to detect changes in real-time.

### S3 Bucket Lifecycle
CloudTrail logs in S3 are automatically deleted after 7 days to minimize storage costs.

### CloudWatch Log Retention
All log groups have 7-day retention to minimize costs.

---

## Files in This Repository

| File | Description |
|------|-------------|
| `security-group-monitor-marketplace.yaml` | Main CloudFormation template (use this) |
| `security-group-monitor.yaml` | Original standalone version (no Marketplace) |
| `README.md` | This file |
| `MARKETPLACE_DEPLOYMENT_GUIDE.md` | Step-by-step Marketplace submission guide |
| `MARKETPLACE_METADATA_GUIDE.md` | Required metadata for Marketplace listing |
| `AMI_SETUP_INSTRUCTIONS.md` | How to create and configure the AMI |
| `COST_BREAKDOWN.md` | Detailed cost analysis and pricing strategy |
| `CONSOLE_CLEANUP_GUIDE.md` | How to delete all resources (console only) |
| `EC2_INSTANCE_REQUIREMENT.md` | Why the EC2 instance must keep running |
| `DELETION_PROTECTION_GUIDE.md` | Resource protection information |
| `MARKETPLACE_TEMPLATE_CHANGES.md` | Changes made for Marketplace compatibility |
| `security-group-monitor-architecture.md` | Architecture documentation |

---

## Support

For issues or questions:
- Check CloudWatch Logs for Lambda function errors
- Verify SNS email subscription is confirmed
- Ensure CloudTrail is logging (check trail status)
- Verify EventBridge rule is enabled

---

## Support & Contact

- **Email:** [email]
- **WhatsApp:** [phone_number]

For issues, feature requests, or general inquiries, reach out via email or WhatsApp.

---

## License

© 2026 Security Group Monitor™. All rights reserved.

Security Group Monitor is a trademark. Unauthorized reproduction, distribution, or modification of this software is strictly prohibited.

Available through AWS Marketplace.
