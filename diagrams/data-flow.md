# Data Flow Diagrams

## 1. Paper Ingestion Flow

```
┌─────────────┐
│   API User  │
│ (Submitter) │
└──────┬──────┘
       │ POST /papers
       │ {title, abstract, metadata}
       ▼
┌─────────────────┐
│  API Gateway    │
│  /papers        │
└────────┬────────┘
         │
         ▼
┌──────────────────────┐
│ Lambda: Ingestion    │
│ ┌──────────────────┐ │
│ │1. Validate input │ │
│ │2. Generate UUID  │ │
│ │3. Store to S3    │ │
│ │4. Send to SQS    │ │
│ └──────────────────┘ │
└─────┬────────┬───────┘
      │        │
      │        └─────────────┐
      │                      │
      ▼                      ▼
┌──────────┐          ┌─────────────┐
│    S3    │          │  SQS Queue  │
│ papers/  │          │ processing  │
└──────────┘          └──────┬──────┘
                             │
                             │ Message:
                             │ {paper_id, s3_key}
                             ▼
                      [ML Processing]
```

## 2. ML Extraction Flow

```
┌─────────────┐
│  SQS Queue  │
│ processing  │
└──────┬──────┘
       │ Poll (Lambda trigger)
       ▼
┌──────────────────────────┐
│ Lambda: Orchestrator     │
│ ┌──────────────────────┐ │
│ │1. Retrieve from SQS  │ │
│ │2. Fetch paper from S3│ │
│ │3. Invoke ML Lambda   │ │
│ │4. Handle errors      │ │
│ └──────────────────────┘ │
└────────┬─────────────────┘
         │ {paper_text}
         ▼
┌─────────────────────────────────────┐
│ Lambda Container: BioBERT           │
│ ┌─────────────────────────────────┐ │
│ │ NER (Named Entity Recognition)  │ │
│ │   ├─ Extract DRUG entities      │ │
│ │   └─ Extract DISEASE entities   │ │
│ │                                  │ │
│ │ RE (Relation Extraction)        │ │
│ │   ├─ Find entity pairs          │ │
│ │   ├─ Classify relationship      │ │
│ │   └─ Generate confidence        │ │
│ └─────────────────────────────────┘ │
└────────┬────────────────────────────┘
         │ [{subject, predicate, object, confidence}]
         ▼
┌──────────────────────────┐
│ Lambda: Entity Linker    │
│ ┌──────────────────────┐ │
│ │ For each entity:     │ │
│ │  - Lookup MeSH API   │ │
│ │  - Lookup RxNorm API │ │
│ │  - Handle ambiguity  │ │
│ │  - Add standard IDs  │ │
│ └──────────────────────┘ │
└────────┬─────────────────┘
         │ Enriched extractions
         ▼
┌────────────────────┐
│   RDS PostgreSQL   │
│                    │
│ INSERT INTO        │
│   extractions      │
│ SET                │
│   status='pending' │
└────────────────────┘
```

## 3. Validation Flow

```
┌──────────────┐
│ Medical      │
│ Expert User  │
└──────┬───────┘
       │ 1. Login
       ▼
┌─────────────────┐
│ Cognito         │
│ Authentication  │
└────────┬────────┘
         │ JWT Token
         ▼
┌─────────────────┐
│   CloudFront    │
│   Static SPA    │
└────────┬────────┘
         │ Load React App
         │
         │ 2. GET /queue
         ▼
┌─────────────────────────┐
│  API Gateway            │
│  (Validation API)       │
│  ┌────────────────────┐ │
│  │ Verify JWT token   │ │
│  └────────────────────┘ │
└────────┬────────────────┘
         │
         ▼
┌──────────────────────────┐
│ Lambda: Validation CRUD  │
│ ┌──────────────────────┐ │
│ │ Query RDS for        │ │
│ │ pending extractions  │ │
│ └──────────────────────┘ │
└────────┬─────────────────┘
         │
         ▼
┌────────────────────┐
│   RDS PostgreSQL   │
│                    │
│ SELECT * FROM      │
│   extractions      │
│ WHERE              │
│   status='pending' │
│ ORDER BY           │
│   confidence ASC   │
└────────┬───────────┘
         │ Results
         │
         │ 3. Display in UI
         │ 4. User reviews
         │ 5. POST /extractions/{id}/validate
         │    {action: 'approve', feedback: '...'}
         │
         ▼
┌──────────────────────────┐
│ Lambda: Validation CRUD  │
│ ┌──────────────────────┐ │
│ │ UPDATE extractions   │ │
│ │ SET status='approved'│ │
│ │ Store feedback       │ │
│ │ Trigger graph update │ │
│ └──────────────────────┘ │
└────────┬─────────────────┘
         │
         ▼
┌─────────────────────────┐
│   RDS PostgreSQL        │
│                         │
│ 1. UPDATE extractions   │
│ 2. INSERT INTO AGE graph│
│    (Drug)-[TREATS]->(Disease) │
└─────────────────────────┘
```

## 4. Knowledge Graph Generation Flow

```
┌─────────────────┐
│  EventBridge    │
│  Daily @ 2am    │
└────────┬────────┘
         │ Trigger
         ▼
┌──────────────────────────────┐
│ Lambda: TTL Generator        │
│ ┌──────────────────────────┐ │
│ │ Query AGE graph:         │ │
│ │                          │ │
│ │ MATCH (d:Drug)-[r]->(di:Disease) │
│ │ WHERE r.validated = true │ │
│ │ RETURN d, r, di          │ │
│ └──────────────────────────┘ │
└────────┬─────────────────────┘
         │ Graph data
         │
         ▼
┌──────────────────────────────┐
│ Lambda: TTL Generator (cont) │
│ ┌──────────────────────────┐ │
│ │ For each triple:         │ │
│ │   Generate RDF/TTL:      │ │
│ │                          │ │
│ │   rxnorm:123             │ │
│ │     :treats mesh:456 ;   │ │
│ │     :confidence 0.89 .   │ │
│ └──────────────────────────┘ │
└────────┬─────────────────────┘
         │ TTL content
         ▼
┌────────────────────────┐
│      S3 Bucket         │
│   knowledge-graph/     │
│                        │
│ exports/               │
│   2025-01-15/          │
│     graph.ttl          │
│     metadata.json      │
└────────────────────────┘
```

## 5. Complete End-to-End Flow

```
┌─────────┐
│  START  │
└────┬────┘
     │
     ▼
┌─────────────────────┐
│ 1. PAPER SUBMISSION │
│    User uploads     │
│    API stores       │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 2. ML PROCESSING    │
│    Extract entities │
│    Extract relations│
│    Link to ontology │
│    [~30-60 seconds] │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 3. QUEUE FOR REVIEW │
│    Status: pending  │
│    [Hours to days]  │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 4. HUMAN VALIDATION │
│    Expert reviews   │
│    Approve/Reject   │
│    Provide feedback │
└─────────┬───────────┘
          │
          ├─ Approved ─────┐
          │                │
          │                ▼
          │         ┌──────────────┐
          │         │ 5. GRAPH DB  │
          │         │    Add triple│
          │         │    Update KG │
          │         └──────┬───────┘
          │                │
          │                ▼
          │         ┌──────────────┐
          │         │ 6. TRAINING  │
          │         │    Save as   │
          │         │    positive  │
          │         │    example   │
          │         └──────┬───────┘
          │                │
          ├─ Rejected ─────┤
          │                │
          ▼                ▼
   ┌──────────────────────────┐
   │ 7. CONTINUOUS LEARNING   │
   │    Accumulate feedback   │
   │    Retrain model (manual)│
   │    Improve future runs   │
   └──────────┬───────────────┘
              │
              ▼
       ┌──────────────┐
       │ 8. EXPORT    │
       │    Generate  │
       │    .ttl file │
       │    Daily     │
       └──────┬───────┘
              │
              ▼
       ┌──────────────┐
       │   COMPLETE   │
       │   Knowledge  │
       │   Available  │
       └──────────────┘
```

## Key Latencies

| Stage | Latency | Notes |
|-------|---------|-------|
| Paper submission | <1 second | API call + S3 upload |
| Queue wait | <1 minute | SQS polling interval |
| ML extraction | 30-60 seconds | BioBERT inference |
| Entity linking | 5-10 seconds | API lookups with caching |
| Storage | <1 second | RDS insert |
| **Total automated** | **~1-2 minutes** | From submission to ready for validation |
| Human validation | Hours to days | Depends on validator availability |
| Graph update | <1 second | On approval |
| TTL export | 1-5 minutes | Depends on graph size |

## Error Handling

```
┌─────────────┐
│   Error     │
│   Occurs    │
└──────┬──────┘
       │
       ▼
  ┌─────────┐
  │ Lambda  │
  │ catches │
  │ error   │
  └────┬────┘
       │
       ├── Retryable? ──YES──┐
       │                     │
       NO                    ▼
       │              ┌──────────────┐
       │              │ Return to    │
       │              │ SQS queue    │
       │              │ Retry 1/3    │
       │              └──────────────┘
       │
       ▼
┌──────────────┐
│ Send to DLQ  │
│ (Dead Letter │
│  Queue)      │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ CloudWatch   │
│ Alarm        │
│ Notify admin │
└──────────────┘
```

## Monitoring Points

```
    ┌─────────────────┐
    │   CloudWatch    │
    │   Logs/Metrics  │
    └────────┬────────┘
             │
             │ Collect from:
             │
    ┌────────┼────────────┐
    │        │            │
    ▼        ▼            ▼
┌────────┐ ┌─────┐  ┌─────────┐
│ Lambda │ │ RDS │  │   SQS   │
│  Logs  │ │Slow │  │  Depth  │
│Errors  │ │Query│  │ Age     │
└────────┘ └─────┘  └─────────┘
    │        │            │
    └────────┼────────────┘
             │
             ▼
    ┌─────────────────┐
    │   Dashboard     │
    │   - Papers/day  │
    │   - Error rate  │
    │   - Confidence  │
    │   - Latency     │
    └─────────────────┘
```
