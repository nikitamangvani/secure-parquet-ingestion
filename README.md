# secure-parquet-ingestion
Event-driven Parquet ingestion pipeline with virus scanning, Redis Streams Orchestration, Kubernetes workers and Duck lake Storage.

# 🦆 Secure Parquet → DuckLake Pipeline

![Python](https://img.shields.io/badge/python-3.10%2B-blue)
![AWS](https://img.shields.io/badge/AWS-S3%20%7C%20Lambda-orange)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Workers-blue)
![Redis](https://img.shields.io/badge/Redis-Streams-red)

## Introduction

This project is a simple, reliable pipeline for safely moving Parquet files into DuckLake.  
Files uploaded to S3 are automatically scanned for viruses, queued through Redis, and processed in Kubernetes worker pods.  
Once ingested, the data is stored in DuckLake, with an optional step to update a metric column.

**Flow:**  
`S3 Landing → Lambda Virus Scan → Redis Streams → K8s Worker → DuckLake`

---

## Features

- Automatically scans files for viruses  
- Keeps landing, trusted, and quarantine buckets separate  
- Uses Redis Streams to manage ingestion jobs reliably  
- Runs worker pods on Kubernetes with set resources (4Gi RAM, 1 vCPU)  
- Creates tables in DuckLake automatically and updates metrics  
- Designed to handle duplicates and retries safely  

---

## Quick Start

```bash
1. Start Redis
docker run -p 6379:6379 redis:7

2. Run worker locally
cd worker
pip install -r requirements.txt
python worker.py

3. Publish test job
redis-cli XADD ingestion_requests * \
  job_id abc123 \
  client_id clientA \
  bucket client-bucket \
  key clientA/incoming/test.parquet

```

---

## Repo Structure

lambda/         # scan + routing
pod-manager/    # ensures workers exist
worker/         # ingestion
k8s/            # manifests
scripts/        # deploy / run helpers

---
## ☁️ Deployment

### AWS Setup

**1. Create S3 buckets**

- Landing bucket  
- Trusted client bucket  
- Quarantine bucket  

**2. Configure S3 Event Trigger**

- Event: `ObjectCreated`
- Prefix: `uploads/`
- Target: Lambda function

**3. Deploy Lambda**

- Include ClamAV layer for virus scanning  
- Configure required environment variables  
- Ensure IAM permissions:

  - `s3:GetObject` on landing bucket  
  - `s3:PutObject` on trusted / quarantine buckets  
  - `s3:DeleteObject` on landing bucket  

---

### Kubernetes Setup

- Deploy Redis (or use AWS ElastiCache)  
- Deploy Pod Manager as a Kubernetes Deployment  
- Ensure Pod Manager has RBAC permission to create worker pods  

**Worker Pod Resources**
- Memory: `4Gi`
- CPU: `1 vCPU`

---

## 🧯 Troubleshooting

### Lambda triggers but no job runs

- Check Redis connectivity (VPC / security groups)  
- Verify Redis stream exists  
- Check Lambda logs for publish errors  

---

### Worker pod starts and exits immediately

- Redis URL or authentication may be incorrect  
- Consumer group not created  
- Worker idle timeout set too low  

---

### Duplicate ingestion happens

- Ensure `job_id` is deterministic  
- Make sure job state store is enabled  
- Worker should verify job status before processing

---
### Detailed Workflow

```mermaid
flowchart TD
    %% Step 1: Initialization
    Start[Initialize DuckLake] --> Schema[Define Tables and Columns]
    Schema --> S3[(S3: Drop Parquet File)]

    %% Step 2: Security & Lambda
    S3 --> Lambda[Trigger Lambda]
    Lambda --> Scan{Virus Scan?}
    
    Scan -- Virus Found --> Delete[Delete File]
    Scan -- Clean --> Copy[Copy to Client Bucket]

    %% Step 2.5: Redis Logic
    Copy --> CheckPod{Pod Running?}
    
    CheckPod -- Yes --> PushWorker[Push to Worker Redis Stream]
    
    CheckPod -- No --> PushManager[Push to Pod Manager Stream]
    PushManager --> PushWorker

    %% Step 3: Pod Manager
    subgraph K8s_Control [K8s Cluster Management]
        PushManager -.-> Poll[Pod Manager Polls Redis]
        Poll --> CreatePod[Create Worker Pod: 1 vCPU / 4GB RAM]
    end

    %% Step 4: Ingestion Job
    PushWorker -.-> Worker[Worker Receives Message]
    CreatePod --> Worker
    
    subgraph Worker_Job [Ingestion Execution]
        Worker --> Load[Copy Parquet to DuckLake]
        Load --> Calc[Update Metric: Value * 5]
        Calc --> Finish([Graceful Shutdown])
    end

    %% Styles
    style Scan fill:#fff4dd,stroke:#d4a017
    style CheckPod fill:#fff4dd,stroke:#d4a017
    style Delete fill:#ff9999,stroke:#b91c1c
    style Finish fill:#dcfce7,stroke:#166534
```
