# AWS Marketplace Deployment Guide
## Security Group Monitor - Complete Process (Console-Based)

This guide walks you through listing your Security Group Monitor on AWS Marketplace using the AWS Console.

**Estimated Timeline:** 2-4 weeks
**Prerequisites:** AWS Account, basic Linux knowledge

---

## Phase 1: Seller Registration (Week 1)

### Step 1.1: Register as AWS Marketplace Seller

1. **Go to AWS Marketplace Management Portal**
   - Visit: https://aws.amazon.com/marketplace/management/
   - Sign in with your AWS account

2. **Start Registration**
   - Click "Register as a Seller"
   - Choose seller type: "Individual" or "Company"

3. **Complete Seller Profile**
   - Company/Individual name
   - Contact information
   - Business address
   - Phone number

4. **Tax Information**
   - US Sellers: Complete W-9 form
   - Non-US Sellers: Complete W-8 form
   - Provide Tax ID (EIN or SSN)

5. **Banking Information**
   - Bank account details for payments
   - AWS uses ACH for US banks
   - Wire transfer for international

6. **Accept Agreements**
   - Read and accept AWS Marketplace Seller Agreement
   - Review fee structure (AWS takes ~3% of sales)

7. **Submit and Wait**
   - Submit registration
   - Wait for approval email (1-3 business days)

**Cost:** Free to register

---

## Phase 2: Create the License Server AMI (Week 1-2)

### Step 2.1: Launch Base EC2 Instance

1. **Open EC2 Console**
   - Go to: https://console.aws.amazon.com/ec2/
   - Select region: **us-east-1** (required for Marketplace submission)

2. **Launch Instance**
   - Click "Launch Instance"
   - Name: `security-monitor-build`

3. **Choose AMI**
   - Select "Amazon Linux 2 AMI (HVM)"
   - Architecture: 64-bit (x86)
   - Click "Select"

4. **Choose Instance Type**
   - Select: **t3.nano** (cheapest option, ~$0.0052/hour)
   - Click "Next: Configure Instance Details"

5. **Configure Instance**
   - Leave defaults
   - Click "Next: Add Storage"

6. **Add Storage**
   - Size: 8 GB (default is fine)
   - Volume Type: General Purpose SSD (gp3)
   - Click "Next: Add Tags"

7. **Add Tags**
   - Key: `Name`, Value: `security-monitor-build`
   - Key: `Purpose`, Value: `AMI-Creation`
   - Click "Next: Configure Security Group"

8. **Configure Security Group**
   - Create new security group
   - Name: `security-monitor-build-sg`
   - Rule: SSH (port 22) from "My IP"
   - Click "Review and Launch"

9. **Select Key Pair**
   - Choose existing key pair OR create new one
   - Download .pem file if creating new
   - Check acknowledgment box
   - Click "Launch Instances"

10. **Wait for Instance**
    - Click "View Instances"
    - Wait until "Instance State" = "Running"
    - Note the "Public IPv4 address"

**Keep this instance running - you'll configure it next**

---

### Step 2.2: Connect to Instance and Configure

1. **Connect via SSH**
   
   **On Mac/Linux:**
   ```bash
   chmod 400 your-key.pem
   ssh -i your-key.pem ec2-user@YOUR_INSTANCE_IP
   ```
   
   **On Windows (using PuTTY):**
   - Convert .pem to .ppk using PuTTYgen
   - Open PuTTY
   - Host: `ec2-user@YOUR_INSTANCE_IP`
   - Auth: Browse to .ppk file
   - Click "Open"

   **Or use EC2 Instance Connect (easier):**
   - In EC2 Console, select your instance
   - Click "Connect" button
   - Choose "EC2 Instance Connect"
   - Click "Connect" (opens browser terminal)

2. **Once connected, run these commands:**

```bash
# Update system
sudo yum update -y

# Install dependencies
sudo yum install -y python3 python3-pip jq

# Create directory structure
sudo mkdir -p /opt/security-monitor
sudo mkdir -p /var/log/security-monitor
sudo chown ec2-user:ec2-user /var/log/security-monitor

echo "✓ System updated and directories created"
```

3. **Create the validation script:**

```bash
sudo tee /opt/security-monitor/validate.sh > /dev/null << 'EOF'
#!/bin/bash
# Validates Marketplace subscription via product code

LOG_FILE="/var/log/security-monitor/validation.log"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

# Get instance metadata
INSTANCE_ID=$(ec2-metadata --instance-id 2>/dev/null | cut -d " " -f 2)
REGION=$(ec2-metadata --availability-zone 2>/dev/null | cut -d " " -f 2 | sed 's/[a-z]$//')
PRODUCT_CODE=$(ec2-metadata --product-codes 2>/dev/null | cut -d " " -f 2)

log "Starting validation check"
log "Instance ID: $INSTANCE_ID"
log "Region: $REGION"

# Check for product code
if [ -z "$PRODUCT_CODE" ] || [ "$PRODUCT_CODE" == "not" ]; then
    log "WARNING: No product code detected. This may be a test deployment."
    exit 0
else
    log "Valid Marketplace subscription detected: $PRODUCT_CODE"
    
    # Send metric to CloudWatch
    aws cloudwatch put-metric-data \
        --namespace "SecurityMonitor/License" \
        --metric-name "SubscriptionActive" \
        --value 1 \
        --dimensions InstanceId=$INSTANCE_ID \
        --region $REGION 2>/dev/null || true
    
    exit 0
fi
EOF

sudo chmod +x /opt/security-monitor/validate.sh
echo "✓ Validation script created"
```

4. **Create the health check script:**

```bash
sudo tee /opt/security-monitor/health-check.sh > /dev/null << 'EOF'
#!/bin/bash
# Periodic health check and monitoring

LOG_FILE="/var/log/security-monitor/health.log"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOG_FILE"
}

# Get metadata
INSTANCE_ID=$(ec2-metadata --instance-id 2>/dev/null | cut -d " " -f 2)
REGION=$(ec2-metadata --availability-zone 2>/dev/null | cut -d " " -f 2 | sed 's/[a-z]$//')
PRODUCT_CODE=$(ec2-metadata --product-codes 2>/dev/null | cut -d " " -f 2)

log "Health check started"
log "Instance: $INSTANCE_ID | Region: $REGION | Product: $PRODUCT_CODE"

# Send health metric
aws cloudwatch put-metric-data \
    --namespace "SecurityMonitor/License" \
    --metric-name "LicenseServerHealth" \
    --value 1 \
    --dimensions InstanceId=$INSTANCE_ID \
    --region $REGION 2>/dev/null || true

log "Health check completed"
EOF

sudo chmod +x /opt/security-monitor/health-check.sh
echo "✓ Health check script created"
```

5. **Create systemd service:**

```bash
sudo tee /etc/systemd/system/security-monitor.service > /dev/null << 'EOF'
[Unit]
Description=Security Monitor License Service
After=network.target cloud-final.service

[Service]
Type=oneshot
ExecStart=/opt/security-monitor/validate.sh
ExecStartPost=/opt/security-monitor/health-check.sh
RemainAfterExit=yes
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

# Enable service
sudo systemctl daemon-reload
sudo systemctl enable security-monitor.service

echo "✓ Systemd service created and enabled"
```

6. **Create hourly health check cron job:**

```bash
echo "0 * * * * /opt/security-monitor/health-check.sh" | sudo crontab -
echo "✓ Cron job created"
```

7. **Create info file:**

```bash
sudo tee /opt/security-monitor/info.txt > /dev/null << 'EOF'
Security Group Monitor - License Server
Version: 1.0.0
Purpose: Validates AWS Marketplace subscription
Contact: your-email@example.com
Documentation: https://your-docs-url.com
EOF

echo "✓ Info file created"
```

8. **Test the scripts:**

```bash
# Test validation
sudo /opt/security-monitor/validate.sh

# Test health check
sudo /opt/security-monitor/health-check.sh

# Check logs
cat /var/log/security-monitor/validation.log
cat /var/log/security-monitor/health.log

# Test service
sudo systemctl start security-monitor.service
sudo systemctl status security-monitor.service
```

You should see output showing the scripts ran successfully.

9. **Clean up for AMI creation:**

```bash
# Clean package cache
sudo yum clean all
sudo rm -rf /var/cache/yum

# Remove temporary files
sudo rm -rf /tmp/*
sudo rm -rf /var/tmp/*

# Clear bash history
history -c
sudo rm -f /root/.bash_history
sudo rm -f /home/ec2-user/.bash_history

echo "✓ Cleanup complete - ready for AMI creation"
```

10. **Exit SSH:**

```bash
exit
```

---

### Step 2.3: Create AMI from Console

1. **Go to EC2 Console**
   - Navigate to "Instances"
   - Select your `security-monitor-build` instance

2. **Stop the Instance (Recommended)**
   - Click "Instance state" → "Stop instance"
   - Wait until state shows "Stopped" (~1-2 minutes)
   - This ensures a consistent AMI

3. **Create Image**
   - With instance selected, click "Actions"
   - Choose "Image and templates" → "Create image"

4. **Configure Image**
   - **Image name:** `security-group-monitor-v1.0.0`
   - **Image description:** `Security Group Monitor - Automated security group compliance monitoring and remediation. License server for AWS Marketplace.`
   - **No reboot:** Leave unchecked (already stopped)
   - **Instance volumes:** Leave defaults
   - **Tags:**
     - Key: `Name`, Value: `security-group-monitor`
     - Key: `Version`, Value: `1.0.0`
     - Key: `Purpose`, Value: `Marketplace-License-Server`

5. **Create Image**
   - Click "Create image"
   - Note the AMI ID (e.g., `ami-0123456789abcdef0`)
   - Click "Close"

6. **Wait for AMI**
   - Go to "AMIs" in left sidebar
   - Find your AMI
   - Wait until "Status" = "Available" (5-10 minutes)
   - ☕ Take a coffee break

7. **Note Your AMI ID**
   - Copy the AMI ID - you'll need it later
   - Example: `ami-0123456789abcdef0`

---

### Step 2.4: Test Your AMI

1. **Launch Test Instance**
   - In AMIs section, select your new AMI
   - Click "Launch instance from AMI"
   - Name: `security-monitor-test`
   - Instance type: `t3.nano`
   - Key pair: Select your key
   - Security group: Allow SSH from your IP
   - Click "Launch instance"

2. **Connect and Verify**
   - Wait for instance to be "Running"
   - Connect via SSH or EC2 Instance Connect
   
   ```bash
   # Check service status
   sudo systemctl status security-monitor.service
   
   # Check logs
   cat /var/log/security-monitor/validation.log
   cat /var/log/security-monitor/health.log
   
   # Verify scripts exist
   ls -la /opt/security-monitor/
   ```

3. **Clean Up Test**
   - Exit SSH
   - In EC2 Console, select test instance
   - "Instance state" → "Terminate instance"

4. **Clean Up Build Instance**
   - Select your `security-monitor-build` instance
   - "Instance state" → "Terminate instance"
   - Confirm termination

**✓ Your AMI is ready for Marketplace!**

---

---

### Step 2.2: Configure the License Server

Create the setup script locally:

```bash
cat > setup-license-server.sh << 'SCRIPT_END'
#!/bin/bash
set -e

echo "=== Security Monitor License Server Setup ==="

# Update system
sudo yum update -y

# Install dependencies
sudo yum install -y aws-cli python3 python3-pip jq

# Create directory structure
sudo mkdir -p /opt/security-monitor
sudo mkdir -p /var/log/security-monitor
sudo chown ec2-user:ec2-user /var/log/security-monitor

# Create validation script
sudo tee /opt/security-monitor/validate.sh > /dev/null << 'EOF'
#!/bin/bash
# Validates Marketplace subscription via product code

LOG_FILE="/var/log/security-monitor/validation.log"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

# Get instance metadata
INSTANCE_ID=$(ec2-metadata --instance-id 2>/dev/null | cut -d " " -f 2)
REGION=$(ec2-metadata --availability-zone 2>/dev/null | cut -d " " -f 2 | sed 's/[a-z]$//')
PRODUCT_CODE=$(ec2-metadata --product-codes 2>/dev/null | cut -d " " -f 2)

log "Starting validation check"
log "Instance ID: $INSTANCE_ID"
log "Region: $REGION"

# Check for product code
if [ -z "$PRODUCT_CODE" ] || [ "$PRODUCT_CODE" == "not" ]; then
    log "WARNING: No product code detected. This may be a test deployment."
    # In production, you might want to disable functionality here
    exit 0
else
    log "Valid Marketplace subscription detected: $PRODUCT_CODE"
    
    # Send metric to CloudWatch
    aws cloudwatch put-metric-data \
        --namespace "SecurityMonitor/License" \
        --metric-name "SubscriptionActive" \
        --value 1 \
        --dimensions InstanceId=$INSTANCE_ID \
        --region $REGION 2>/dev/null || true
    
    exit 0
fi
EOF

sudo chmod +x /opt/security-monitor/validate.sh

# Create health check script
sudo tee /opt/security-monitor/health-check.sh > /dev/null << 'EOF'
#!/bin/bash
# Periodic health check and monitoring

LOG_FILE="/var/log/security-monitor/health.log"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOG_FILE"
}

# Get metadata
INSTANCE_ID=$(ec2-metadata --instance-id 2>/dev/null | cut -d " " -f 2)
REGION=$(ec2-metadata --availability-zone 2>/dev/null | cut -d " " -f 2 | sed 's/[a-z]$//')
PRODUCT_CODE=$(ec2-metadata --product-codes 2>/dev/null | cut -d " " -f 2)

log "Health check started"
log "Instance: $INSTANCE_ID | Region: $REGION | Product: $PRODUCT_CODE"

# Send health metric
aws cloudwatch put-metric-data \
    --namespace "SecurityMonitor/License" \
    --metric-name "LicenseServerHealth" \
    --value 1 \
    --dimensions InstanceId=$INSTANCE_ID \
    --region $REGION 2>/dev/null || true

log "Health check completed"
EOF

sudo chmod +x /opt/security-monitor/health-check.sh

# Create systemd service for startup validation
sudo tee /etc/systemd/system/security-monitor.service > /dev/null << 'EOF'
[Unit]
Description=Security Monitor License Service
After=network.target cloud-final.service

[Service]
Type=oneshot
ExecStart=/opt/security-monitor/validate.sh
ExecStartPost=/opt/security-monitor/health-check.sh
RemainAfterExit=yes
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

# Enable service
sudo systemctl daemon-reload
sudo systemctl enable security-monitor.service

# Create hourly cron job for health checks
echo "0 * * * * /opt/security-monitor/health-check.sh" | sudo crontab -

# Create info file
sudo tee /opt/security-monitor/info.txt > /dev/null << 'EOF'
Security Group Monitor - License Server
Version: 1.0.0
Purpose: Validates AWS Marketplace subscription and provides licensing
Contact: your-email@example.com
EOF

# Clean up for AMI creation
sudo yum clean all
sudo rm -rf /var/cache/yum
sudo rm -rf /tmp/*
sudo rm -rf /var/tmp/*
sudo rm -f /root/.bash_history
sudo rm -f /home/ec2-user/.bash_history

echo "=== Setup Complete ==="
echo "Ready to create AMI"
SCRIPT_END

chmod +x setup-license-server.sh
```

Copy and run the script on your instance:

```bash
# Copy script to instance
scp -i YOUR_KEY.pem setup-license-server.sh ec2-user@$PUBLIC_IP:/tmp/

# SSH to instance
ssh -i YOUR_KEY.pem ec2-user@$PUBLIC_IP

# Run setup script
sudo bash /tmp/setup-license-server.sh

# Test the scripts
sudo /opt/security-monitor/validate.sh
sudo /opt/security-monitor/health-check.sh

# Check logs
cat /var/log/security-monitor/validation.log
cat /var/log/security-monitor/health.log

# Exit SSH
exit
```

---

### Step 2.3: Create the AMI

```bash
# Stop the instance (recommended for consistent AMI)
aws ec2 stop-instances --instance-ids $INSTANCE_ID
aws ec2 wait instance-stopped --instance-ids $INSTANCE_ID

# Create AMI
AMI_NAME="security-group-monitor-v1.0.0-$(date +%Y%m%d)"
NEW_AMI_ID=$(aws ec2 create-image \
  --instance-id $INSTANCE_ID \
  --name "$AMI_NAME" \
  --description "Security Group Monitor - Automated security group compliance monitoring and remediation" \
  --tag-specifications 'ResourceType=image,Tags=[{Key=Name,Value=security-group-monitor},{Key=Version,Value=1.0.0}]' \
  --query 'ImageId' \
  --output text)

echo "AMI ID: $NEW_AMI_ID"

# Wait for AMI to be available (can take 5-10 minutes)
aws ec2 wait image-available --image-ids $NEW_AMI_ID

echo "AMI is ready: $NEW_AMI_ID"

# Clean up build instance
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
aws ec2 delete-security-group --group-id $SG_ID
```

---

### Step 2.4: Test Your AMI

```bash
# Launch test instance from your new AMI
TEST_INSTANCE=$(aws ec2 run-instances \
  --image-id $NEW_AMI_ID \
  --instance-type t3.nano \
  --key-name YOUR_KEY_NAME \
  --query 'Instances[0].InstanceId' \
  --output text)

# Wait and check logs
aws ec2 wait instance-running --instance-ids $TEST_INSTANCE

# Get IP and SSH
TEST_IP=$(aws ec2 describe-instances \
  --instance-ids $TEST_INSTANCE \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

ssh -i YOUR_KEY.pem ec2-user@$TEST_IP

# On the instance, verify:
sudo systemctl status security-monitor.service
cat /var/log/security-monitor/validation.log
cat /var/log/security-monitor/health.log

# Exit and terminate test instance
exit
aws ec2 terminate-instances --instance-ids $TEST_INSTANCE
```

---

## Phase 3: Update CloudFormation Template (Week 2)

### Step 3.1: Create Marketplace-Ready Template

I'll create this in the next file...

## Phase 3: Prepare CloudFormation Template (Week 2)

### Step 3.1: Update Your Template for Marketplace

Your current template needs to be modified to include the EC2 license server. I'll create the updated version for you.

**Key changes needed:**
1. Add EC2 instance resource (license server)
2. Add VPC/Subnet parameters (customer provides these)
3. Add ImageId parameter (AWS Marketplace will populate this)
4. Organize parameters for better UX
5. Add proper metadata for CloudFormation UI

---

### Step 3.2: Test Template Locally

1. **Go to CloudFormation Console**
   - Navigate to: https://console.aws.amazon.com/cloudformation/

2. **Create Stack**
   - Click "Create stack" → "With new resources"
   - Choose "Template is ready"
   - Upload your updated template file
   - Click "Next"

3. **Specify Stack Details**
   - Stack name: `security-monitor-test`
   - Fill in all parameters:
     - AdminEmail: your email
     - ImageId: Your AMI ID from Step 2.3
     - VpcId: Select a VPC
     - SubnetId: Select a subnet
     - AutoRemediate: true
     - RunInitialScan: true
   - Click "Next"

4. **Configure Stack Options**
   - Tags: Add any tags you want
   - Permissions: Leave default
   - Click "Next"

5. **Review**
   - Check "I acknowledge that AWS CloudFormation might create IAM resources"
   - Click "Submit"

6. **Wait for Completion**
   - Watch the "Events" tab
   - Wait until Status = "CREATE_COMPLETE" (~5-10 minutes)
   - Check "Outputs" tab for resource ARNs

7. **Verify Deployment**
   - Check EC2 console - license server should be running
   - Check Lambda console - functions should exist
   - Check your email - SNS subscription confirmation
   - Create a test security group with 0.0.0.0/0 rule
   - Wait for alert email

8. **Clean Up Test**
   - Select your stack
   - Click "Delete"
   - Confirm deletion
   - Wait for DELETE_COMPLETE

**✓ Template works! Ready for Marketplace submission**

---

## Phase 4: Prepare Supporting Materials (Week 2)

### Step 4.1: Architecture Diagram

You already have `security-group-monitor-architecture.md` but need to update it to include the EC2 license server.

**Requirements:**
- Format: PNG or JPG
- Size: 1100 x 700 pixels
- Use official AWS icons
- Show all resources deployed

**Tools to create diagram:**
- [draw.io](https://app.diagrams.net/) - Free, has AWS icons
- [Lucidchart](https://www.lucidchart.com/) - Free tier available
- [CloudCraft](https://www.cloudcraft.co/) - AWS-specific, free tier

**What to include:**
- EC2 License Server (t3.nano)
- Lambda Functions (3)
- EventBridge Rule
- CloudTrail
- S3 Bucket
- SNS Topic
- CloudWatch Logs
- IAM Roles
- VPC/Subnet (customer-provided)

### Step 4.2: Product Description

Create a document with:

**Short Description (200 chars):**
```
Automated security group monitoring and remediation. Detects and removes open CIDR rules (0.0.0.0/0) in real-time with customizable whitelisting.
```

**Long Description:**
```
Security Group Monitor provides automated, real-time monitoring of AWS Security Groups 
to detect and remediate compliance violations. 

KEY FEATURES:
• Real-time monitoring via EventBridge and CloudTrail
• Automatic remediation of open CIDR rules (0.0.0.0/0 and ::/0)
• Tag-based whitelisting (Name=testing)
• Email alerts via SNS
• Initial scan of existing security groups
• Multi-region support
• Serverless architecture (minimal cost)

DEPLOYMENT:
Deploys via CloudFormation in your AWS account. Includes:
- License server (t3.nano EC2 instance)
- Lambda functions for monitoring and remediation
- EventBridge rules for real-time detection
- CloudTrail for API event capture
- SNS topic for email alerts

PRICING:
- License: $X/month (billed through AWS Marketplace)
- AWS infrastructure: ~$5-10/month (EC2, Lambda, CloudTrail)

SUPPORT:
- Documentation: [your-url]
- Email: [your-email]
- Response time: 24-48 hours
```

**Highlights (bullet points):**
- Automated security compliance
- Real-time monitoring and remediation
- Customizable whitelisting
- Email alerts
- Easy CloudFormation deployment
- Minimal infrastructure cost

### Step 4.3: Usage Instructions

Create a PDF or Markdown document:

**Title:** Security Group Monitor - Deployment Guide

**Contents:**
1. Prerequisites
   - AWS Account
   - VPC with subnet
   - Email address for alerts
   
2. Deployment Steps
   - Subscribe on AWS Marketplace
   - Click "Launch CloudFormation Stack"
   - Fill in parameters
   - Wait for deployment
   - Confirm SNS subscription email
   
3. Configuration
   - Whitelisting security groups (add Name=testing tag)
   - Enabling/disabling auto-remediation
   - Viewing logs in CloudWatch
   
4. Monitoring
   - Check email for alerts
   - View CloudWatch metrics
   - Review Lambda logs
   
5. Troubleshooting
   - Common issues and solutions
   - How to contact support

### Step 4.4: Pricing Calculator Estimate

1. **Go to AWS Pricing Calculator**
   - Visit: https://calculator.aws/

2. **Add Services:**
   - EC2: 1 x t3.nano (730 hours/month)
   - Lambda: 1000 invocations/month, 128MB, 30s avg
   - CloudTrail: 1 trail
   - S3: 1GB storage, 1000 PUT requests
   - SNS: 100 notifications/month
   - CloudWatch Logs: 1GB ingestion

3. **Generate Estimate**
   - Click "Export" → "Public link"
   - Copy the URL
   - You'll provide this to AWS Marketplace

4. **Expected Monthly Cost:**
   - EC2 (t3.nano): ~$3.80
   - Lambda: ~$0.20
   - CloudTrail: ~$2.00
   - S3: ~$0.50
   - SNS: ~$0.10
   - CloudWatch: ~$0.50
   - **Total AWS cost: ~$7-10/month**
   - **Your license fee: $X/month** (you decide)

---

## Phase 5: AWS Marketplace Submission (Week 3)

### Step 5.1: Scan Your AMI

1. **Go to AWS Marketplace Management Portal**
   - Visit: https://aws.amazon.com/marketplace/management/
   - Sign in

2. **Navigate to Assets**
   - Click "Assets" in left menu
   - Click "AMI" tab

3. **Scan AMI**
   - Click "Scan AMI"
   - Enter your AMI ID
   - Select region: us-east-1
   - Click "Scan"

4. **Wait for Results**
   - Scan takes 30-60 minutes
   - You'll receive email when complete
   - Check for vulnerabilities

5. **Fix Any Issues**
   - If vulnerabilities found, update your AMI
   - Common issues:
     - Outdated packages (run yum update)
     - Open ports (check security groups)
     - Missing security patches
   - Rescan after fixes

**✓ AMI must pass scan before proceeding**

---

### Step 5.2: Create Product Listing

1. **Start New Product**
   - In Marketplace Management Portal
   - Click "Products" → "Server"
   - Click "Create new server product"

2. **Choose Product Type**
   - Select "AMI with CloudFormation"
   - Click "Next"

3. **Product Information**
   - **Product title:** Security Group Monitor
   - **SKU:** security-group-monitor-v1 (unique identifier)
   - **Short description:** (from Step 4.2)
   - **Long description:** (from Step 4.2)
   - **Product logo:** Upload 110x110 PNG (your logo)
   - **Highlights:** (from Step 4.2)
   - **Categories:** 
     - Primary: Security
     - Secondary: Monitoring & Logging
   - **Keywords:** security groups, compliance, monitoring, remediation, aws
   - Click "Next"

4. **Pricing**
   - **Pricing model:** Choose one:
     - Hourly: $0.05/hour ($36/month)
     - Monthly: $29/month
     - Annual: $299/year
   - **Free trial:** Optional (7 or 30 days)
   - **Refund policy:** Standard AWS Marketplace policy
   - Click "Next"

5. **AMI Details**
   - **AMI ID:** Your AMI ID from Step 2.3
   - **Region:** us-east-1
   - **OS:** Amazon Linux 2
   - **Instance types:** 
     - Recommended: t3.nano
     - Supported: t3.nano, t3.micro (add more if needed)
   - **Root device:** EBS
   - **Virtualization:** HVM
   - Click "Next"

6. **CloudFormation Template**
   - **Upload template:** Your updated template file
   - **Template name:** security-group-monitor-v1.0.0
   - **Description:** CloudFormation template for Security Group Monitor
   - **Architecture diagram:** Upload PNG (from Step 4.1)
   - **Pricing estimate URL:** From AWS Pricing Calculator (Step 4.4)
   - Click "Next"

7. **Support Information**
   - **Support description:** Email support with 24-48 hour response time
   - **Support email:** your-support-email@example.com
   - **Support URL:** https://your-docs-site.com
   - **Refund policy:** Standard
   - Click "Next"

8. **Usage Instructions**
   - **Upload PDF:** Your deployment guide (from Step 4.3)
   - Or paste instructions directly
   - Click "Next"

9. **Review and Submit**
   - Review all information
   - Check for errors
   - Click "Submit for review"

**✓ Submission complete! Now wait for AWS review**

---

### Step 5.3: AWS Review Process

**What happens next:**

1. **Initial Review (1-3 days)**
   - AWS checks for completeness
   - Validates AMI scan passed
   - Reviews template syntax
   - Checks pricing model

2. **Technical Review (3-7 days)**
   - AWS tests your CloudFormation template
   - Verifies deployment works
   - Checks security best practices
   - Tests in multiple regions

3. **Possible Outcomes:**
   - **Approved:** Product goes live!
   - **Needs changes:** You'll receive email with required fixes
   - **Rejected:** Rare, usually fixable issues

4. **If Changes Needed:**
   - Read feedback email carefully
   - Make required changes
   - Resubmit
   - Review process restarts

**Common rejection reasons:**
- AMI vulnerabilities not fixed
- CloudFormation template errors
- Missing or incorrect documentation
- Pricing issues
- Security group rules too permissive

---

## Phase 6: Post-Launch (Week 4+)

### Step 6.1: Product Goes Live

Once approved:

1. **Product Page Created**
   - Your product appears on AWS Marketplace
   - URL: https://aws.amazon.com/marketplace/pp/prodview-XXXXX
   - Share this link!

2. **Test Customer Flow**
   - Subscribe to your own product (free)
   - Deploy via CloudFormation
   - Verify everything works
   - Unsubscribe

3. **Monitor Sales**
   - Check Marketplace Management Portal
   - View subscriber count
   - Track revenue
   - Review customer feedback

### Step 6.2: Marketing Your Product

**Free marketing:**
- Share on LinkedIn, Twitter
- Post on Reddit (r/aws, r/devops)
- Write blog post about the problem it solves
- Create YouTube demo video
- Submit to AWS Partner Network blog

**Paid marketing:**
- AWS Marketplace ads
- Google Ads for "AWS security monitoring"
- Sponsor AWS community events

### Step 6.3: Customer Support

**Set up support channels:**
- Email: Create dedicated support email
- Documentation: Host on GitHub Pages or your site
- FAQ: Common questions and answers
- Slack/Discord: Optional community

**Response times:**
- Critical issues: 4-8 hours
- General questions: 24-48 hours
- Feature requests: Log and prioritize

### Step 6.4: Updates and Maintenance

**Updating your product:**

1. **Create new AMI version**
   - Follow Step 2 again
   - Increment version (v1.0.1, v1.1.0, etc.)
   - Test thoroughly

2. **Update CloudFormation template**
   - Make improvements
   - Test in multiple regions
   - Update version number

3. **Submit update**
   - In Marketplace Management Portal
   - Go to your product
   - Click "Create new version"
   - Upload new AMI and template
   - Submit for review

4. **Notify customers**
   - AWS sends update notification
   - Customers can upgrade at their convenience

**Recommended update schedule:**
- Security patches: Immediately
- Bug fixes: As needed
- New features: Quarterly
- AMI base updates: Every 6 months

---

## Pricing Recommendations

Based on similar products on AWS Marketplace:

**Option 1: Hourly**
- $0.05/hour = ~$36/month
- Good for: Testing, small deployments
- Customer pays: ~$43/month total (including AWS costs)

**Option 2: Monthly**
- $29/month flat fee
- Good for: Predictable pricing
- Customer pays: ~$36/month total

**Option 3: Annual**
- $299/year (~$25/month)
- 30% discount vs monthly
- Good for: Committed customers
- Customer pays: ~$32/month total

**Option 4: Tiered**
- Free: Up to 10 security groups
- Pro: $49/month - Up to 100 security groups
- Enterprise: $199/month - Unlimited + priority support

**My recommendation:** Start with $29/month flat fee. Simple, predictable, competitive.

---

## Cost Summary

**Your costs:**
- Seller registration: $0
- AMI creation: ~$1 (EC2 time)
- Testing: ~$5-10
- Time investment: 20-40 hours

**Customer costs per month:**
- Your license: $29 (example)
- AWS infrastructure: ~$7-10
- Total: ~$36-39/month

**Your revenue:**
- AWS Marketplace fee: 3%
- Your net: $28.13 per customer/month
- 10 customers = $281/month
- 100 customers = $2,813/month
- 1000 customers = $28,130/month

---

## Troubleshooting

### AMI Scan Fails
- Update all packages: `sudo yum update -y`
- Remove unnecessary software
- Check for open ports
- Rescan

### CloudFormation Template Errors
- Validate syntax: Use cfn-lint
- Test in multiple regions
- Check IAM permissions
- Review AWS documentation

### Product Rejected
- Read feedback email carefully
- Fix all issues mentioned
- Test thoroughly
- Resubmit with explanation

### No Sales
- Improve product description
- Add more screenshots
- Lower price temporarily
- Increase marketing efforts
- Ask for reviews

---

## Next Steps

1. ✅ Register as seller (Week 1)
2. ✅ Create AMI (Week 1-2)
3. ✅ Update CloudFormation template (Week 2)
4. ✅ Prepare materials (Week 2)
5. ✅ Submit to Marketplace (Week 3)
6. ⏳ Wait for approval (Week 3-4)
7. 🚀 Launch and market (Week 4+)

---

## Resources

**AWS Documentation:**
- [Marketplace Seller Guide](https://docs.aws.amazon.com/marketplace/latest/userguide/)
- [CloudFormation Best Practices](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/best-practices.html)
- [AMI Creation Guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/creating-an-ami-ebs.html)

**Tools:**
- [AWS Pricing Calculator](https://calculator.aws/)
- [cfn-lint](https://github.com/aws-cloudformation/cfn-lint) - Template validator
- [TaskCat](https://github.com/aws-quickstart/taskcat) - Multi-region testing

**Community:**
- [AWS Marketplace Sellers Forum](https://forums.aws.amazon.com/forum.jspa?forumID=153)
- [r/aws on Reddit](https://reddit.com/r/aws)
- [AWS Marketplace Slack](https://aws-marketplace-community.slack.com/)

---

## Questions?

Feel free to ask if you need clarification on any step!

Good luck with your AWS Marketplace launch! 🚀
