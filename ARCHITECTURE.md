# ARCHITECTURE — Long-Running Downloads (Proxy-safe)

## 1. Problem
This service simulates downloads that may take ~10–120+ seconds. When deployed behind a reverse proxy (Cloudflare/nginx/ALB), long-running HTTP requests can exceed proxy or client timeouts and result in 504 errors. We need a design that avoids long-lived requests, provides progress visibility, and continues to work even if the user closes the browser tab.

## 2. Chosen Approach (Async Jobs + Polling)
Downloads are handled asynchronously:

- The client sends a short request to initiate a download.
- The API immediately returns a `jobId`.
- A background worker performs the long-running download and packaging.
- The client periodically polls a status endpoint to check progress.
- Once complete, the result is downloaded from storage (S3/MinIO), preferably using a pre-signed URL.

This approach ensures that all user-facing requests are short and safe behind reverse proxies.

## 3. Architecture Diagram

```mermaid
flowchart LR
  U[Browser / Frontend] -->|POST /v1/download/initiate| API[Download API]
  API -->|enqueue job| Q[(Queue)]
  Q --> W[Worker]
  W -->|download & package| DL[Download Processing]
  W -->|upload result| S3[(S3/MinIO - downloads bucket)]
  W -->|update status| JS[(Job Store)]
  U -->|GET /v1/download/status (poll)| API
  API --> JS
  U -->|GET result or pre-signed URL| S3
