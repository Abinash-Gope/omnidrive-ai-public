# OmniDrive AI — REST API Interface Specification

> **Public API Contracts, Authentication Specifications & Data Models**

All API requests communicate over HTTPS via the Cloud API Gateway v2 regional endpoint. Authenticated endpoints require an `Authorization: Bearer <ID_TOKEN>` header containing a verified JWT.

---

## 1. Authentication & Security Headers

| Header | Type | Description |
| :--- | :--- | :--- |
| `Authorization` | `string` | `Bearer <JWT_TOKEN>` issued by the Cloud Identity Provider. |
| `Content-Type` | `string` | `application/json` for standard payload bodies. |
| `X-Client-Version` | `string` | Optional client build version identifier. |

---

## 2. API Endpoints

### 2.1 Presigned Upload URL Generator
Requests authorized temporary presigned S3 PUT URLs for direct client-to-cloud streaming.

* **Route:** `POST /upload-url`
* **Auth Required:** Yes (`Bearer <token>`)

#### Request Body (Single Upload Initiation)
```json
{
  "action": "single_upload",
  "file_name": "quarterly_financial_report.pdf",
  "file_type": "application/pdf",
  "file_size": 14502800,
  "folder_id": "fld_92b8d41e"
}
```

#### Response (`200 OK`)
```json
{
  "file_id": "f83a1b02-5c91-4e78-bc02-8610a8c2f104",
  "upload_url": "https://omnidrive-ai-raw-dev.s3.accelerate.amazonaws.com/raw/user_sub/f83a1b02/quarterly_report.pdf?X-Amz-Algorithm=...",
  "s3_key": "raw/user_sub/f83a1b02-5c91-4e78-bc02-8610a8c2f104/quarterly_financial_report.pdf",
  "expires_in": 7200,
  "status": "PENDING_UPLOAD"
}
```

---

### 2.2 Multipart Resumable Chunk Initiation
Initiates a high-speed multi-chunk transfer session for files up to 5 TB.

* **Route:** `POST /upload-url`
* **Auth Required:** Yes (`Bearer <token>`)

#### Request Body
```json
{
  "action": "initiate_multipart",
  "file_name": "4k_cinematic_demo.mp4",
  "file_type": "video/mp4",
  "file_size": 2411724800,
  "parts_count": 240,
  "folder_id": null
}
```

#### Response (`200 OK`)
```json
{
  "file_id": "90e1fc83-b712-4211-9e20-7f2894ca1391",
  "upload_id": "2.XkLmNopQRstUvWxYz...",
  "s3_key": "raw/user_sub/90e1fc83/4k_cinematic_demo.mp4",
  "chunk_size": 10485760
}
```

---

### 2.3 Files Registry Listing & Search
Queries the single-table DynamoDB registry with cursor pagination and category filtering.

* **Route:** `GET /files`
* **Auth Required:** Yes (`Bearer <token>`)
* **Query Parameters:**
  * `folder_id` *(optional, string)*: Filter files inside a specific directory.
  * `category` *(optional, string)*: Filter by media type (`images`, `videos`, `documents`, `audio`).
  * `search` *(optional, string)*: Keyword search across file names and AI-generated vision tags.
  * `limit` *(optional, number)*: Number of items to return (default: `50`).

#### Response (`200 OK`)
```json
{
  "items": [
    {
      "file_id": "f83a1b02-5c91-4e78-bc02-8610a8c2f104",
      "file_name": "quarterly_financial_report.pdf",
      "file_size": 14502800,
      "file_type": "application/pdf",
      "status": "READY",
      "created_at": "2026-10-15T10:14:22Z",
      "folder_id": "fld_92b8d41e",
      "vision_tags": ["finance", "table", "summary", "audit"],
      "ai_summary": "Executive briefing covering Q3 revenue expansion, operating margin metrics, and cash flow projections.",
      "cdn_url": "https://d3by850sf4vvuz.cloudfront.net/processed/user_sub/f83a1b02/quarterly_financial_report.pdf"
    }
  ],
  "next_token": null
}
```

---

### 2.4 Folder Management
Manages multi-tier organizational hierarchies.

* **Route:** `POST /create-folder`
* **Auth Required:** Yes (`Bearer <token>`)

#### Request Body
```json
{
  "name": "Confidential Architecture Reviews",
  "parent_folder_id": null,
  "is_vault": false
}
```

#### Response (`201 Created`)
```json
{
  "folder_id": "fld_92b8d41e",
  "name": "Confidential Architecture Reviews",
  "parent_folder_id": null,
  "created_at": "2026-10-16T08:30:00Z"
}
```

---

### 2.5 DPDP Act 2023 Statutory Consent
Logs explicit statutory data processing and storage consent with immutable audit timestamps.

* **Route:** `POST /consent`
* **Auth Required:** Yes (`Bearer <token>`)

#### Request Body
```json
{
  "dpdp_consent_status": "GRANTED",
  "dpdp_notice_version": "DPDP-2023-V1",
  "dpdp_consent_timestamp": "2026-10-17T06:45:00Z"
}
```

#### Response (`200 OK`)
```json
{
  "success": true,
  "message": "Statutory audit record logged successfully."
}
```

---

## 3. Standard HTTP Status Codes

| Code | Status | Meaning |
| :--- | :--- | :--- |
| `200` | OK | Successful request with response body. |
| `201` | Created | Resource successfully created. |
| `400` | Bad Request | Missing required parameters or invalid MIME format. |
| `401` | Unauthorized | Missing or expired JWT identity token. |
| `403` | Forbidden | Insufficient permissions or IP blocked by WAF rate limiter. |
| `404` | Not Found | Requested file or folder does not exist. |
| `429` | Too Many Requests | Rate limit throttled by API Gateway or perimeter firewall. |
| `500` | Server Error | Internal microservice fault (alerted to CloudWatch). |

---

*Copyright © 2026 OmniDrive AI. All rights reserved.*
