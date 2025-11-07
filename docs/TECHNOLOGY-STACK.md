# Technology Stack & Recommendations

## Overview
This document provides specific technology recommendations, libraries, models, and tools for building the Medical Knowledge Graph MVP.

---

## ML Models & Frameworks

### 1. Pre-trained Language Models

**Recommended: PubMedBERT**
- **Model**: `microsoft/BiomedNLP-PubMedBERT-base-uncased-abstract`
- **Source**: HuggingFace
- **Size**: ~420 MB
- **Context**: 512 tokens
- **Advantages**:
  - Pre-trained on PubMed abstracts
  - Better performance on biomedical text than BioBERT
  - Actively maintained by Microsoft
  - Good documentation

**Alternative: BioBERT**
- **Model**: `dmis-lab/biobert-base-cased-v1.2`
- **Source**: HuggingFace
- **Size**: ~430 MB
- **Advantages**:
  - Widely cited in research
  - Pre-trained on PubMed + PMC
  - Good for entity recognition

**For Relation Extraction (Fine-tuned)**:
- Base: PubMedBERT
- Fine-tune on: **ChemProt** dataset (drug-protein interactions)
- Or: **BC5CDR** dataset (chemical-disease relations)
- Available from: https://github.com/ncbi-nlp/BLUE_Benchmark

---

### 2. ML Framework Stack

```python
# Core ML
transformers==4.35.0        # HuggingFace models
torch==2.1.0               # PyTorch backend
datasets==2.15.0           # Dataset loading

# NER/RE specific
spacy==3.7.0               # NLP pipelines
scispacy==0.5.3            # Scientific NLP models
en-core-sci-md             # Medium scientific model

# Entity linking
requests==2.31.0           # API calls
lxml==4.9.3               # XML parsing for MeSH
```

**Lambda Container Base**:
```dockerfile
FROM public.ecr.aws/lambda/python:3.11

# Install PyTorch (CPU-only, smaller)
RUN pip install torch==2.1.0+cpu -f https://download.pytorch.org/whl/torch_stable.html
RUN pip install transformers==4.35.0

# Copy model files (baked into image)
COPY models/ /var/task/models/
```

---

## Entity Linking & Normalization

### MeSH (Medical Subject Headings)

**API**: NCBI E-utilities
- Endpoint: `https://eutils.ncbi.nlm.nih.gov/entrez/eutils/`
- Free tier: 3 requests/second (no API key), 10/sec with key
- Cost: Free

**Usage Example**:
```python
import requests

def link_to_mesh(disease_name):
    url = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi"
    params = {
        "db": "mesh",
        "term": disease_name,
        "retmode": "json"
    }
    response = requests.get(url, params=params)
    # Parse and return MeSH ID
```

**Caching Strategy**: Store results in RDS to minimize API calls

---

### RxNorm (Drug Normalization)

**API**: RxNav API
- Endpoint: `https://rxnav.nlm.nih.gov/REST/`
- Free tier: Unlimited
- Cost: Free

**Endpoints**:
- `/approximateTerm?term={drug_name}` - Fuzzy search
- `/rxcui/{id}/allrelated` - Get all related concepts

**Usage Example**:
```python
def link_to_rxnorm(drug_name):
    url = f"https://rxnav.nlm.nih.gov/REST/approximateTerm.json"
    params = {"term": drug_name, "maxEntries": 5}
    response = requests.get(url, params=params)
    # Return best match RxCUI
```

---

### Alternative: UMLS API

**Unified Medical Language System**
- Requires free account registration
- More comprehensive than MeSH alone
- Includes MeSH, SNOMED, RxNorm, and more
- Endpoint: `https://uts-ws.nlm.nih.gov/rest/`

**When to use**: If you expand beyond drug-disease (genes, procedures, etc.)

---

## Database Technologies

### PostgreSQL with Apache AGE

**Version**: PostgreSQL 14+ with AGE 1.4.0

**Installation** (for local dev):
```bash
# PostgreSQL
brew install postgresql@14

# Apache AGE extension
git clone https://github.com/apache/age.git
cd age
make install
```

**RDS Configuration**:
- Engine: PostgreSQL 14.9
- Instance: db.t4g.micro
- Parameter group: Custom with AGE extension enabled

**AGE Setup**:
```sql
CREATE EXTENSION age;
LOAD 'age';
SET search_path = ag_catalog, "$user", public;

-- Create graph
SELECT create_graph('medical_kg');
```

**Graph Schema**:
```sql
-- Create nodes
SELECT * FROM cypher('medical_kg', $$
  CREATE (:Drug {mesh_id: 'D001241', name: 'Aspirin', rxnorm_cui: '1191'})
$$) as (v agtype);

SELECT * FROM cypher('medical_kg', $$
  CREATE (:Disease {mesh_id: 'D009203', name: 'Myocardial Infarction'})
$$) as (v agtype);

-- Create relationships
SELECT * FROM cypher('medical_kg', $$
  MATCH (d:Drug {mesh_id: 'D001241'}), (di:Disease {mesh_id: 'D009203'})
  CREATE (d)-[:TREATS {confidence: 0.89, paper_id: 'abc123'}]->(di)
$$) as (v agtype);

-- Query
SELECT * FROM cypher('medical_kg', $$
  MATCH (d:Drug)-[r:TREATS]->(di:Disease)
  WHERE r.confidence > 0.8
  RETURN d.name, di.name, r.confidence
$$) as (drug_name agtype, disease_name agtype, confidence agtype);
```

**Python Driver**:
```python
import psycopg2
from psycopg2.extras import RealDictCursor

conn = psycopg2.connect(
    host="your-rds-endpoint",
    database="medical_kg_db",
    user="postgres",
    password="your-password"
)

# Execute Cypher query via SQL
cursor = conn.cursor(cursor_factory=RealDictCursor)
cursor.execute("""
    SELECT * FROM cypher('medical_kg', $$
      MATCH (d:Drug)-[:TREATS]->(di:Disease)
      RETURN d, di
    $$) as (drug agtype, disease agtype);
""")
results = cursor.fetchall()
```

---

## Frontend Technologies

### Validation UI

**Framework**: React 18 with TypeScript
```json
{
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "typescript": "^5.0.0",

    "aws-amplify": "^6.0.0",        // Cognito auth
    "axios": "^1.6.0",              // API calls

    "@mui/material": "^5.14.0",     // UI components
    "@mui/icons-material": "^5.14.0",

    "react-router-dom": "^6.20.0",  // Routing
    "react-query": "^3.39.0",       // Data fetching

    "recharts": "^2.10.0"           // Charts/metrics
  }
}
```

**Build & Deploy**:
```bash
# Build
npm run build

# Deploy to S3
aws s3 sync build/ s3://your-ui-bucket/ --delete

# Invalidate CloudFront
aws cloudfront create-invalidation \
  --distribution-id YOUR_DIST_ID \
  --paths "/*"
```

---

### UI Components

**Key Pages**:
1. **Login** (`/login`)
   - Cognito authentication via Amplify

2. **Queue Dashboard** (`/queue`)
   - List of pending extractions
   - Sort by confidence (lowest first)
   - Filter by paper, entity type

3. **Validation Detail** (`/validate/:id`)
   - Source paper display
   - Entity highlighting
   - Relationship visualization
   - Approve/Reject/Edit actions

4. **Metrics** (`/metrics`)
   - Papers processed
   - Validation rate
   - Confidence distribution
   - Model performance over time

**Example Component**:
```typescript
// ExtractionCard.tsx
import { Card, Chip, Button } from '@mui/material';

interface Extraction {
  id: string;
  subject: { text: string; type: string; mesh_id: string };
  predicate: string;
  object: { text: string; type: string; mesh_id: string };
  confidence: number;
}

export function ExtractionCard({ extraction, onValidate }: Props) {
  return (
    <Card>
      <Chip label={extraction.subject.text} color="primary" />
      <span>{extraction.predicate}</span>
      <Chip label={extraction.object.text} color="secondary" />
      <span>Confidence: {(extraction.confidence * 100).toFixed(1)}%</span>

      <Button onClick={() => onValidate('approve')}>✓ Approve</Button>
      <Button onClick={() => onValidate('reject')}>✗ Reject</Button>
    </Card>
  );
}
```

---

## Backend/Lambda Technologies

### Python Packages

**requirements.txt**:
```txt
# AWS SDK
boto3==1.34.0
botocore==1.34.0

# Web framework (for validation API)
fastapi==0.104.0        # If using Lambda Web Adapter
mangum==0.17.0          # ASGI adapter for Lambda
pydantic==2.5.0         # Data validation

# Database
psycopg2-binary==2.9.9
SQLAlchemy==2.0.23

# ML (for inference Lambda)
transformers==4.35.0
torch==2.1.0
numpy==1.24.0

# Utils
python-dotenv==1.0.0
requests==2.31.0
python-json-logger==2.0.7
```

**Lambda Layer** (shared dependencies):
```bash
mkdir python
pip install -r requirements.txt -t python/
zip -r lambda-layer.zip python/
aws lambda publish-layer-version \
  --layer-name medical-kg-deps \
  --zip-file fileb://lambda-layer.zip \
  --compatible-runtimes python3.11
```

---

### Lambda Function Structure

```
lambda/
├── ingestion/
│   ├── handler.py          # Main handler
│   ├── validator.py        # Input validation
│   └── requirements.txt
├── ml-processing/
│   ├── orchestrator/
│   │   ├── handler.py
│   │   └── requirements.txt
│   ├── inference/          # Container
│   │   ├── Dockerfile
│   │   ├── handler.py
│   │   ├── model_loader.py
│   │   └── requirements.txt
│   └── entity-linker/
│       ├── handler.py
│       ├── mesh_client.py
│       └── rxnorm_client.py
├── validation/
│   ├── handler.py
│   ├── models.py           # Pydantic models
│   └── db.py               # Database utils
└── ttl-generator/
    ├── handler.py
    └── rdf_builder.py
```

**Example Handler**:
```python
# lambda/ingestion/handler.py
import json
import boto3
import uuid
from datetime import datetime

s3 = boto3.client('s3')
sqs = boto3.client('sqs')

def lambda_handler(event, context):
    # Parse API Gateway event
    body = json.loads(event['body'])

    # Validate
    if 'abstract' not in body or 'title' not in body:
        return {
            'statusCode': 400,
            'body': json.dumps({'error': 'Missing required fields'})
        }

    # Generate ID
    paper_id = str(uuid.uuid4())

    # Store in S3
    s3.put_object(
        Bucket='medical-kg-papers',
        Key=f'papers/{datetime.now().year}/{paper_id}.json',
        Body=json.dumps({
            'id': paper_id,
            'title': body['title'],
            'abstract': body['abstract'],
            'metadata': body.get('metadata', {}),
            'created_at': datetime.now().isoformat()
        })
    )

    # Queue for processing
    sqs.send_message(
        QueueUrl='https://sqs.us-east-1.amazonaws.com/xxx/processing-queue',
        MessageBody=json.dumps({'paper_id': paper_id})
    )

    return {
        'statusCode': 202,
        'body': json.dumps({'paper_id': paper_id, 'status': 'queued'})
    }
```

---

## RDF/TTL Generation

### Python Library: rdflib

```bash
pip install rdflib==7.0.0
```

**TTL Generator**:
```python
from rdflib import Graph, Namespace, Literal, URIRef
from rdflib.namespace import RDF, RDFS

# Define namespaces
MKG = Namespace("http://medical-kg.example.org/")
MESH = Namespace("http://id.nlm.nih.gov/mesh/")
RXNORM = Namespace("http://purl.bioontology.org/ontology/RXNORM/")

def generate_ttl(extractions):
    g = Graph()
    g.bind("mkg", MKG)
    g.bind("mesh", MESH)
    g.bind("rxnorm", RXNORM)

    for ext in extractions:
        # Create drug node
        drug = RXNORM[ext['subject']['rxnorm_cui']]
        g.add((drug, RDF.type, MKG.Drug))
        g.add((drug, RDFS.label, Literal(ext['subject']['text'])))

        # Create disease node
        disease = MESH[ext['object']['mesh_id']]
        g.add((disease, RDF.type, MKG.Disease))
        g.add((disease, RDFS.label, Literal(ext['object']['text'])))

        # Create relationship
        predicate = MKG[ext['predicate'].lower()]
        g.add((drug, predicate, disease))

        # Add metadata
        stmt = URIRef(f"http://medical-kg.example.org/statement/{ext['id']}")
        g.add((stmt, MKG.subject, drug))
        g.add((stmt, MKG.predicate, predicate))
        g.add((stmt, MKG.object, disease))
        g.add((stmt, MKG.confidence, Literal(ext['confidence'])))
        g.add((stmt, MKG.source, Literal(ext['paper_id'])))

    # Serialize to TTL
    return g.serialize(format='turtle')

# Usage
ttl_content = generate_ttl(validated_extractions)
s3.put_object(
    Bucket='medical-kg-exports',
    Key=f'exports/{datetime.now().date()}/graph.ttl',
    Body=ttl_content
)
```

---

## Infrastructure as Code

### Terraform Modules

**Recommended Structure**:
```
terraform/
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── database/
│   ├── compute/
│   ├── storage/
│   └── monitoring/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf
│       └── terraform.tfvars
└── main.tf
```

**Terraform Version**:
```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.30"
    }
  }

  backend "s3" {
    bucket = "medical-kg-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}
```

---

## Testing Tools

### Unit Testing
```python
# requirements-dev.txt
pytest==7.4.3
pytest-cov==4.1.0
pytest-mock==3.12.0
moto==4.2.0              # Mock AWS services
faker==20.1.0            # Fake data generation
```

**Example Test**:
```python
# tests/test_extraction.py
import pytest
from moto import mock_s3
from lambda.ingestion.handler import lambda_handler

@mock_s3
def test_paper_ingestion():
    # Setup
    event = {
        'body': json.dumps({
            'title': 'Test Paper',
            'abstract': 'Test abstract'
        })
    }

    # Execute
    response = lambda_handler(event, {})

    # Assert
    assert response['statusCode'] == 202
    body = json.loads(response['body'])
    assert 'paper_id' in body
```

### Integration Testing
- **LocalStack**: Mock AWS locally
- **Docker Compose**: Test full pipeline locally

---

## Monitoring & Observability

### Application Performance Monitoring

**AWS X-Ray** (built-in):
```python
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

patch_all()

@xray_recorder.capture('extract_entities')
def extract_entities(text):
    # Your code here
    pass
```

**Structured Logging**:
```python
import logging
from pythonjsonlogger import jsonlogger

logger = logging.getLogger()
logHandler = logging.StreamHandler()
formatter = jsonlogger.JsonFormatter()
logHandler.setFormatter(formatter)
logger.addHandler(logHandler)
logger.setLevel(logging.INFO)

# Usage
logger.info("Paper processed", extra={
    "paper_id": paper_id,
    "extractions_count": len(extractions),
    "avg_confidence": avg_conf
})
```

---

## Development Tools

### Local Development

**Docker Compose** (`docker-compose.yml`):
```yaml
version: '3.8'

services:
  postgres:
    image: apache/age:latest
    ports:
      - "5432:5432"
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: medical_kg
    volumes:
      - pg-data:/var/lib/postgresql/data

  localstack:
    image: localstack/localstack:latest
    ports:
      - "4566:4566"
    environment:
      SERVICES: s3,sqs,lambda
      DEBUG: 1
    volumes:
      - ./localstack:/etc/localstack/init/ready.d

  ui:
    build: ./frontend
    ports:
      - "3000:3000"
    volumes:
      - ./frontend:/app
    environment:
      REACT_APP_API_URL: http://localhost:4566

volumes:
  pg-data:
```

**Running locally**:
```bash
docker-compose up -d
python lambda/ingestion/handler.py  # Test locally
```

---

### CI/CD

**GitHub Actions** (`.github/workflows/deploy.yml`):
```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        run: terraform plan

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve

      - name: Build Lambda Container
        run: |
          docker build -t medical-kg-ml ./lambda/ml-processing/inference

      - name: Push to ECR
        run: |
          aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REGISTRY
          docker tag medical-kg-ml:latest $ECR_REGISTRY/medical-kg-ml:latest
          docker push $ECR_REGISTRY/medical-kg-ml:latest
```

---

## Security Tools

### Secrets Management

**AWS Secrets Manager** (optional):
```python
import boto3
import json

def get_secret(secret_name):
    client = boto3.client('secretsmanager', region_name='us-east-1')
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response['SecretString'])

# Usage
db_creds = get_secret('prod/medical-kg/db')
conn = psycopg2.connect(
    host=db_creds['host'],
    user=db_creds['username'],
    password=db_creds['password']
)
```

**Alternative: Parameter Store** (cheaper):
```python
import boto3

ssm = boto3.client('ssm')
response = ssm.get_parameter(
    Name='/medical-kg/db/password',
    WithDecryption=True
)
password = response['Parameter']['Value']
```

---

## Recommended Learning Resources

### ML/NLP
- HuggingFace Course: https://huggingface.co/course
- BioBERT Paper: https://arxiv.org/abs/1901.08746
- BLUE Benchmark: https://github.com/ncbi-nlp/BLUE_Benchmark

### Graph Databases
- Apache AGE Docs: https://age.apache.org/age-manual/master/intro/overview.html
- Cypher Query Language: https://neo4j.com/developer/cypher/

### AWS
- Lambda Best Practices: https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html
- Serverless Patterns: https://serverlessland.com/patterns

---

## Version Control & Project Structure

```
medical-knowledge-graph/
├── README.md
├── docs/
│   ├── ARCHITECTURE.md
│   ├── COST-ESTIMATE.md
│   └── TECHNOLOGY-STACK.md
├── terraform/
│   └── (infrastructure code)
├── lambda/
│   └── (function code)
├── frontend/
│   └── (React app)
├── tests/
│   └── (test suites)
├── scripts/
│   ├── deploy.sh
│   └── seed-data.sh
├── data/
│   └── sample-papers/
├── models/
│   └── (pre-trained model downloads)
└── docker-compose.yml
```

---

## Next Steps

See the main README for getting started with implementation.

Key decisions made:
✓ PubMedBERT for ML model
✓ PostgreSQL + Apache AGE for graph DB
✓ React + Material-UI for frontend
✓ Terraform for infrastructure
✓ Python 3.11 for all Lambda functions
