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

%% Initialization
A[Initialize DuckLake<br/>Create tables from Parquet schema]

%% Upload + Lambda
B[Upload Parquet to S3 Landing]
C[Lambda Triggered]
D{Virus Scan Result}

E[Delete or Quarantine File]
F[Copy File to Client Bucket]

%% Control Plane Starts AFTER clean copy
G[Insert Job as PENDING in PostgreSQL]
H[Query PostgreSQL for Active Pod]
I{Active Pod Found?}

J[Store Pod Check Result YES]
K[Store Pod Check Result NO]

%% YES path
L[Push Job to Worker Redis Stream]

%% NO path
M[Insert Pod Record as STARTING in PostgreSQL]
N[Push Message to Pod Manager Stream]
O[Push Job to Worker Redis Stream]

%% Pod Manager
P[Pod Manager Listens to Redis]
Q[Create K8s Worker Pod<br/>4GB RAM - 1 vCPU]
R[Update Pod Status to RUNNING in PostgreSQL]

%% Worker
S[Worker Receives Job]
T[Update Job Status to RUNNING]
U[Copy Parquet Data to DuckLake]
V[Update Metric Column<br/>value = value * 5]
W[Update Job Status to COMPLETED]
X[Update Pod Status to TERMINATED]
Y[Graceful Pod Shutdown]

%% Flow Connections
A --> B
B --> C
C --> D

D -->|Virus Found| E
D -->|Clean File| F

F --> G
G --> H
H --> I

I -->|Yes| J
I -->|No| K

J --> L
K --> M

M --> N
N --> P
P --> Q
Q --> R
R --> O

L --> S
O --> S

S --> T
T --> U
U --> V
V --> W
W --> X
X --> Y
```
## Detailed Sequence Diagram 

```mermaid
sequenceDiagram

participant User
participant S3_Landing as S3 Landing Bucket
participant Lambda
participant S3_Client as S3 Client Bucket
participant Postgres
participant Redis
participant PodManager
participant K8s
participant Worker
participant DuckLake

%% Upload Flow
User->>S3_Landing: Upload Parquet file
S3_Landing->>Lambda: Trigger event
Lambda->>Lambda: Scan for virus

alt Virus Found
    Lambda->>S3_Landing: Delete / Quarantine file
else Clean File
    Lambda->>S3_Client: Copy file to client bucket

    %% Control Plane Starts
    Lambda->>Postgres: Insert Job (PENDING)
    Lambda->>Postgres: Check Active Pod

    alt Active Pod Exists
        Lambda->>Postgres: Store pod_check = YES
        Lambda->>Redis: Push job to worker stream
    else No Active Pod
        Lambda->>Postgres: Store pod_check = NO
        Lambda->>Postgres: Insert Pod (STARTING)
        Lambda->>Redis: Push message to pod-manager stream
        Lambda->>Redis: Push job to worker stream
    end
end

%% Pod Manager Flow
Redis->>PodManager: Pod creation message
PodManager->>K8s: Create worker pod (4GB RAM, 1 vCPU)
K8s->>Postgres: Update Pod Status = RUNNING

%% Worker Flow
Redis->>Worker: Deliver ingestion job
Worker->>Postgres: Update Job = RUNNING
Worker->>DuckLake: Copy Parquet data
Worker->>DuckLake: Update metric column (value * 5)
Worker->>Postgres: Update Job = COMPLETED
Worker->>Postgres: Update Pod = TERMINATED
Worker->>K8s: Graceful shutdown
```
