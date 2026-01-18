# Design Document: Async URL Summarization Orchestrator

## 1. Overview

This document describes the design of an **event-driven job orchestration system** for **URL article summarization**. The system allows users to submit one or multiple URLs, processes them asynchronously, generates summaries, stores results, and sends completion notifications via **Discord**.

The system is designed as an **internal automation platform** with a strong focus on:

* Asynchronous execution
* Clear separation of responsibilities (API / worker / storage)
* Reliability and traceability
* Ease of future extension (e.g., transcription jobs, K8s deployment)

---

## 2. Goals

* Allow users to submit long-running URL summarization jobs without blocking
* Automatically notify users when jobs complete (success or failure)
* Persist job state and results for later inspection
* Provide a clean, extensible system architecture similar to real-world platform systems

## 3. Non-Goals (MVP)

* No public-facing service (private/internal use only)
* No complex authentication or multi-tenant support
* No advanced Web UI (API + Discord notification only)
* No optimization for very large-scale traffic

---

## 4. High-Level Architecture

```
flowchart LR
  U[User / Client] -->|POST /jobs/url-summary| API[FastAPI Job API]
  API -->|Create job record| DB[(PostgreSQL)]
  API -->|Enqueue job_id| Q[(Redis Queue)]
  Q --> W[Worker: URL Summarizer]
  W -->|Update status + store result| DB
  W -->|Send notification| D[Discord Webhook]
  API -->|GET /jobs/{id}| U
```

### Core Idea

* **API** handles short-lived requests and job management
* **Worker** executes long-running summarization tasks
* **Queue** decouples request handling from execution
* **Database** serves as the system memory
* **Discord** provides user-facing notifications

---

## 5. Components

### 5.1 Job API Service (FastAPI)

**Responsibilities**:

* Accept job submissions
* Validate input
* Create and persist job metadata
* Enqueue jobs for processing
* Expose job status and results

#### Endpoints (MVP)

* `POST /jobs/url-summary`

  * Input:

    ```json
    {
      "urls": ["https://example.com/article"],
      "summary_style": "concise",
      "max_length": 300,
      "language": "en"
    }
    ```
  * Output:

    ```json
    { "job_id": "uuid" }
    ```

* `GET /jobs/{job_id}`

  * Returns job metadata and current status

* `GET /jobs/{job_id}/result`

  * Returns summarized text when job is completed

#### Internal Behavior

* Generate a unique `job_id`
* Set initial status to `PENDING`
* Persist request payload
* Push `job_id` into Redis queue

---

### 5.2 Database (PostgreSQL)

#### Table: jobs

| Column          | Type      | Description                            |
| --------------- | --------- | -------------------------------------- |
| job_id          | UUID      | Primary key                            |
| job_type        | TEXT      | `URL_SUMMARY`                          |
| status          | TEXT      | `PENDING / RUNNING / SUCCESS / FAILED` |
| request_payload | JSONB     | Original request                       |
| result_text     | TEXT      | Final summary (MVP)                    |
| error_message   | TEXT      | Failure reason                         |
| created_at      | TIMESTAMP | Job creation time                      |
| started_at      | TIMESTAMP | Execution start time                   |
| finished_at     | TIMESTAMP | Execution end time                     |
| attempts        | INT       | Retry counter                          |

---

### 5.3 Queue (Redis)

* Used as a **job dispatch mechanism**
* Stores `job_id` references only (not payloads)
* Enables loose coupling between API and workers

---

### 5.4 Worker Service (URL Summarizer)

**Responsibilities**:

* Fetch job from queue
* Execute summarization pipeline
* Update job status and results
* Trigger Discord notification

#### Worker Lifecycle

1. Pop `job_id` from Redis
2. Load job metadata from DB
3. Set status to `RUNNING`
4. Execute summarization pipeline
5. Store summary text
6. Update final status
7. Send Discord notification

---

## 6. URL Summarization Pipeline

### Input

* One or more article URLs

### Output

* A combined textual summary

### Steps

1. **Fetch Content**

   * HTTP GET with timeout
   * Reject private IP ranges (basic SSRF protection)

2. **Extract Main Text**

   * Remove navigation, ads, boilerplate
   * Normalize whitespace

3. **Preprocess**

   * Truncate excessively long articles
   * Limit number of URLs per job

4. **Summarization**

   * MVP: LLM API or lightweight summarizer
   * Configurable length/style

5. **Result Storage**

   * Store final summary text in DB

6. **Notification**

   * Send Discord message with preview and job ID

---

## 7. Job State Machine

```
PENDING → RUNNING → SUCCESS
                 → FAILED
```

* State transitions are persisted
* All failures must result in a terminal state

---

## 8. Discord Notification Design

### Delivery Method

* Discord Webhook (outbound HTTP)

### Message Content

* Job status (success/failure)
* Job ID
* Execution duration
* Summary preview (first N characters)
* Link to `GET /jobs/{job_id}/result`

---

## 9. Error Handling & Retries

* Fetch errors → mark job as FAILED with error message
* Summarization failure → FAILED
* Discord notification failure → log error (no retry in MVP)
* Worker crash → job remains RUNNING (manual inspection in MVP)

---

## 10. Security Considerations (MVP)

* Private/internal deployment only
* API token authentication (single shared token)
* URL allowlist / private IP blocking
* Request size and URL count limits
* No arbitrary command execution

---

## 11. Observability (MVP)

* Structured logs with job_id
* Status transitions recorded in DB
* Discord messages include job_id for traceability

---

## 12. Deployment Plan

### MVP

* Local development using Docker Compose
* Services:

  * api
  * worker
  * redis
  * postgres

### Future (Out of Scope)

* Kubernetes deployment
* Horizontal worker scaling
* Artifact storage (S3/GCS)
* Web UI

---

## 13. Future Extensions

* Add transcription job type (audio/video)
* Add Discord slash command as submission interface
* Add retry policies and dead-letter queue
* Add metrics and dashboards
* Multi-job orchestration and dependencies

---

## 14. Summary

This design provides a realistic, production-inspired foundation for an asynchronous automation system. It emphasizes correct system boundaries, reliability, and extensibility while keeping the MVP scope focused and achievable.
