# Medical Knowledge Graph Architecture

## Overview
This document describes the AWS architecture for a medical knowledge extraction and graph construction system with continuous learning capabilities.

**Scale**: MVP (1 paper/day, ~30/month)
**Focus**: Drug-Disease relationship extraction
**Estimated Cost**: $22-25/month

---

## Architecture Principles

1. **Serverless-First**: Minimize operational overhead and costs using Lambda, API Gateway
2. **Pay-Per-Use**: Most services scale to zero when not processing
3. **Cost-Optimized**: Single-AZ, VPC endpoints instead of NAT Gateway
4. **Scalable Foundation**: Can grow to 100+ papers/day without major redesign

---

## System Architecture

### 1. Ingestion Layer

**Purpose**: Accept medical papers and queue them for processing

**Components**:
- **API Gateway (REST)**: Public endpoint for paper submission
  - Method: `POST /papers`
  - Authentication: API Key (initial MVP)
  - Payload: Paper metadata + abstract text or S3 reference

- **Lambda (Ingestion)**:
  - Runtime: Python 3.11
  - Memory: 256 MB
  - Timeout: 30s
  - Actions:
    1. Validate input (required fields, format)
    2. Generate unique paper ID
    3. Store raw paper in S3
    4. Send message to SQS queue
    5. Return paper ID to caller

- **S3 Bucket (papers/)**:
  - Lifecycle: Transition to IA after 30 days
  - Structure: `papers/{year}/{month}/{paper-id}.json`

- **SQS Queue (processing-queue)**:
  - Standard queue
  - Visibility timeout: 5 minutes
  - Dead Letter Queue: 3 retries

**Data Flow**:
```
User → API Gateway → Lambda → S3 (paper stored)
                             → SQS (queued for processing)
```

---

### 2. ML Processing Layer

**Purpose**: Extract medical entities and relationships using BioBERT

**Components**:

- **Lambda (Orchestrator)**:
  - Polls SQS queue
  - Retrieves paper from S3
  - Invokes ML inference
  - Handles retry logic

- **Lambda Container (BioBERT Inference)**:
  - Base Image: `public.ecr.aws/lambda/python:3.11`
  - Model: `dmis-lab/biobert-base-cased-v1.2` or `PubMedBERT`
  - Memory: 3 GB (minimum for model)
  - Timeout: 5 minutes
  - Storage: 10 GB ephemeral

  **Processing Steps**:
  1. Load pre-trained BioBERT model (cached in /tmp)
  2. Named Entity Recognition (NER):
     - Extract diseases (DISO entities)
     - Extract drugs (CHEM entities)
  3. Relation Extraction (RE):
     - Identify drug-disease pairs
     - Classify relationship (treats, causes, prevents)
     - Generate confidence scores
  4. Output structured JSON with extractions

- **Lambda (Entity Linker)**:
  - Maps extracted entities to standard IDs
  - Diseases → MeSH IDs (via NCBI API or local cache)
  - Drugs → RxNorm CUIs (via RxNav API)
  - Handles ambiguity (multiple matches → flag for human review)

- **ECR (Container Registry)**:
  - Stores ML model container images
  - Private repository
  - Lifecycle: Keep last 3 images

**Data Flow**:
```
SQS → Orchestrator → BioBERT Container → Entity Linker → RDS
                                                        (extraction queue)
```

**Extraction Output Schema**:
```json
{
  "paper_id": "abc123",
  "extractions": [
    {
      "id": "ext_001",
      "subject": {
        "text": "aspirin",
        "type": "DRUG",
        "mesh_id": "D001241",
        "rxnorm_cui": "1191"
      },
      "predicate": "TREATS",
      "object": {
        "text": "myocardial infarction",
        "type": "DISEASE",
        "mesh_id": "D009203"
      },
      "confidence": 0.89,
      "source_span": "text snippet showing relationship",
      "status": "pending_validation"
    }
  ]
}
```

---

### 3. Validation Layer

**Purpose**: Human-in-the-loop review and correction of extractions

**Components**:

- **CloudFront + S3 (Static UI)**:
  - Single Page Application (React)
  - Hosted in S3, distributed via CloudFront
  - HTTPS only, custom domain optional

- **Cognito User Pool**:
  - Authentication for medical experts
  - Users: 1-5 validators initially
  - MFA: Optional but recommended

- **API Gateway (Validation API)**:
  - REST endpoints:
    - `GET /queue` - Fetch pending extractions
    - `GET /extractions/{id}` - Get extraction details
    - `POST /extractions/{id}/validate` - Approve/reject/edit
    - `GET /papers/{id}` - Retrieve source paper
  - Authorization: Cognito JWT tokens

- **Lambda (Validation Logic)**:
  - CRUD operations on extraction queue
  - Business logic:
    - Mark extractions as approved/rejected
    - Store correction reasons
    - Trigger graph update on approval
    - Collect training data for model improvement

**Validation UI Features**:
1. **Queue View**: List pending extractions, sorted by confidence (lowest first)
2. **Detail View**:
   - Source paper with highlighted entities
   - Extracted relationship visualization
   - Confidence scores
   - Actions: ✓ Approve | ✗ Reject | ✎ Edit
3. **Feedback Form**:
   - Rejection reasons (wrong entity, wrong relation, etc.)
   - Ability to correct entity linking
   - Add notes for model training

**Data Flow**:
```
User → CloudFront → S3 (React App)
     → API Gateway (Auth: Cognito) → Lambda → RDS
```

---

### 4. Knowledge Graph Storage

**Purpose**: Store validated medical knowledge as a queryable graph

**Components**:

- **RDS PostgreSQL with Apache AGE**:
  - Instance: `db.t4g.micro` (2 vCPU, 1 GB RAM)
  - Storage: 20 GB GP3
  - Deployment: Single-AZ (Multi-AZ later)
  - Extension: Apache AGE for graph queries (Cypher syntax)

  **Why PostgreSQL + AGE vs Neptune?**
  - Cost: $12/month vs $350/month
  - Sufficient for MVP (<1M triples)
  - Familiar SQL interface
  - Easy migration to Neptune when needed

**Schema Design**:

**Relational Tables**:
```sql
-- Papers metadata
papers (
  id UUID PRIMARY KEY,
  title TEXT,
  abstract TEXT,
  authors TEXT[],
  pub_date DATE,
  source TEXT,
  created_at TIMESTAMP
)

-- Extraction queue
extractions (
  id UUID PRIMARY KEY,
  paper_id UUID REFERENCES papers(id),
  subject_text TEXT,
  subject_type TEXT,
  subject_mesh_id TEXT,
  predicate TEXT,
  object_text TEXT,
  object_type TEXT,
  object_mesh_id TEXT,
  confidence FLOAT,
  source_span TEXT,
  status TEXT, -- pending, approved, rejected
  validated_by TEXT,
  validated_at TIMESTAMP,
  feedback TEXT
)

-- Training data for continuous learning
training_examples (
  id UUID PRIMARY KEY,
  extraction_id UUID REFERENCES extractions(id),
  is_correct BOOLEAN,
  correction_type TEXT,
  notes TEXT,
  created_at TIMESTAMP
)
```

**Graph Schema (Apache AGE)**:
```cypher
// Nodes
(:Drug {mesh_id, rxnorm_cui, name, synonyms[]})
(:Disease {mesh_id, name, synonyms[]})
(:Paper {id, title, pub_date})

// Edges
(:Drug)-[:TREATS {confidence, paper_id, validated_at}]->(:Disease)
(:Drug)-[:CAUSES {confidence, paper_id, validated_at}]->(:Disease)
(:Drug)-[:PREVENTS {confidence, paper_id, validated_at}]->(:Disease)
(:Paper)-[:MENTIONS]->(:Drug)
(:Paper)-[:MENTIONS]->(:Disease)
```

**Data Flow**:
```
Validation Lambda (approval) → Insert into AGE graph
                              → Update extraction status
```

---

### 5. TTL Export & Graph Serving

**Purpose**: Generate standard RDF/TTL files for knowledge sharing

**Components**:

- **Lambda (TTL Generator)**:
  - Triggered by: EventBridge schedule (daily) or manual invocation
  - Queries RDS graph for approved triples
  - Generates .ttl file using RDF vocabulary
  - Stores in S3

**TTL Output Format**:
```turtle
@prefix : <http://example.org/medical-kg#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix mesh: <http://id.nlm.nih.gov/mesh/> .
@prefix rxnorm: <http://purl.bioontology.org/ontology/RXNORM/> .

rxnorm:1191 a :Drug ;
    rdfs:label "Aspirin" ;
    :treats mesh:D009203 ;
    :confidence "0.89"^^xsd:float ;
    :sourceDocument :paper_abc123 .

mesh:D009203 a :Disease ;
    rdfs:label "Myocardial Infarction" .
```

- **S3 Bucket (knowledge-graph/)**:
  - Stores TTL exports
  - Versioned (track graph evolution)
  - Structure: `knowledge-graph/exports/{date}/graph.ttl`

**Export Triggers**:
1. Scheduled (EventBridge): Daily at 2 AM
2. Manual: API call to trigger on-demand export
3. Webhook: After N new validations

---

### 6. Monitoring & Observability

**Components**:

- **CloudWatch Logs**:
  - All Lambda function logs
  - API Gateway access logs
  - RDS slow query logs

- **CloudWatch Metrics**:
  - Custom metrics:
    - Papers processed per day
    - Extraction confidence distribution
    - Validation rate (approved vs rejected)
    - Processing latency
  - Alarms:
    - Lambda errors > 5%
    - SQS queue depth > 100
    - RDS CPU > 80%

- **CloudWatch Dashboards**:
  - System health overview
  - Processing pipeline metrics
  - Cost tracking

- **Systems Manager Parameter Store**:
  - Configuration values (non-secret)
  - Model version tracking
  - Feature flags

- **Secrets Manager** (optional):
  - RDS credentials (can use IAM auth instead)
  - Third-party API keys (MeSH, RxNorm)

---

## Networking Architecture

### VPC Design (Cost-Optimized)

```
VPC: 10.0.0.0/16

Availability Zones: us-east-1a, us-east-1b

Subnets:
- Public Subnet A: 10.0.1.0/24 (us-east-1a)
- Public Subnet B: 10.0.2.0/24 (us-east-1b)
- Private Subnet A: 10.0.10.0/24 (us-east-1a) - Lambda, RDS
- Private Subnet B: 10.0.11.0/24 (us-east-1b) - RDS standby (future)

NAT Gateway: NONE (use VPC Endpoints)

VPC Endpoints:
- com.amazonaws.us-east-1.s3 (Gateway type, free)
- com.amazonaws.us-east-1.ecr.api (Interface, $7.30/month)
- com.amazonaws.us-east-1.ecr.dkr (Interface, $7.30/month)
- com.amazonaws.us-east-1.sqs (Interface, $7.30/month)
```

**Cost Savings**: VPC Endpoints (~$22/month) vs NAT Gateway (~$32/month) + data transfer

### Security Groups

**SG-Lambda** (attached to all Lambda functions):
- Outbound: All traffic (0.0.0.0/0) - for API calls
- Inbound: None

**SG-RDS** (attached to PostgreSQL):
- Inbound: Port 5432 from SG-Lambda
- Outbound: None

**SG-VPC-Endpoints**:
- Inbound: Port 443 from SG-Lambda
- Outbound: None

---

## Data Flow Examples

### Complete Paper Processing Flow

```
1. Paper Submission
   User → POST /papers (API Gateway)
        → Lambda Ingestion
        → S3 (store paper)
        → SQS (queue message)
        → Response: paper_id

2. ML Processing (Async)
   SQS → Lambda Orchestrator
       → Lambda BioBERT Container
          - Load model
          - Extract entities (drugs, diseases)
          - Extract relations (TREATS, CAUSES, etc.)
       → Lambda Entity Linker
          - Map to MeSH IDs
          - Map to RxNorm CUIs
       → RDS (save extractions, status=pending)

3. Validation
   User → GET /queue (API Gateway + Cognito)
        → Lambda Validation
        → RDS query pending extractions
        → Response: extraction list

   User → POST /extractions/{id}/validate
        → Lambda Validation
        → RDS update (status=approved, feedback)
        → Trigger: Lambda TTL Generator (EventBridge)

4. Graph Construction
   Lambda TTL → Query RDS (AGE graph)
             → Generate TTL triples
             → S3 (save TTL file)

5. Query Knowledge
   User → SPARQL endpoint (future)
        → RDS AGE graph query
        → Response: results
```

---

## Scalability Considerations

### Current Capacity (MVP)
- **Throughput**: 1 paper/day (can handle 100/day)
- **Storage**: 20 GB RDS (good for ~500K triples)
- **Concurrent Processing**: 1 paper at a time (SQS + Lambda)

### Scale Triggers & Solutions

| Metric | MVP Limit | Solution |
|--------|-----------|----------|
| Papers/day | 100 | Increase Lambda concurrency |
| Graph size | 50 GB | Upgrade RDS to db.t4g.small |
| Graph size | 100 GB | Migrate to Neptune |
| Query load | 10 QPS | Add read replicas |
| ML latency | 5 min/paper | Move to SageMaker or EC2 GPU |
| Validators | 10 | No change needed |

### Migration Path to Production Scale

**Phase 1 (Current)**: 1-100 papers/day
- Lambda + RDS PostgreSQL

**Phase 2**: 100-1000 papers/day
- Add SageMaker endpoint for ML
- Upgrade RDS to db.r6g.large
- Add ElastiCache for query caching

**Phase 3**: 1000+ papers/day
- Migrate to Neptune
- Add EMR for batch processing
- Implement data lake (S3 + Athena)

---

## Security Considerations

### Data Protection
- **Encryption at Rest**:
  - S3: AES-256 (SSE-S3)
  - RDS: Enabled (AWS managed keys)

- **Encryption in Transit**:
  - HTTPS only (API Gateway, CloudFront)
  - TLS 1.2+ for all AWS service connections

### Access Control
- **IAM Roles**: Least privilege for Lambda functions
- **Cognito**: User authentication and authorization
- **API Keys**: Rate limiting on public API

### Compliance
- **HIPAA**: Not currently required (using public abstracts)
- **GDPR**: No PII collected
- **Data Provenance**: Track paper sources and validation history

### Audit Trail
- **CloudTrail**: All API calls logged
- **RDS Audit**: Track all data modifications
- **Validation History**: immutable log of approvals/rejections

---

## Deployment Strategy

### Infrastructure as Code
- **Terraform**: All AWS resources defined in code
- **Modules**:
  - `network` - VPC, subnets, security groups
  - `ingestion` - API Gateway, Lambda, S3, SQS
  - `ml-processing` - Lambda containers, ECR
  - `validation` - CloudFront, S3, Cognito, API Gateway
  - `database` - RDS PostgreSQL with AGE
  - `monitoring` - CloudWatch, dashboards, alarms

### CI/CD Pipeline (Future)
- **GitHub Actions**:
  - Terraform plan/apply on PR
  - Lambda function deployment
  - Container image builds
  - Run integration tests

### Environments
- **Development**: Local Docker Compose + LocalStack
- **Staging**: Minimal AWS (scaled-down RDS)
- **Production**: Full architecture as described

---

## Cost Breakdown (Detailed)

### Monthly Costs

**Compute**:
- Lambda (all functions): $0.50 (free tier covers most)
- Lambda Container (ML): $1.00 (storage) + $0.05 (execution)

**Storage**:
- S3 (papers + UI + TTL): $0.25
- RDS storage (20 GB): $2.30
- RDS backups: $2.00
- ECR: $1.00

**Database**:
- RDS instance (db.t4g.micro): $12.41

**Networking**:
- VPC Endpoints (3x): $21.90
- Data transfer: $0.50

**Other**:
- CloudWatch Logs: $2.50
- API Gateway: $0.10
- EventBridge: Free tier
- Cognito: Free tier
- Route53 (optional): $0.50

**Total**: ~$22-25/month

### Cost Optimization Tips
1. Use VPC endpoints instead of NAT Gateway: Save $10/month
2. Enable S3 Intelligent-Tiering: Save on storage
3. Use Lambda instead of EC2: Pay per use
4. Single-AZ RDS: Save 50% vs Multi-AZ
5. Reserved Instances: 40% discount after 6 months

---

## Technical Debt & Future Improvements

### Known Limitations (MVP)
1. **No automatic model retraining**: Manual process
2. **Single-threaded processing**: One paper at a time
3. **No caching**: Every request hits database
4. **Basic error handling**: Limited retry logic
5. **No real-time updates**: Polling-based UI

### Roadmap
- [ ] Implement automatic retraining pipeline (Step Functions)
- [ ] Add Redis cache for frequent queries
- [ ] Implement WebSocket for real-time validation UI
- [ ] Add support for full-text papers (not just abstracts)
- [ ] Expand to multi-domain extraction (gene-disease, drug-drug)
- [ ] Integrate with external knowledge bases (UMLS, DisGeNET)
- [ ] Add SPARQL query endpoint
- [ ] Implement graph visualization UI
- [ ] Add automated quality metrics and testing

---

## References

### Technologies Used
- **Apache AGE**: https://age.apache.org/
- **BioBERT**: https://github.com/dmis-lab/biobert
- **PubMedBERT**: https://huggingface.co/microsoft/BiomedNLP-PubMedBERT-base-uncased-abstract
- **MeSH**: https://www.nlm.nih.gov/mesh/
- **RxNorm**: https://www.nlm.nih.gov/research/umls/rxnorm/

### AWS Documentation
- Lambda Container Images: https://docs.aws.amazon.com/lambda/latest/dg/images-create.html
- RDS PostgreSQL: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/
- VPC Endpoints: https://docs.aws.amazon.com/vpc/latest/privatelink/

### Related Papers
- BioBERT: https://arxiv.org/abs/1901.08746
- Relation Extraction in Biomedical Text: Various (TBD)
- Knowledge Graph Construction: Various (TBD)
