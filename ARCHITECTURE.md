# ARCHITECTURE — Long-Running Downloads (Proxy-safe)

## 1. Problem
This service simulates downloads that may take ~10–120+ seconds. When deployed behind a reverse proxy (Cloudflare/nginx/ALB), long-running HTTP requests can exceed proxy/client timeouts and result in 504 errors. We need a design that avoids long-lived requests, provides progress, and still works if the user closes the tab.

## 2. Chosen Approach (Async Jobs + Polling)
We handle downloads asynchronously:
- Client sends a short request to initiate a download.
- API immediately returns a `jobId`.
- A background worker performs the long download + packaging.
- Client polls a status endpoint until the job finishes.
- When complete, the result is downloaded from storage (S3/MinIO), preferably via a pre-signed URL.

This keeps all user-facing requests short and proxy-safe.

## 3. Architecture Diagram

```mermaid
flowchart LR
  U[Browser / Frontend] -->|POST /v1/download/initiate| API[Download API]
  API -->|enqueue job| Q[(Queue)]
  W[Worker] -->|download & package| DL[Download Processing]
  W -->|upload result| S3[(S3/MinIO: downloads bucket)]
  W -->|update status| JS[(Job Store)]
  U -->|GET /v1/download/status/{jobId} (poll)| API
  API --> JS
  U -->|GET /v1/download/{jobId} OR pre-signed URL| S3
