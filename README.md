# Medical Knowledge Graph - MVP Architecture

> An AI-powered system to extract drug-disease relationships from medical literature and build a comprehensive knowledge graph.

## Overview

This system processes medical papers through an ML pipeline (BioBERT), validates extractions with human experts, and constructs a queryable medical knowledge graph stored as RDF/TTL format.

**Status**: Architecture Design Phase
**Target Scale**: 1-30 papers/day (MVP)
**Estimated Cost**: $42-45/month

---

## Documentation

### Core Documents

1. **[ARCHITECTURE.md](docs/ARCHITECTURE.md)** - Comprehensive system architecture
   - Component descriptions
   - Data flows
   - Security considerations
   - Scalability roadmap

2. **[COST-ESTIMATE.md](docs/COST-ESTIMATE.md)** - Detailed cost breakdown
   - Monthly costs by service
   - Scaling projections
   - Optimization strategies
   - ROI analysis

3. **[TECHNOLOGY-STACK.md](docs/TECHNOLOGY-STACK.md)** - Technology recommendations
   - ML models and frameworks
   - Database setup
   - Frontend/backend tools
   - Development environment

### Diagrams

4. **[architecture.mmd](diagrams/architecture.mmd)** - Mermaid architecture diagram
   - View in GitHub or use Mermaid viewer
   - Shows all AWS components and connections

5. **[data-flow.md](diagrams/data-flow.md)** - Data flow diagrams
   - Paper ingestion flow
   - ML extraction pipeline
   - Validation workflow
   - Graph generation

---

## Quick Start

### View Architecture Diagram

The architecture diagram is in Mermaid format. View it using:

**Option 1: GitHub**
- Push to GitHub and view `diagrams/architecture.mmd` directly

**Option 2: VS Code**
- Install "Markdown Preview Mermaid Support" extension
- Open `diagrams/architecture.mmd`

**Option 3: Online Viewer**
- Copy contents of `diagrams/architecture.mmd`
- Paste into https://mermaid.live/

**Option 4: CLI**
```bash
npm install -g @mermaid-js/mermaid-cli
mmdc -i diagrams/architecture.mmd -o diagrams/architecture.png
```

---

## Architecture Summary

### High-Level Flow

```
1. Paper Submission (API)
   ↓
2. ML Extraction (BioBERT in Lambda Container)
   ↓
3. Entity Linking (MeSH, RxNorm)
   ↓
4. Human Validation (Web UI)
   ↓
5. Knowledge Graph (PostgreSQL + Apache AGE)
   ↓
6. TTL Export (RDF/OWL format)
```

### Key Components

| Layer | AWS Services | Purpose |
|-------|--------------|---------|
| **Ingestion** | API Gateway, Lambda, S3, SQS | Accept papers and queue processing |
| **ML Processing** | Lambda Container (BioBERT), ECR | Extract entities and relationships |
| **Validation** | CloudFront, S3, Cognito, Lambda | Human review interface |
| **Storage** | RDS PostgreSQL + Apache AGE | Graph database |
| **Networking** | VPC, VPC Endpoints | Secure, cost-optimized connectivity |
| **Monitoring** | CloudWatch, EventBridge | Logging, metrics, automation |

---

## Cost Summary

### Monthly Breakdown

```
Compute (Lambda + ECR):      $  1.05
Database (RDS PostgreSQL):   $ 16.71
Networking (VPC Endpoints):  $ 21.91
Storage (S3):                $  0.13
Monitoring (CloudWatch):     $  2.53
────────────────────────────────────
TOTAL:                       $ 42.33/month
```

**Variable costs**: ~$0.002 per paper processed (negligible)

### Scaling Economics

| Papers/Day | Monthly Cost | Cost per Paper |
|-----------|--------------|----------------|
| 1 | $42 | $1.40 |
| 10 | $43 | $0.14 |
| 100 | $87 | $0.03 |

---

## Technology Highlights

### ML Model
- **PubMedBERT** (`microsoft/BiomedNLP-PubMedBERT-base-uncased-abstract`)
- Pre-trained on 14M PubMed abstracts
- Fine-tuned for biomedical relation extraction

### Graph Database
- **PostgreSQL 14 + Apache AGE**
- Cypher query support
- Cost: $12/month (vs Neptune $350/month)
- Scales to 1M+ triples before migration needed

### Frontend
- **React 18 + TypeScript**
- Material-UI components
- Cognito authentication
- Hosted on S3 + CloudFront

### Infrastructure
- **100% Terraform**
- Serverless-first architecture
- VPC endpoints for cost optimization
- Single-AZ for MVP (Multi-AZ ready)

---

## MVP Scope

### What's Included
- ✅ Drug-disease relationship extraction
- ✅ Human-in-the-loop validation
- ✅ Knowledge graph storage (AGE)
- ✅ TTL/RDF export
- ✅ Basic monitoring and alerting

### Phase 2 (Future)
- ⏳ Automated model retraining
- ⏳ Multi-domain extraction (genes, procedures)
- ⏳ Full-text PDF processing
- ⏳ SPARQL query endpoint
- ⏳ Graph visualization UI

---

## Next Steps

### 1. Deploy Infrastructure ✅

Infrastructure code is ready! Deploy using:

```bash
cd tf-contexa-infrastructure
make plan
make apply
```

See [tf-contexa-infrastructure/QUICKSTART.md](tf-contexa-infrastructure/QUICKSTART.md) for details.

**Status**: ✅ Complete - Ready to deploy

### 2. Post-Infrastructure Tasks

- [ ] Install Apache AGE extension on RDS
- [ ] Build Lambda functions for ML processing
- [ ] Create React validation UI
- [ ] Deploy containers to ECR
- [ ] Configure API Gateway
- [ ] Set up Cognito authentication
- [ ] Test end-to-end pipeline

### 3. Optional Enhancements

- [ ] Add CloudWatch dashboards
- [ ] Set up billing alerts
- [ ] Configure automated backups
- [ ] Add monitoring and alarms
- [ ] Implement CI/CD pipeline

---

## Project Timeline (Estimated)

| Phase | Duration | Deliverables |
|-------|----------|--------------|
| **Architecture** | 1 week | ✅ Complete |
| **Infrastructure** | 2-3 weeks | Terraform, VPC, RDS, Lambda scaffold |
| **ML Pipeline** | 2-3 weeks | BioBERT integration, entity linking |
| **Validation UI** | 2 weeks | React app, Cognito auth |
| **Integration** | 1 week | End-to-end testing |
| **MVP Launch** | 1 week | Deploy, monitor, iterate |
| **Total** | **8-10 weeks** | Production-ready MVP |

---

## Key Decisions Made

### Architecture
- ✓ Serverless-first (Lambda over EC2)
- ✓ PostgreSQL + AGE (over Neptune for MVP)
- ✓ VPC Endpoints (over NAT Gateway)
- ✓ Single-AZ RDS (upgrade to Multi-AZ later)

### ML/NLP
- ✓ PubMedBERT (over BioBERT)
- ✓ Pre-trained + fine-tuned (not training from scratch)
- ✓ Start with abstracts (not full-text PDFs)
- ✓ Drug-disease focus (expand domains later)

### Data Standards
- ✓ MeSH for diseases
- ✓ RxNorm for drugs
- ✓ RDF/TTL for knowledge graph export
- ✓ Apache AGE for graph queries

---

## Questions & Support

### Common Questions

**Q: Can I use Neptune instead of PostgreSQL + AGE?**
A: Yes, but it costs $350/month vs $12/month. Recommended only for production scale (>1M triples, high query load).

**Q: What if I want to process 100+ papers/day?**
A: The architecture supports this with minimal changes. See scaling section in ARCHITECTURE.md.

**Q: Can I train my own model from scratch?**
A: Not recommended for MVP. Start with fine-tuning PubMedBERT on your domain, then consider custom models in Phase 2.

**Q: How do I add support for genes or proteins?**
A: Extend the entity types in the ML pipeline and add mappings to gene ontologies (HGNC, UniProt). See Phase 2 roadmap.

---

## File Structure

```
.
├── README.md                         # This file
├── docs/
│   ├── ARCHITECTURE.md               # Detailed architecture
│   ├── COST-ESTIMATE.md              # Cost breakdown
│   └── TECHNOLOGY-STACK.md           # Tech recommendations
├── diagrams/
│   ├── architecture.mmd              # Mermaid diagram
│   └── data-flow.md                  # Flow diagrams
├── tf-contexa-infrastructure/        # ✅ Terraform infrastructure
│   ├── QUICKSTART.md                 # Quick deployment guide
│   ├── README.md                     # Infrastructure docs
│   ├── Makefile                      # Convenient commands
│   ├── main.tf                       # VPC configuration
│   ├── security_groups.tf            # Security groups
│   ├── rds.tf                        # PostgreSQL + AGE
│   ├── s3.tf                         # Storage buckets
│   ├── sqs.tf                        # Processing queue
│   └── tfvars/dev.tfvars             # Development config
└── old/                              # Previous project files
```

---

## References

### Medical Standards
- **MeSH**: https://www.nlm.nih.gov/mesh/
- **RxNorm**: https://www.nlm.nih.gov/research/umls/rxnorm/
- **UMLS**: https://www.nlm.nih.gov/research/umls/

### ML Models
- **PubMedBERT**: https://huggingface.co/microsoft/BiomedNLP-PubMedBERT-base-uncased-abstract
- **BioBERT**: https://github.com/dmis-lab/biobert
- **SciSpaCy**: https://allenai.github.io/scispacy/

### Graph Technology
- **Apache AGE**: https://age.apache.org/
- **RDF/OWL**: https://www.w3.org/TR/rdf11-primer/

### AWS
- **Lambda Containers**: https://docs.aws.amazon.com/lambda/latest/dg/images-create.html
- **RDS PostgreSQL**: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/

---

## License

TBD

---

## Contact

For questions about this architecture, please contact the project team.

**Last Updated**: January 2025
