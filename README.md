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
</> **Bash**

# Start Redis
docker run -p 6379:6379 redis:7

# Run worker locally
cd worker
pip install -r requirements.txt
python worker.py

# Publish test job
redis-cli XADD ingestion_requests * \
  job_id abc123 \
  client_id clientA \
  bucket client-bucket \
  key clientA/incoming/test.parquet

---
## Repo Structure

lambda/         # scan + routing
pod-manager/    # ensures workers exist
worker/         # ingestion
k8s/            # manifests
scripts/        # deploy / run helpers

---
