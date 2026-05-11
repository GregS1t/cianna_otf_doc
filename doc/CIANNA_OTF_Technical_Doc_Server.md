# CIANNA OTF — Server Technical Documentation

---

## 1. Introduction

CIANNA_OTF is a service designed to execute **on-the-fly (OTF) batch inference**
of astronomical FITS images using the **CIANNA** framework.

The system follows a **client / server architecture**:
- the **client** prepares and submits jobs,
- the **server** persists, batches, executes, and exposes results.

This document targets:
- Developers onboarding on the project,
- Contributors extending models, parameters, or batching logic,
- Operators deploying or maintaining the service.

This documentation is **code-driven** and reflects the current server
implementation.

---

## 2. Scope

### Client (see separate documentation)
- Build UWS-compliant XML job descriptions
- Submit XML + FITS files
- Poll job status
- Download results

### Server (this documentation)
- Receive and validate jobs
- Persist job metadata and state
- Batch compatible jobs
- Execute inference using CIANNA
- Post-process outputs
- Expose job status and results via HTTP

---

### IVOA Compliance Scope

| Standard | Status | Notes |
|----------|--------|-------|
| UWS | Partial | Job lifecycle, phases, XML representation |
| VOTable | Full | Result delivery format — generated via astropy |
| PROV | Level 1 | Structured provenance metadata embedded in UWS XML |

---

## 3. Server Architecture Overview

### 3.1 High-Level Flow

```
Client
  |
  | POST /jobs/  (XML + FITS)
  v
Flask API  (app/server.py)
  |
  |-- JobStore      (core/job_logger.py)         authoritative state
  |-- Batcher       (core/scheduler/batcher.py)  grouping by model/quant
  |-- Worker Threads
        |
        v
  Inference Pipeline  (core/pipeline/)
        |
        v
  Results on disk  (VOTable .vot)
```

### 3.2 Main Components

| Component | File | Role |
|-----------|------|------|
| HTTP API | `app/server.py` | Routes, validation, worker startup |
| Job persistence | `core/job_logger.py` | JobStore: phases, timestamps, metadata |
| Batching | `core/scheduler/batcher.py` | Group jobs by model / quantization |
| Model buffer | `core/model_job_buffer.py` | Per-model queue with deduplication |
| Orchestration | `core/pipeline/orchestrator.py` | Batch execution |
| Preprocessing | `core/pipeline/preprocessing.py` | FITS → patches |
| CIANNA runtime | `core/pipeline/cianna_runtime.py` | Forward pass |
| Postprocessing | `core/pipeline/postprocessing.py` | YOLO decoding, NMS |
| Output | `core/pipeline/votable_writer.py` | VOTable generation |
| Model catalog | `core/models/model_catalog.py` | In-memory catalog loader |
| Configuration | `core/config.py` | Paths, policies, env resolution |
| CIANNA loader | `core/cianna_bootstrap.py` | Lazy load of CIANNA shared library |

---

## 4. Configuration

### 4.1 Resolution Order

Machine-specific values are provided via environment variables (`.env`).
The TOML file contains only portable, shared configuration.

**CIANNA library path:**
1. `CIANNA_PATH` (environment variable — required)
2. `[cianna].path` in TOML (deprecated fallback)

**Data root:**
1. `CIANNA_SERVER_DATA_ROOT` (environment variable)
2. `[paths].data_root` in TOML
3. Default: `<project_root>/../server_data`

### 4.2 Key Policy Parameters (TOML `[policy]`)

| Key | Default | Description |
|-----|---------|-------------|
| `max_jobs` | 10 | Maximum jobs in the batcher queue |
| `max_workers` | 2 | Number of worker threads |
| `min_jobs` | 2 | Minimum batch size before forcing flush |
| `max_batch` | 8 | Maximum batch size |
| `max_wait_s` | 2.0 | Maximum wait before forcing a batch |
| `round_robin` | true | Fair scheduling between model queues |

Configuration is loaded once at startup and is immutable at runtime.

---

## 5. Job Model and Persistence

### 5.1 Job Identifier

Each job is identified by a **UUID v4** generated server-side at submission.

The job ID is used as:
- job directory name under `JOBS_ROOT/`,
- `JobStore` key,
- result file suffix (`net0_rts_<jobId>.vot`).

### 5.2 Job Phases — IVOA UWS

Job lifecycle follows the [IVOA UWS](https://www.ivoa.net/documents/UWS/) pattern.

```
PENDING → QUEUED → EXECUTING → COMPLETED
                             → ERROR
                 → ABORTED
```

Phase transitions are explicit, monotonic, and persisted.

Each transition is reflected in:
1. `job_log.json` (JobStore) — **authoritative**
2. `requestJob_uws.xml` — derived representation, patched atomically

### 5.3 JobStore

The JobStore (`core/job_logger.py`):
- is thread-safe,
- uses atomic JSON writes,
- maintains rotating backups,
- stores per-job metadata: model, quantization, ownerId, timestamps, result path.

The UWS XML is derived from JobStore state and must not be treated as a state
holder.

---

## 6. Filesystem Layout

```
SERVER_DATA_ROOT/          # CIANNA_SERVER_DATA_ROOT in .env — not versioned
├── runtime/
│   └── JOBS/
│       └── <jobId>/
│           ├── requestJob_uws.xml        canonical UWS representation
│           ├── requestJob_uws.input.xml  copy of client-submitted XML
│           ├── image.fits
│           └── fwd_res/
│               └── net0_rts_<jobId>.vot
├── logs/
│   ├── job_log.json
│   └── log_backups/
└── models/
    ├── CIANNA_models.xml   updated at each client connection
    └── <model .dat files>
```

Jobs are **not moved** between directories. Phases are logical states stored
in `job_log.json`.

---

## 7. HTTP API

### 7.1 Submit a Job

**`POST /jobs/`**

- Content: `multipart/form-data` with fields `xml` and `fits`
- Validates file extensions (`.xml`, `.fits`) and upload size
- Checks batcher capacity (`max_jobs`)
- Generates a UUID, saves files, initializes UWS XML with provenance metadata
- Enqueues to the batcher
- Returns `HTTP 303` with `Location: /jobs/<jobId>`

### 7.2 Query Job Status

**`GET /jobs/<jobId>`**

- `Accept: application/json` → JSON with current phase
- Otherwise → UWS XML (`requestJob_uws.xml`)

### 7.3 Change Job Phase

**`POST /jobs/<jobId>/phase`**

- Parameter: `PHASE=ABORT`
- Marks job as ABORTED in JobStore and UWS XML
- Attempts to remove job from batcher (best-effort)
- Returns `HTTP 303`

### 7.4 Retrieve Results

**`GET /jobs/<jobId>/results`**

- Returns `net0_rts_<jobId>.vot` as `application/x-votable+xml`
- Returns 404 if result not found or job not COMPLETED

### 7.5 Abort a Job (alternate route)

**`POST /jobs/<jobId>/abort`**

- JSON response: `{"job_id": ..., "aborted": true}`
- Attempts batcher removal and marks ABORTED

### 7.6 Models Registry

**`GET /model-files/<filename>`**

- Serves files from `SERVER_DATA_ROOT/models/`
- Used by clients to fetch `CIANNA_models.xml` at connection

---

## 8. Batching and Workers

### 8.1 Batching Strategy

Jobs are grouped by **(model identifier, quantization mode)**.

A batch is flushed when:
- batch size ≥ `MAX_BATCH`, or
- oldest pending job has waited ≥ `MAX_WAIT_S`.

The batcher supports round-robin scheduling between model queues to avoid
starvation.

### 8.2 Model Job Buffer

`core/model_job_buffer.py` provides per-model queues with:
- deduplication by `process_id` (idempotent enqueue),
- atomic snapshot-and-lock via `try_snapshot_batch()`,
- per-model locks to serialize `batch_prediction`.

### 8.3 Workers

Worker threads are started at server boot (`startWorkers(MAX_WORKERS)`).
Each worker loops on `batcher.pop_ready_batch()` and calls the inference
pipeline. With a single GPU, workers effectively execute sequentially.

---

## 9. Inference Pipeline

### 9.1 Pipeline Stages

1. Model loading (once per batch, via `cianna_bootstrap.get_cnn()`)
2. Parameter resolution: catalog → server config → job request (merge)
3. FITS preprocessing and patch extraction
4. Single batched forward pass (`cianna_runtime.py`)
5. Postprocessing: YOLO decoding, NMS
6. VOTable generation (`votable_writer.py`)

### 9.2 Failure Isolation

| Stage | Impact |
|-------|--------|
| Preprocessing | Job → ERROR |
| Forward pass | Batch → ERROR (all jobs in batch) |
| Postprocessing | Job → ERROR |

### 9.3 Model Catalog

The model catalog is loaded once at startup via `initModelCatalog()`.
It is read from `SERVER_DATA_ROOT/models/CIANNA_models.xml` and stored
in memory. All runtime model lookups use `getModelInfo()` from
`core/models/model_catalog.py`. UWS-style catalogs that do not enumerate
multiple models store parameters under the special key `__uws__`.

---

## 10. Provenance — IVOA PROV (Level 1)

> **Status**: Level 1 implemented. Level 2 is the long-term objective.
> Reference: https://www.ivoa.net/documents/ProvenanceDM/

Provenance metadata is embedded in `requestJob_uws.xml` at job creation.

### Tracked Elements

| Namespace | Field | Description |
|-----------|-------|-------------|
| `srv` | `ServerJobUUID` | Server-generated job UUID |
| `srv` | `ServerReceivedAt` | UTC timestamp of reception |
| `srv` | `HostName` | Server hostname |
| `srv` | `OS` / `KernelVersion` | Operating system info |
| `input` | `FitsFilename` | Original FITS filename |
| `input` | `FitsSizeBytes` | File size |
| `input` | `FitsChecksumSHA256` | SHA-256 hash of the input file |
| `sw` | `PythonVersion` | Python version |
| `sw` | `GitCommit` | Git commit hash at runtime |
| `sw` | `pkg.*` | Key package versions |
| `hw` | `GPUModel` / `GPUUUID` | GPU identity |
| `hw` | `CUDASupportedVersion` | CUDA version |

This enables reconstruction of execution timeline, data lineage, and
logical execution context.

**Non-goals (current version):**
- standalone PROV documents,
- PROV-XML / PROV-JSON exports,
- full activity/entity graphs.

---

## 11. Deployment

### Prerequisites

- Python 3.11+
- CIANNA framework compiled (see
  [installation instructions](https://github.com/Deyht/CIANNA/wiki/2%29-Installation-instructions))
- NVIDIA GPU recommended

### Install

```bash
cd C_OTF_SERVER/
pip install -e .
```

### Configure

```bash
cp .env.example .env
# Edit .env: set CIANNA_PATH and CIANNA_SERVER_DATA_ROOT
```

### Run

```bash
./run.sh
```

The server listens by default on `0.0.0.0:5000`.

---
