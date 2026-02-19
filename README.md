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

## Workflow

```mermaid
flowchart TD
  A[S3 Landing Bucket] --> B[Lambda: Virus Scan]
  B -->|infected| Q[Quarantine / Delete]
  B -->|clean| C[S3 Trusted Client Bucket]
  C --> R[Redis Streams: ingestion_requests]
  R --> PM[Pod Manager]
  PM --> W[K8s Worker Pod]
  W --> D[DuckLake]
  W --> U[Metric Update]
  W --> X[Shutdown]
