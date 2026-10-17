# OmniDrive AI — Technical Architecture & System Design

> **Engineering Reference: Event-Driven Serverless Media Intelligence & Cloud Storage**

---

## 1. System Overview

**OmniDrive AI** is an event-driven serverless cloud platform that converts static file storage into an active media intelligence workspace.

Instead of routing high-bandwidth binary uploads through traditional servers, clients upload directly to cloud storage using temporary signed URLs. Once a payload lands, an autonomous event mesh immediately triggers parallel serverless pipelines for:
* **Pre-Ingestion Safety Moderation**: Automated quarantine and filtering of non-compliant media.
* **Computer Vision Intelligence**: Autonomous scene recognition and tagging.
* **Document OCR & AI Synthesis**: Text extraction, spatial layout parsing, and executive AI summaries.
* **Adaptive Video Transcoding**: Broadcast-grade multi-bitrate HLS streaming (`1080p`, `720p`, `480p`, `360p`).
* **Zero-Knowledge Encrypted Vault**: Air-gapped vault protected by client-side SHA-256 PIN hashing and progressive brute-force rate-limiting.

---

## 2. Core Cloud Architecture

```mermaid
flowchart TD
    subgraph ClientLayer ["1. Client & Presentation"]
        User["User Browser / React 18 SPA"]
        Vault["Encrypted Vault Enclave"]
    end

    subgraph EdgeAuth ["2. Edge Ingress & Identity"]
        Cognito["Amazon Cognito<br/>(JWT User Pool & Auth)"]
        APIGW["Amazon API Gateway v2<br/>(HTTP API & Route Throttling)"]
        WAF["AWS WAF<br/>(Perimeter Rate Limiting)"]
        CDN["Amazon CloudFront<br/>(Edge CDN & Video Streaming)"]
    end

    subgraph StorageDB ["3. Storage & Metadata"]
        S3Raw["Amazon S3 Raw Bucket<br/>(Direct Presigned PUT Ingest)"]
        S3Proc["Amazon S3 Processed Bucket<br/>(Thumbnails & HLS Streams)"]
        DDB["Amazon DynamoDB<br/>(Single-Table Registry)"]
        KMS["AWS KMS<br/>(Customer Master Key Encryption)"]
    end

    subgraph EventMesh ["4. Event Routing & Queues"]
        EB["Amazon EventBridge<br/>(S3 ObjectCreated Event Bus)"]
        Q_Mod["SQS Moderation Queue"]
        Q_Vid["SQS Video Queue"]
        Q_Vis["SQS Vision Queue"]
        Q_Doc["SQS Document Queue"]
    end

    subgraph ProcessingEngines ["5. AI & Media Processing"]
        L_Presign["Presigned URL Lambda"]
        L_Mod["Safety Gatekeeper (Rekognition)"]
        L_Vis["Vision AI Tagger (Rekognition)"]
        L_Doc["Document Synthesizer (Textract + Claude 3)"]
        MediaConvert["Elemental MediaConvert (HLS Transcoder)"]
    end

    User -->|Authenticate| Cognito
    User -->|Request Upload URL| APIGW
    APIGW --> L_Presign
    L_Presign -->|Signed PUT URL| User
    User -->|Direct Binary Stream| S3Raw
    S3Raw -->|Emit ObjectCreated| EB

    EB --> Q_Mod --> L_Mod
    EB --> Q_Vid --> MediaConvert
    EB --> Q_Vis --> L_Vis
    EB --> Q_Doc --> L_Doc

    L_Mod & L_Vis & L_Doc & MediaConvert -->|Update File Metadata| DDB
    MediaConvert -->|Output HLS Playlists| S3Proc
    S3Proc -->|Deliver Stream| CDN
    CDN --> User
```

---

## 3. Active Cloud Services Matrix

| Service | Architecture Layer | Concrete Implementation in OmniDrive AI |
| :--- | :--- | :--- |
| **Amazon S3** | Ingestion & Storage | Dual-bucket topology separating raw client uploads from processed media assets. S3 Transfer Acceleration enabled for high-bandwidth uplinks. |
| **Amazon DynamoDB** | Database | Single-table schema (`omnidrive-ai-registry-dev`, `OmniDrive_Users`, `OmniDrive_ConsentLogs`) with on-demand capacity and sub-millisecond lookups. |
| **AWS Lambda** | Microservice Compute | Python 3.11 serverless handlers for presigned URL issuance, folder hierarchies, AI enrichment dispatching, and content indexing. |
| **Amazon API Gateway** | Gateway Ingress | Regional HTTP API v2 with native Cognito JWT authorizer, CORS preflight routing, and route token throttling. |
| **Amazon Cognito** | Identity & Auth | Secure User Pool directory, Secure Remote Password (SRP) authentication, short-lived JWT issuance, and session token renewal. |
| **Amazon CloudFront** | Edge CDN Delivery | Low-latency global edge network with Origin Access Control (OAC) for instant playback of adaptive HLS video manifests. |
| **Amazon EventBridge** | Event Routing Bus | Captures S3 `ObjectCreated` events and fans them out to decoupled processing queues. |
| **Amazon SQS** | Asynchronous Queuing | Worker queues (`moderation`, `video`, `vision`, `pdf`) with 14-day retention Dead Letter Queues (DLQs) for resilient fault recovery. |
| **AWS Elemental MediaConvert** | Media Transcoding | Automated 5-tier Adaptive Bitrate (ABR) HLS transcode pipeline generating 1080p, 720p, 480p, 360p, and 240p stream segments. |
| **Amazon Rekognition** | Computer Vision AI | Content moderation scanning (`detect_moderation_labels`) and visual categorization tag generation (`detect_labels`). |
| **Amazon Textract** | Document OCR | Multi-page text extraction, tabular recognition, and bounding-box parsing for PDF and scanned documents. |
| **Amazon Bedrock** | Generative AI | Foundation Model inference (Anthropic Claude 3 Haiku) generating executive document summaries and contextual insights. |
| **AWS KMS** | Cryptographic Security | Customer Master Key (CMK) managing envelope encryption for sensitive assets at rest in storage and database tiers. |
| **AWS WAF** | Perimeter Defense | Web Application Firewall defending API endpoints against automated scrapers, brute force, and OWASP Top 10 exploits. |
| **Amazon CloudWatch** | Observability | Centralized telemetry logging and threshold alarms monitoring Lambda execution errors and SQS DLQ backlogs. |
| **Amazon SNS** | Notification Topic | Multi-subscriber fanout publishing operational updates for completed transcodes, safety flags, and storage quota thresholds. |
| **AWS Systems Manager (SSM)** | Configuration Store | Parameter Store managing centralized application configurations, runtime flags, and model parameters. |

---

## 4. End-to-End Direct Streaming Pipeline

```mermaid
sequenceDiagram
    autonumber
    actor User as User / Browser
    participant Gateway as API Gateway v2
    participant Presign as Presign Lambda
    participant S3Raw as S3 Raw Bucket
    participant EventBus as EventBridge Bus

    User->>Gateway: POST /upload-url (JWT Header)
    Gateway->>Presign: Invoke Handler (Validated Claims)
    Presign->>Presign: Check MIME, Quotas & File Size
    Presign->>S3Raw: Request Presigned PUT URL (300s TTL)
    Presign-->>User: Return Signed PUT URL & file_id
    User->>S3Raw: Direct Binary Stream (XHR / Fetch PUT)
    Note over User,S3Raw: High-throughput transfer bypasses API Gateway entirely
    S3Raw->>EventBus: Emit S3:ObjectCreated Event
```

---

## 5. Security & Rate Limiting Framework

### Perimeter Defense
* **IP Rate Limiting**: Managed Web Application Firewall (WAF) tracks client IPs across a 5-minute sliding window. Any client exceeding **1,000 requests per 5 minutes** is temporarily blocked (HTTP 403 Forbidden).
* **Token-Bucket Throttling**: API Gateway default route enforcement restricts calls to a **1,000 req/sec steady-state rate** and a **500 request burst limit**, preventing microservice overload.
* **Brute-Force Lockouts**: Multi-tiered exponential delays and client-side lockouts defend user accounts and 4-digit Master PIN vaults against automated guessing attacks (15s cooldown at 3 attempts, 60s at 5, and 5 minutes at 8 attempts).

### Compliance & Governance
* **Zero Client Credentials**: No cloud secret keys or IAM credentials are ever bundled in client code.
* **India DPDP Act (2023) Alignment**: Explicit consent logs recorded with timestamps, IP hash, and audit trail in an immutable table before workspace activation.

---

*Copyright © 2026 OmniDrive AI. All rights reserved.*
