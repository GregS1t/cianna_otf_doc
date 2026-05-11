# CIANNA OTF — Client Technical Documentation

---

## 1. Introduction

This document describes the **client-side architecture, usage modes, and
contracts** of the CIANNA_OTF system.

This document targets:
- Developers onboarding on the project,
- Contributors extending models, parameters, or batching logic,
- Operators deploying or maintaining the service.

The client interacts with a CIANNA_OTF server implementing:
- IVOA UWS (asynchronous job execution),
- IVOA VOTable (scientific results),
- IVOA PROV level 1 (server-side provenance).

This documentation is **code-driven** and reflects the current client
implementation.

---

## 2. Client Usage Modes

Three complementary usage modes share the same underlying runtime API.

### 2.1 Graphical User Interface (GUI) — `app/gui_c_otf.py`

- Implemented in **PyQt6**
- Intended for:
  - visual inspection of FITS images,
  - interactive ROI definition,
  - model and quantization selection,
  - exploratory usage and demonstrations
- **Under active development**
- Not required for production or automation

### 2.2 Terminal Dashboard (TTY) — `app/db_c_otf.py`

- Runs entirely in a terminal
- Developer-oriented
- Allows:
  - monitoring locally submitted jobs,
  - visualizing UWS phases,
  - polling server-side job XML,
  - downloading results,
  - emulating multiple jobs
- **Under active development**

### 2.3 Emulator / CLI — `app/emul_c_otf.py`

- No GUI, no interactive dashboard
- Scriptable — intended for automation, benchmarks, and stress tests
- Uses the same runtime API as GUI and TTY

```bash
python -m app.emul_c_otf --nb 5 --interval 1.0
python -m app.emul_c_otf --nb 10 --abort-after 30 --abort-prob 0.5
```

---

## 3. Client Architecture Overview

```
+----------------------------------+
|    GUI / TTY / CLI               |
|    (app/gui_c_otf.py             |
|     app/db_c_otf.py              |
|     app/emul_c_otf.py)           |
+---------------+------------------+
                |
                v
+----------------------------------+
|  OTFClient  (core/otf_client.py) |
|  UI-free facade — recommended    |
|  entry point for new interfaces  |
+---------------+------------------+
                |
                v
+----------------------------------+
|  Client Runtime API              |
|  (core/client_runtime.py)        |
|  - connectserver / disconnect    |
|  - submitjobfromxml              |
|  - runSingleJob / emulateJobs    |
|  - refreshmodelscatalog          |
|  - abortjob / downloadjobresult  |
+---------------+------------------+
                |
                v
+----------------------------------+
|  Server  (UWS-compliant)         |
+----------------------------------+
```

The client is **intentionally thin**:
- no execution logic,
- no batching,
- no authoritative state.

The server is the **single source of truth**.

---

## 4. Client Runtime Layer

### 4.1 Main Modules

| Module | Responsibility |
|--------|----------------|
| `core/otf_client.py` | UI-free facade: `OTFClient`, `JobParams` |
| `core/client_runtime.py` | High-level orchestration functions |
| `core/client_config.py` | TOML + env configuration loader |
| `core/server_comm.py` | Polling, abort, download |
| `core/file_transfer.py` | Multipart upload (XML + FITS) |
| `core/xml_utils.py` | UWS XML creation and parsing |
| `core/job_store.py` | Local job tracking |
| `core/fits_payload.py` | FITS serialization helpers |
| `core/ssh_tunnel.py` | SSH tunnel for remote mode |
| `core/cianna_xml_updater.py` | Models catalog update from server |

### 4.2 Core Responsibilities

- Build **UWS-compliant XML** job descriptions
- Upload XML + FITS via `POST /jobs/`
- Manage optional SSH tunnels (remote mode)
- Poll job status via `GET /jobs/{jobId}`
- Abort jobs (`PHASE=ABORT`)
- Download results (`GET /jobs/{jobId}/results`)
- Maintain a local mirror of job metadata

---

## 5. OTFClient — UI-Free Facade

`core/otf_client.py` provides the recommended programmatic entry point for
any new interface or script.

### 5.1 JobParams

`JobParams` is a dataclass that holds the full parameters for one inference job.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `image_path` | str | `""` | Absolute path to the FITS file |
| `yolo_model` | str | `"default_model"` | Model name as in `CIANNA_models.xml` |
| `quantization` | str | `"FP32C_FP32A"` | Quantization mode |
| `norm_fct` | list[str] | `["tanh"]` | Normalization functions |
| `ra` / `dec` | float | 0.0 | ROI center coordinates (ICRS, degrees) |
| `roi_w` / `roi_h` | int | 512 | ROI dimensions in pixels |
| `min_pix` / `max_pix` | float | 99.4 / 99.8 | Clipping percentiles |
| `do_tiling` | bool | False | Enable server-side tiling |
| `user_id` | str | auto | User identifier in UWS XML |

### 5.2 OTFClient

```python
# Standard usage via context manager
with OTFClient.from_toml("config/cianna_otf_client_config.toml") as client:
    params = JobParams(image_path="/data/obs.fits", yolo_model="YOLO_CIANNA_...")
    ok, job_id, phase, local_path, msg = client.run(params)
```

Key methods:

| Method | Description |
|--------|-------------|
| `from_toml(path)` | Class method — recommended factory |
| `connect()` | Connect to server (local or SSH) |
| `disconnect()` | Stop SSH tunnel if any |
| `submit(params)` | Submit a `JobParams` job |
| `poll_once(job_id)` | Single phase query |
| `poll_iter(job_id)` | Generator — yields phase on each poll until terminal |
| `download(job_id)` | Download VOTable result |
| `abort(job_id)` | Request cancellation |
| `run(params, on_phase, abort_after_s)` | Full blocking lifecycle |

`poll_iter` is Qt-free and can be driven from any thread or coroutine.

### 5.3 Dependency Injection

`OTFClient.__init__` accepts all runtime functions as explicit parameters,
making unit testing with mocks straightforward. `from_toml()` wires the
production implementations automatically.

---

## 6. client_runtime.py — Function Reference

| Function | Returns | Description |
|----------|---------|-------------|
| `connectserver(cfg)` | `dict` | Connect local or via SSH tunnel |
| `disconnectserver(tunnel)` | — | Stop tunnel |
| `pingServer(server_url)` | `bool` | Reachability check |
| `refreshmodelscatalog(cfg, server_url)` | `(ok, msg, ts)` | Fetch and save `CIANNA_models.xml` |
| `submitjobfromxml(cfg, server_url, xml)` | `(ok, job_id, url, msg)` | Upload XML + FITS |
| `runSingleJob(cfg, server_url, params)` | `(ok, job_id, phase, msg)` | Full job lifecycle |
| `emulateJobs(cfg, server_url, nb, ...)` | `dict` | Parallel multi-job emulation |
| `abortjob(server_url, job_id)` | `bool` | Send ABORT |
| `downloadjobresult(cfg, server_url, job_id)` | `(ok, path, msg)` | Download VOTable |

`connectserver` returns a dict with keys: `ok`, `server_url`, `mode`,
`tunnel`, `message`.

---

## 7. Terminal Dashboard (TTY)

### 7.1 Purpose

Developer monitoring and control tool providing:
- a live view of submitted jobs and their UWS phases,
- explicit control over polling and emulation.

### 7.2 Characteristics

- Terminal-only, no GUI dependencies
- Reads local UWS XML files
- Optionally refreshes by polling the server
- Displays server state without interpretation

### 7.3 Displayed Job Information

For each job: identifier, UWS phase, model name, quantization, normalization,
submission time, local result path if available.

### 7.4 Commands

| Key | Action |
|-----|--------|
| `S` | Connect / reconnect to server |
| `E` | Launch job emulation |
| `n` | Inspect job number *n* |
| `X` | Exit / back |

---

## 8. GUI (PyQt6)

### 8.1 Status

Functional but evolving. Suitable for demos and exploration.

### 8.2 Features

- FITS image visualization
- ROI (Region Of Interest) selection
- Local browsing of `CIANNA_models.xml`
- Model and quantization selection
- Job submission, polling, result download

### 8.3 Workflow

1. Open a FITS image
2. Optionally define an ROI
3. Select model and parameters
4. Submit job
5. Poll job status
6. Download VOTable result

The GUI uses the same `OTFClient` / `client_runtime` API as all other modes.

---

## 9. Configuration

### 9.1 Resolution Order

Machine-specific values are provided via environment variables (`.env`).

| Variable | Description |
|----------|-------------|
| `CIANNA_CLIENT_DATA_ROOT` | Root directory for runtime data (required) |
| `CIANNA_IMAGE_FOLDER` | Directory containing input FITS files (required) |
| `CIANNA_SSH_SERVER_IP` | SSH server IP (remote mode only) |
| `CIANNA_SSH_USERNAME` | SSH username |
| `CIANNA_SSH_PASSWORD` | SSH password |

For each variable: environment variable takes priority over TOML value.

### 9.2 TOML Parameters

| Section | Key | Description |
|---------|-----|-------------|
| `[client]` | `connection_mode` | `"local"` or `"remote"` |
| `[server]` | `host`, `port` | Server address for local mode |
| `[polling]` | `interval_s` | Polling interval in seconds |
| `[polling]` | `timeout_s` | Maximum polling duration |
| `[models]` | `local_registry_path` | Path to local `CIANNA_models.xml` |
| `[ssh]` | `remote_port`, `local_port` | SSH tunnel ports |

Configuration is loaded once at startup and is immutable at runtime.

---

## 10. Local Job Tracking

The client maintains a local job store under `CIANNA_CLIENT_DATA_ROOT/JOBS_SENT/`:
- one directory per submitted job,
- `requestJob_uws.xml` (canonical copy),
- optional server job URL,
- downloaded VOTable results.

The local store mirrors server state and is not authoritative.
It can be rebuilt by polling the server.

---

## 11. Client–Server Contract

### Contract Principles

- UWS is the **single execution contract**
- Server state is authoritative
- Client state is a mirror / cache

### API Summary

| Operation | Endpoint | Notes |
|-----------|----------|-------|
| Submit job | `POST /jobs/` | multipart: `xml` + `fits` — HTTP 303 |
| Poll status | `GET /jobs/{jobId}` | UWS XML or JSON |
| Abort | `POST /jobs/{jobId}/phase` | `PHASE=ABORT` — best-effort |
| Get results | `GET /jobs/{jobId}/results` | Phase must be COMPLETED — VOTable |
| Models registry | `GET /model-files/CIANNA_models.xml` | fetched at connection |

---

## 12. IVOA Compliance (Client Perspective)

| Standard | Client role |
|----------|-------------|
| UWS | Submits UWS-compatible XML, reads UWS phases |
| VOTable | Downloads and stores results locally |
| PROV | Provenance generated server-side — client does not alter it |

---

## 13. Design Principles

- Thin client — no execution logic
- Multiple interfaces, one runtime (`OTFClient` / `client_runtime`)
- Explicit UWS contracts
- Dependency injection for testability (`OTFClient.__init__`)
- Clear separation: UI ↔ facade ↔ runtime ↔ server

---
