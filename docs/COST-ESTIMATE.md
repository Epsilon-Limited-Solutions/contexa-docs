# Cost Estimate - Medical Knowledge Graph MVP

**Last Updated**: January 2025
**Region**: us-east-1
**Scale**: 1 paper/day (~30 papers/month)

---

## Monthly Cost Breakdown

### 1. Compute Services

| Service | Configuration | Usage | Unit Cost | Monthly Cost |
|---------|--------------|-------|-----------|--------------|
| **Lambda - Ingestion** | 256 MB, 30s timeout | 30 invocations | Free tier | $0.00 |
| **Lambda - Orchestrator** | 512 MB, 1 min timeout | 30 invocations | Free tier | $0.00 |
| **Lambda - BioBERT Container** | 3 GB, 5 min timeout | 30 invocations × 1 min avg | $0.0000166667/GB-second | $0.05 |
| **Lambda - Entity Linker** | 512 MB, 30s timeout | 30 invocations | Free tier | $0.00 |
| **Lambda - Validation API** | 256 MB, 10s timeout | ~300 invocations/month | Free tier | $0.00 |
| **Lambda - TTL Generator** | 1 GB, 2 min timeout | 30 invocations | Free tier | $0.00 |
| **ECR - Container Storage** | 10 GB image storage | 10 GB | $0.10/GB | $1.00 |
| | | | **Compute Total** | **$1.05** |

**Lambda Free Tier**: 1M requests/month + 400,000 GB-seconds/month
**Usage**: ~300 requests/month, ~200 GB-seconds/month (well within free tier)

---

### 2. Database Services

| Service | Configuration | Monthly Cost |
|---------|--------------|--------------|
| **RDS PostgreSQL** | db.t4g.micro (2 vCPU, 1 GB) | $12.41 |
| | Deployment: Single-AZ | |
| | Storage Type: GP3 | |
| **RDS Storage** | 20 GB GP3 | $2.30 |
| | ($0.115/GB-month) | |
| **Backup Storage** | 20 GB (automated backups) | $2.00 |
| | ($0.10/GB-month) | |
| | | |
| | **Database Total** | **$16.71** |

**Cost Optimization Notes**:
- Single-AZ saves ~50% vs Multi-AZ ($24.82)
- GP3 is 20% cheaper than GP2
- Can use Reserved Instance for 40% discount after 6 months

---

### 3. Storage Services

| Service | Bucket/Usage | Size | Unit Cost | Monthly Cost |
|---------|--------------|------|-----------|--------------|
| **S3 - Papers** | Raw papers storage | 5 GB | $0.023/GB | $0.12 |
| **S3 - Static UI** | React app (minified) | 50 MB | $0.023/GB | $0.00 |
| **S3 - TTL Exports** | Knowledge graph exports | 100 MB | $0.023/GB | $0.00 |
| **S3 - Requests** | PUT/GET operations | ~1000/month | Various | $0.01 |
| | | | **Storage Total** | **$0.13** |

**S3 Free Tier**: 5 GB storage, 20K GET, 2K PUT requests/month (first 12 months)

---

### 4. Networking Services

| Service | Configuration | Monthly Cost |
|---------|--------------|--------------|
| **VPC Endpoint - S3** | Gateway type | $0.00 |
| **VPC Endpoint - ECR API** | Interface type | $7.30 |
| **VPC Endpoint - ECR DKR** | Interface type | $7.30 |
| **VPC Endpoint - SQS** | Interface type | $7.30 |
| **VPC Endpoint - Data** | ~1 GB/month | $0.01 |
| **Data Transfer Out** | <1 GB/month | Free tier | $0.00 |
| | | **Networking Total** | **$21.91** |

**Alternative**: NAT Gateway = $32.40/month + $0.045/GB data = ~$35/month
**Savings**: $13/month using VPC Endpoints

---

### 5. API & Application Services

| Service | Usage | Monthly Cost |
|---------|-------|--------------|
| **API Gateway** | REST API, 350 requests/month | Free tier | $0.00 |
| **CloudFront** | CDN for static UI | 100 MB data transfer | Free tier | $0.00 |
| **Cognito** | 1-5 active users | Free tier (50K MAU) | $0.00 |
| **SQS** | Standard queue, ~1000 messages | Free tier | $0.00 |
| | | **API/App Total** | **$0.00** |

**Free Tiers**:
- API Gateway: 1M requests/month (12 months)
- CloudFront: 1 TB data transfer out (12 months)
- Cognito: First 50K MAU always free
- SQS: 1M requests/month always free

---

### 6. Monitoring & Management

| Service | Usage | Monthly Cost |
|---------|-------|--------------|
| **CloudWatch Logs** | 5 GB ingestion | $2.50 |
| | 5 GB storage | $0.03 |
| **CloudWatch Metrics** | 10 custom metrics | Free tier | $0.00 |
| **CloudWatch Dashboards** | 1 dashboard | Free tier | $0.00 |
| **CloudWatch Alarms** | 3 alarms | Free tier | $0.00 |
| **EventBridge** | 10 rules | Free tier | $0.00 |
| **Systems Manager** | Parameter Store (free tier) | $0.00 |
| | | **Monitoring Total** | **$2.53** |

---

### 7. Optional Services

| Service | Purpose | Monthly Cost |
|---------|---------|--------------|
| **Route53** | Custom domain (optional) | $0.50 |
| **ACM Certificate** | HTTPS certificate | $0.00 |
| **Secrets Manager** | RDS credentials (can use IAM) | $0.40 |
| | | **Optional Total** | **$0.90** |

---

## Total Monthly Cost Summary

### Baseline Configuration (Minimal)

```
Compute:              $  1.05
Database:             $ 16.71
Storage:              $  0.13
Networking:           $ 21.91
API/Application:      $  0.00
Monitoring:           $  2.53
Optional:             $  0.00
─────────────────────────────
TOTAL:                $ 42.33/month
```

### With Optional Services

```
Baseline:             $ 42.33
Optional (Route53):   $  0.90
─────────────────────────────
TOTAL:                $ 43.23/month
```

### After Free Tier Expires (12 months)

Some services have 12-month free tiers that will expire:

| Service | Current | After 12 months |
|---------|---------|-----------------|
| Lambda | $0.00 (free tier) | ~$0.50 |
| API Gateway | $0.00 | ~$1.20 |
| CloudFront | $0.00 | ~$1.00 |
| S3 | $0.13 (partial free) | ~$0.50 |
| **Impact** | **$0.00** | **+$3.20/month** |

**Year 2+ Cost**: ~$46/month

---

## Cost per Paper Processed

```
Variable Costs per Paper:
- Lambda execution:     $0.0015
- S3 storage (5 MB):    $0.0001
- S3 requests:          $0.0001
- SQS messages:         $0.0000
- Data transfer:        $0.0001
──────────────────────────────
Total per paper:        $0.0018

For 30 papers/month:    $0.05
```

**Marginal cost is negligible** - you could process 10x more papers with minimal cost increase.

---

## Scaling Cost Projections

### Scenario: 10 papers/day (300/month)

| Component | Current (30/mo) | @ 300/mo | Increase |
|-----------|-----------------|----------|----------|
| Lambda | $0.05 | $0.50 | +$0.45 |
| RDS | $16.71 | $16.71 | $0.00 |
| Storage | $0.13 | $0.50 | +$0.37 |
| Other | $25.44 | $25.44 | $0.00 |
| **Total** | **$42.33** | **$43.15** | **+$0.82** |

**Key Insight**: 10x increase in papers = 2% cost increase (highly scalable)

---

### Scenario: 100 papers/day (3,000/month)

| Component | Current | @ 3K/mo | Notes |
|-----------|---------|---------|-------|
| Lambda | $0.05 | $5.00 | Outside free tier |
| RDS | $16.71 | $50.00 | Upgrade to db.t4g.small |
| Storage | $0.13 | $2.00 | ~200 GB stored |
| Networking | $21.91 | $25.00 | More data transfer |
| Other | $3.53 | $5.00 | More monitoring |
| **Total** | **$42.33** | **$87.00** | 2x cost for 100x papers |

---

### Scenario: 1,000 papers/day (30,000/month) - Production Scale

At this scale, architecture changes needed:

| Component | Configuration | Monthly Cost |
|-----------|--------------|--------------|
| **SageMaker** | ml.g4dn.xlarge (GPU) | $250-400 |
| | On-demand or Spot | |
| **Neptune** | db.r5.large | $350 |
| | Replaces RDS + AGE | |
| **ElastiCache** | cache.t4g.small | $15 |
| **RDS** | db.r6g.large (metadata) | $150 |
| **Lambda** | All functions | $50 |
| **Storage** | ~1 TB S3 | $25 |
| **Networking** | NAT + increased transfer | $100 |
| **Monitoring** | Enhanced | $50 |
| **Total** | | **~$990-1,140/month** |

**Cost per paper at scale**: $0.033-0.038 each

---

## Cost Optimization Strategies

### Immediate (MVP)

1. **Use VPC Endpoints instead of NAT**: Save $13/month ✓
2. **Single-AZ RDS**: Save $12/month ✓
3. **Leverage free tiers**: Save ~$10/month ✓
4. **Use Lambda over EC2**: Save ~$50/month ✓

**Current savings**: ~$85/month vs naive approach

---

### After 6 Months

1. **RDS Reserved Instance** (1-year commitment):
   - Current: $12.41/month on-demand
   - Reserved: $7.45/month
   - **Savings**: $5/month ($60/year)

2. **Compute Savings Plan** (1-year commitment):
   - Save 20% on Lambda compute
   - **Savings**: ~$1/month (small at MVP scale)

3. **S3 Intelligent-Tiering**:
   - Auto-move old papers to cheaper tiers
   - **Savings**: ~$0.50/month

**Total additional savings**: ~$6.50/month (15% reduction)

---

### At Scale (1,000+ papers/day)

1. **SageMaker Savings Plan**: 40-60% off
2. **Neptune Reserved Instances**: 40% off
3. **S3 Glacier** for old papers: 80% storage savings
4. **EC2 Spot for batch processing**: 70% savings

**Potential savings at scale**: $300-500/month

---

## Hidden Costs & Considerations

### One-Time Costs

| Item | Estimated Cost | Notes |
|------|---------------|--------|
| Development time | $0 (your time) | ~40-80 hours |
| Pre-trained model download | $0 | Free from HuggingFace |
| Initial testing | <$5 | Minimal AWS usage |
| Domain registration | $12/year | Optional |

---

### Potential Future Costs

| Item | When Needed | Est. Cost |
|------|-------------|-----------|
| GPU for model training | Phase 2 (retraining) | $50-200/month |
| Data labeling service | If outsourcing validation | $100-500/month |
| Third-party APIs | Entity linking at scale | $0-50/month |
| Compliance/Security audit | Production | $1,000-5,000 one-time |

---

## Cost Comparison: Alternative Approaches

### Option A: Current Serverless (Recommended)
- **Monthly**: $42/month
- **Pros**: Low cost, auto-scaling, minimal ops
- **Cons**: Lambda cold starts, 15-min limit

### Option B: EC2-Based
- **Monthly**: ~$120/month
  - t3.medium: $30/month
  - RDS db.t4g.small: $25/month
  - Load Balancer: $16/month
  - NAT Gateway: $32/month
  - Other: $17/month
- **Pros**: More control, no cold starts
- **Cons**: 3x more expensive, always-on costs

### Option C: Managed Services (Fully Managed)
- **Monthly**: ~$500/month
  - SageMaker Endpoint: $200/month
  - Neptune: $350/month
  - Other: $50/month
- **Pros**: Production-ready, highly scalable
- **Cons**: 12x more expensive, overkill for MVP

---

## Break-Even Analysis

**Serverless MVP vs EC2**:
- Serverless: $42/month
- EC2: $120/month
- Difference: $78/month

**Serverless MVP vs Managed**:
- Serverless: $42/month
- Managed: $500/month
- Difference: $458/month

**Conclusion**: Serverless is optimal until >100 papers/day

---

## Annual Budget Projection

### Year 1

| Quarter | Papers Processed | Est. Monthly Avg | Quarterly Cost |
|---------|------------------|------------------|----------------|
| Q1 | 90 (1/day) | $42 | $126 |
| Q2 | 270 (3/day) | $43 | $129 |
| Q3 | 450 (5/day) | $44 | $132 |
| Q4 | 900 (10/day) | $46 | $138 |
| **Total** | **1,710 papers** | **$44 avg** | **$525** |

**Total Year 1**: ~$525 (~$44/month average)

---

### Year 2 (Assuming Growth)

| Quarter | Papers/Day | Monthly Cost |
|---------|-----------|--------------|
| Q1 | 20 | $48 |
| Q2 | 40 | $55 |
| Q3 | 60 | $65 |
| Q4 | 100 | $90 |
| **Avg** | **55/day** | **$65/month** |

**Total Year 2**: ~$780 (~$65/month average)

---

## ROI Considerations

### Cost per Knowledge Triple

Assuming each paper yields 10 validated relationships:

| Papers/Month | Triples/Month | Monthly Cost | Cost per Triple |
|--------------|---------------|--------------|-----------------|
| 30 | 300 | $42 | $0.14 |
| 300 | 3,000 | $45 | $0.015 |
| 3,000 | 30,000 | $90 | $0.003 |

**Insight**: Cost efficiency improves dramatically with scale

---

### Comparison: Manual Curation

**Human curator** extracting knowledge:
- Rate: ~5 papers/day (8 hours)
- Cost: $50/hour × 8 = $400/day
- Monthly: $8,000 (for 100 papers)

**AI-assisted with validation**:
- Processing: Automated ($42/month)
- Validation: 1 hour/paper × 30 papers = 30 hours/month
- Cost: $50/hour × 30 = $1,500
- **Total**: $1,542/month

**Savings vs Pure Manual**: 81% cost reduction

---

## Budget Recommendations

### MVP Phase (Months 1-6)
- **Budget**: $50/month
- **Buffer**: 20% for unexpected costs
- **Total**: $300 for 6 months

### Growth Phase (Months 7-12)
- **Budget**: $75/month
- **Include**: Model retraining experiments
- **Total**: $450 for 6 months

### Year 1 Total Recommended Budget
- **$750** (includes 15% buffer)

---

## Cost Alerts Setup

Recommended CloudWatch Billing Alarms:

1. **Warning**: $30/month threshold
2. **Alert**: $50/month threshold
3. **Critical**: $75/month threshold

---

## Summary

**Bottom Line**:
- **MVP Start**: $42-45/month
- **Year 1 Average**: ~$44/month
- **Scalability**: 10x papers = <5% cost increase
- **Break-even vs EC2**: Always cheaper until 100+ papers/day
- **Production scale**: ~$1,000/month at 1K papers/day

**Recommendation**: Start with serverless architecture, monitor usage, optimize with Reserved Instances after 6 months if sustained usage is confirmed.
