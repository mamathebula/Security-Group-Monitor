# Cost Breakdown - Security Group Monitor

Complete cost analysis for running the Security Group Monitor on AWS.

---

## Monthly Cost Summary

### Current Configuration (t3.medium)

| Service | Resource | Monthly Cost | Notes |
|---------|----------|--------------|-------|
| **EC2** | t3.medium instance (License Server) | $30.37 | 730 hours/month × $0.0416/hour |
| **EC2** | EBS Volume (8 GB gp3) | $0.64 | 8 GB × $0.08/GB/month |
| **S3** | CloudTrail logs storage | $0.50 | ~20 GB × $0.023/GB (first month) |
| **S3** | PUT requests (CloudTrail) | $0.10 | ~20,000 requests × $0.005/1000 |
| **Lambda** | SecurityGroupMonitorFunction | $0.00 | Free tier covers this |
| **Lambda** | InitialScanFunction | $0.00 | Runs once, free tier |
| **CloudTrail** | Management events | $0.00 | First trail is free |
| **SNS** | Email notifications | $0.00 | First 1,000 emails free |
| **EventBridge** | Rules | $0.00 | Free for AWS service events |
| **CloudWatch Logs** | Log storage (7 days retention) | $0.10 | ~2 GB × $0.50/GB/month |
| **Data Transfer** | Outbound data | $0.00 | Minimal, within free tier |
| **TOTAL** | | **$31.71/month** | **$380.52/year** |

---

## Optimized Configuration (t3.nano)

### Recommended for Cost Savings

| Service | Resource | Monthly Cost | Notes |
|---------|----------|--------------|-------|
| **EC2** | t3.nano instance (License Server) | $3.80 | 730 hours/month × $0.0052/hour |
| **EC2** | EBS Volume (8 GB gp3) | $0.64 | 8 GB × $0.08/GB/month |
| **S3** | CloudTrail logs storage | $0.50 | ~20 GB × $0.023/GB |
| **S3** | PUT requests (CloudTrail) | $0.10 | ~20,000 requests × $0.005/1000 |
| **Lambda** | SecurityGroupMonitorFunction | $0.00 | Free tier covers this |
| **Lambda** | InitialScanFunction | $0.00 | Runs once, free tier |
| **CloudTrail** | Management events | $0.00 | First trail is free |
| **SNS** | Email notifications | $0.00 | First 1,000 emails free |
| **EventBridge** | Rules | $0.00 | Free for AWS service events |
| **CloudWatch Logs** | Log storage (7 days retention) | $0.10 | ~2 GB × $0.50/GB/month |
| **Data Transfer** | Outbound data | $0.00 | Minimal, within free tier |
| **TOTAL** | | **$5.14/month** | **$61.68/year** |

**Savings: $26.57/month ($318.84/year) compared to t3.medium**

---

## Cost Breakdown by Service

### 1. EC2 Instance (License Server)

The EC2 instance is required by AWS Marketplace for billing/licensing.

| Instance Type | vCPU | RAM | Hourly Rate | Monthly Cost | Annual Cost |
|---------------|------|-----|-------------|--------------|-------------|
| t3.nano | 2 | 0.5 GB | $0.0052 | $3.80 | $45.60 |
| t3.micro | 2 | 1 GB | $0.0104 | $7.59 | $91.08 |
| t3.small | 2 | 2 GB | $0.0208 | $15.18 | $182.16 |
| t3.medium | 2 | 4 GB | $0.0416 | $30.37 | $364.44 |
| t3.large | 2 | 8 GB | $0.0832 | $60.74 | $728.88 |

**Recommendation:** Use t3.nano ($3.80/month) - the license server does minimal work

**Note:** Prices are for US East (N. Virginia) region. Other regions may vary by ±10%.

### 2. EBS Storage

| Volume Type | Size | Monthly Cost | Notes |
|-------------|------|--------------|-------|
| gp3 (default) | 8 GB | $0.64 | $0.08/GB/month |
| gp3 | 20 GB | $1.60 | If you need more space |
| gp2 | 8 GB | $0.80 | Older generation, not recommended |

**Recommendation:** Keep default 8 GB gp3 ($0.64/month)

### 3. S3 Storage (CloudTrail Logs)

| Month | Storage Size | Monthly Cost | Notes |
|-------|--------------|--------------|-------|
| Month 1 | ~20 GB | $0.50 | Initial accumulation |
| Month 2+ | ~20 GB | $0.50 | Stable (7-day retention with lifecycle) |

**Cost Formula:**
- Storage: 20 GB × $0.023/GB = $0.46
- PUT requests: 20,000 × $0.005/1000 = $0.10
- GET requests: Minimal, ~$0.01
- **Total: ~$0.50/month**

**Note:** With 7-day lifecycle policy, storage stays constant at ~20 GB

### 4. Lambda Functions

| Function | Invocations/Month | Duration | Memory | Monthly Cost |
|----------|-------------------|----------|--------|--------------|
| SecurityGroupMonitorFunction | ~1,000 | 2 seconds | 128 MB | $0.00 |
| InitialScanFunction | 1 | 30 seconds | 128 MB | $0.00 |
| TriggerInitialScanFunction | 1 | 1 second | 128 MB | $0.00 |

**Free Tier Coverage:**
- 1 million requests/month (free)
- 400,000 GB-seconds compute time (free)

**Your Usage:**
- Requests: ~1,000/month (well within free tier)
- Compute: ~0.07 GB-seconds (well within free tier)

**Cost: $0.00/month** (covered by free tier)

**Note:** Even without free tier, cost would be ~$0.20/month

### 5. CloudTrail

| Component | Monthly Cost | Notes |
|-----------|--------------|-------|
| First trail | $0.00 | Free |
| Management events | $0.00 | Included in first trail |
| Data events | N/A | Not used |
| Additional trails | $2.00/trail | Not needed |

**Cost: $0.00/month**

### 6. SNS (Email Notifications)

| Component | Usage | Monthly Cost | Notes |
|-----------|-------|--------------|-------|
| Email notifications | ~50 emails | $0.00 | First 1,000 free |
| SMS notifications | N/A | N/A | Not used |

**Free Tier:**
- 1,000 email notifications/month (free)
- 100 SMS notifications/month (free)

**Cost: $0.00/month** (covered by free tier)

**Note:** Even with 10,000 emails/month, cost would be ~$2/month

### 7. EventBridge (CloudWatch Events)

| Component | Monthly Cost | Notes |
|-----------|--------------|-------|
| Custom event bus | $0.00 | Using default bus |
| Events from AWS services | $0.00 | Free |
| Custom events | N/A | Not used |

**Cost: $0.00/month**

### 8. CloudWatch Logs

| Component | Storage | Monthly Cost | Notes |
|-----------|---------|--------------|-------|
| Log ingestion | ~2 GB | $0.00 | First 5 GB free |
| Log storage (7 days) | ~2 GB | $0.10 | $0.50/GB/month prorated |
| Log queries | Minimal | $0.00 | $0.005/GB scanned |

**Cost: ~$0.10/month**

**Note:** With 7-day retention, storage is minimal

### 9. Data Transfer

| Type | Monthly Usage | Cost | Notes |
|------|---------------|------|-------|
| Inbound | Unlimited | $0.00 | Always free |
| Outbound (to internet) | ~100 MB | $0.00 | First 100 GB free |
| Inter-region | None | $0.00 | Single region deployment |

**Cost: $0.00/month**

---

## Cost Comparison: Different Scenarios

### Scenario 1: Minimal Deployment (Recommended)
- Instance: t3.nano
- Auto-remediate: Enabled
- Initial scan: Enabled
- **Cost: $5.14/month ($61.68/year)**

### Scenario 2: Standard Deployment (Current Template Default)
- Instance: t3.medium
- Auto-remediate: Enabled
- Initial scan: Enabled
- **Cost: $31.71/month ($380.52/year)**

### Scenario 3: High-Activity Environment
- Instance: t3.nano
- Auto-remediate: Enabled
- Initial scan: Enabled
- 1,000+ security group changes/day
- **Cost: $5.50/month ($66/year)**
  - Lambda costs increase to ~$0.40/month
  - CloudWatch Logs increase to ~$0.20/month

### Scenario 4: Multi-Region Deployment (3 regions)
- Instance: t3.nano per region
- Auto-remediate: Enabled
- Initial scan: Enabled
- **Cost: $15.42/month ($185.04/year)**
  - 3× single region cost

---

## Cost Optimization Tips

### 1. Use t3.nano Instead of t3.medium
**Savings: $26.57/month ($318.84/year)**

The license server does almost nothing - it just validates the Marketplace subscription. t3.nano is more than sufficient.

**How to change:**
- Edit CloudFormation template
- Change `InstanceType: t3.medium` to `InstanceType: t3.nano`
- Update stack

### 2. Reduce Log Retention
**Savings: $0.05-0.10/month**

Current: 7 days (recommended)
Alternative: 3 days (saves ~$0.05/month)

**Not recommended** - savings are minimal

### 3. Disable Initial Scan (if not needed)
**Savings: $0.00/month**

Initial scan runs once and is covered by free tier. No savings.

### 4. Use Reserved Instances (1-year commitment)
**Savings: $1.14/month ($13.68/year) for t3.nano**

| Instance | On-Demand | 1-Year Reserved | Savings |
|----------|-----------|-----------------|---------|
| t3.nano | $3.80/month | $2.66/month | 30% |
| t3.medium | $30.37/month | $21.26/month | 30% |

**Recommendation:** Only if you're committed to running this for 1+ year

### 5. Use Savings Plans
**Savings: Similar to Reserved Instances**

More flexible than Reserved Instances but requires AWS account-wide commitment.

---

## Pricing by AWS Region

Prices vary by region. Here's a comparison for t3.nano:

| Region | Region Code | t3.nano/month | Notes |
|--------|-------------|---------------|-------|
| US East (N. Virginia) | us-east-1 | $3.80 | Cheapest US region |
| US East (Ohio) | us-east-2 | $3.80 | Same as us-east-1 |
| US West (Oregon) | us-west-2 | $3.80 | Same as us-east-1 |
| US West (N. California) | us-west-1 | $4.18 | 10% more expensive |
| EU (Ireland) | eu-west-1 | $4.18 | 10% more expensive |
| EU (Frankfurt) | eu-central-1 | $4.56 | 20% more expensive |
| Asia Pacific (Singapore) | ap-southeast-1 | $4.56 | 20% more expensive |
| Asia Pacific (Sydney) | ap-southeast-2 | $4.94 | 30% more expensive |
| Asia Pacific (Tokyo) | ap-northeast-1 | $4.94 | 30% more expensive |

**Recommendation:** Deploy in us-east-1 for lowest cost

---

## Customer Pricing Strategy

### What to Charge Customers

If selling on AWS Marketplace, you need to cover your costs plus profit:

| Your Cost | Customer Price | Your Profit | Margin |
|-----------|----------------|-------------|--------|
| $5.14/month | $10/month | $4.86/month | 49% |
| $5.14/month | $15/month | $9.86/month | 66% |
| $5.14/month | $20/month | $14.86/month | 74% |
| $5.14/month | $30/month | $24.86/month | 83% |
| $5.14/month | $50/month | $44.86/month | 90% |

**AWS Marketplace Fee:** 3% of customer payment

**Example with $20/month customer price:**
- Customer pays: $20/month
- AWS takes: $0.60/month (3%)
- You receive: $19.40/month
- Your cost: $5.14/month
- Your profit: $14.26/month (73% margin)

**Recommendation:** Charge $15-30/month for current version (security groups only)

---

## Cost Projections

### Year 1 Projections (t3.nano)

| Scenario | Customers | Monthly Revenue | Monthly Costs | Monthly Profit | Annual Profit |
|----------|-----------|-----------------|---------------|----------------|---------------|
| Conservative | 10 | $200 | $51.40 | $148.60 | $1,783 |
| Moderate | 25 | $500 | $128.50 | $371.50 | $4,458 |
| Optimistic | 50 | $1,000 | $257.00 | $743.00 | $8,916 |
| Aggressive | 100 | $2,000 | $514.00 | $1,486.00 | $17,832 |

**Assumptions:**
- Customer price: $20/month
- Your cost per customer: $5.14/month
- AWS Marketplace fee: 3% (already deducted from revenue)

### Year 1 Projections (t3.medium - Not Recommended)

| Scenario | Customers | Monthly Revenue | Monthly Costs | Monthly Profit | Annual Profit |
|----------|-----------|-----------------|---------------|----------------|---------------|
| Conservative | 10 | $200 | $317.10 | -$117.10 | -$1,405 |
| Moderate | 25 | $500 | $792.75 | -$292.75 | -$3,513 |
| Optimistic | 50 | $1,000 | $1,585.50 | -$585.50 | -$7,026 |
| Aggressive | 100 | $2,000 | $3,171.00 | -$1,171.00 | -$14,052 |

**Result:** You lose money with t3.medium at $20/month pricing!

**To break even with t3.medium, you'd need to charge $35-40/month**

---

## Free Tier Impact

### First 12 Months (New AWS Account)

If your customers are using new AWS accounts, they get additional free tier benefits:

| Service | Free Tier | Value |
|---------|-----------|-------|
| EC2 | 750 hours/month t2.micro or t3.micro | $7.59/month |
| S3 | 5 GB storage | $0.12/month |
| Lambda | 1M requests + 400K GB-seconds | $0.20/month |
| CloudWatch Logs | 5 GB ingestion | $0.50/month |

**Total Free Tier Value: ~$8.41/month**

**Note:** Your template uses t3.medium/t3.nano (not t2.micro/t3.micro), so EC2 free tier doesn't apply directly. But customers could modify the template.

---

## Cost Monitoring

### How to Track Your Costs

1. **AWS Cost Explorer:**
   - Go to: https://console.aws.amazon.com/cost-management/home
   - Filter by tag: `Name=do-not-delete`
   - View daily/monthly costs

2. **AWS Budgets:**
   - Set budget alert at $10/month
   - Get notified if costs exceed threshold

3. **CloudWatch Billing Alarms:**
   - Create alarm for estimated charges
   - Get email when threshold is reached

### Cost Allocation Tags

All resources are tagged with:
- `Name=do-not-delete`
- `ManagedBy=CloudFormation`
- `Purpose=SecurityGroupMonitoring`

Use these tags in Cost Explorer to track spending.

---

## Summary

### Current Configuration (t3.medium)
- **Monthly Cost:** $31.71
- **Annual Cost:** $380.52
- **Recommendation:** Switch to t3.nano

### Optimized Configuration (t3.nano)
- **Monthly Cost:** $5.14
- **Annual Cost:** $61.68
- **Recommendation:** Use this configuration

### Cost Breakdown (t3.nano)
- EC2 instance: $3.80 (74%)
- EBS volume: $0.64 (12%)
- S3 storage: $0.50 (10%)
- CloudWatch Logs: $0.10 (2%)
- Everything else: $0.00 (0%)

### Customer Pricing Recommendation
- **Charge:** $15-30/month
- **Your cost:** $5.14/month
- **Your profit:** $9.86-24.86/month per customer
- **Margin:** 66-83%

### Break-Even Analysis
- **Minimum customers needed:** 1 customer at $10/month
- **Comfortable profit:** 10+ customers at $20/month = $148/month profit
- **Good business:** 50+ customers at $20/month = $743/month profit

---

## Cost Comparison with Alternatives

### vs. AWS Config
- **AWS Config:** $2/rule/region/month = $2/month for 1 rule
- **Your solution:** $5.14/month but includes auto-remediation
- **Verdict:** Competitive if you add more checks

### vs. Third-Party Tools
- **Prowler:** Free (open source) but no auto-remediation
- **CloudSploit:** Free (open source) but no auto-remediation
- **Dome9/CloudGuard:** $10-50/month per account
- **Prisma Cloud:** $500+/month enterprise
- **Your solution:** $5.14/month with auto-remediation
- **Verdict:** Good value proposition

---

## Questions?

For cost optimization help or pricing strategy advice, refer to:
- AWS Pricing Calculator: https://calculator.aws
- AWS Cost Explorer: https://console.aws.amazon.com/cost-management/home
- AWS Marketplace Seller Guide: https://docs.aws.amazon.com/marketplace/
