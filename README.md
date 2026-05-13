# CIANNA On-the-fly

![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)
![Research Software](https://img.shields.io/badge/type-research%20software-blueviolet)
![Domain](https://img.shields.io/badge/domain-astronomy-lightgrey)
![Python](https://img.shields.io/badge/python-3.11%2B-blue.svg)
![Status](https://img.shields.io/badge/status-active%20development-yellow)

---

> **Please note that for now, the code is not yet available to the community**. 

---

## Overview

**CIANNA_OTF** (On-The-Fly) is a client–server system designed to run **asynchronous, batched inference** on astronomical FITS images using the [**CIANNA** deep‑learning framework](https://github.com/Deyht/CIANNA).

The project addresses a common operational problem in scientific machine learning:

> How to efficiently serve deep‑learning inference when requests arrive asynchronously, but models are expensive to load and run.

CIANNA_OTF solves this by **grouping compatible requests into batches**, loading the model once, and processing multiple jobs together while preserving a clean, job‑oriented API for users.

CIANNA_OTF follows the **IVOA Universal Worker Service (UWS)** pattern for asynchronous execution, and exposes results using the **IVOA VOTable** format.
Initial provenance information is tracked server-side to ensure traceability of scientific results. 
The project is not yet fully FAIR-compliant ([rules](https://www.ccsd.cnrs.fr/en/fair-guidelines/)), but reaching this level is an explicit long-term objective.

---

## What Problem does CIANNA_OTF Solve?

In many scientific workflows:
- inference requests arrive one by one,
- models are large and slow to initialize,
- users expect asynchronous execution and traceable results.

Running inference naïvely (one request → one model load) leads to:
- poor GPU/CPU utilization,
- long response times,
- fragile scripts tightly coupled to the model code.

**CIANNA_OTF introduces an intermediate execution layer** that:
- decouples clients from model execution,
- batches requests automatically,
- exposes a simple, robust job interface.

---

## Key Concepts

### Job
A **job** represents a single inference request. It consists of:
- an XML description (parameters, model choice, region of interest...),
- a FITS image file.

Each job is assigned a unique identifier and progresses through well‑defined execution phases.

### Batch
A **batch** is a group of jobs that share the same:
- model identifier,
- quantization mode.

Batches are created automatically by the server according to configurable rules (minimum size, maximum size, waiting time).


### Asynchronous Execution

Jobs are submitted asynchronously following the IVOA Universal Worker Service model:
- clients submit jobs and receive a job identifier,
- execution is fully server-driven,
- clients poll job phases,
- results are retrieved once the job reaches the COMPLETED phase.

---

## High-Level Architecture

```
┌──────────┐        ┌──────────────┐        ┌──────────────┐
│  Client  │  XML + │   CIANNA_OTF │ Batch  │   CIANNA     │
│          ├───────▶│   Server     ├───────▶│   Model      │
│          │  FITS  │              │        │              │
└──────────┘        └──────┬───────┘        └──────┬───────┘
                            │                         │
                            │  Results (.vot)         │
                            └──────────────◀──────────┘
```

The client–server interaction strictly follows UWS semantics:
the client never controls execution directly and only interacts with the server through job submission, polling, abort, and result retrieval endpoints.

---

## What CIANNA_OTF Is *Not*

- It is not suited for a full survey analysis.
- It is **not** a training framework.
- It does **not** aim to replace workflow managers or schedulers.

CIANNA_OTF focuses on **efficient, asynchronous inference** for data inspection and visualisation.

> **WARNING** : In case you would like to perform predictions on a large dataset, we advise you to use CIANNA framework direclty on your computer/server with an appropriate hardware configuration.


---
## Client Interfaces

CIANNA_OTF provides several **client-side interaction modes**, all relying on the same runtime API:

- **Graphical User Interface (GUI)** *(under development)*  
  A PyQt-based interface for FITS visualization, ROI selection, and exploratory usage.

- **Terminal Dashboard (TTY)**  *(under development)*  
  A developer-oriented interface for monitoring jobs, polling execution phases, aborting jobs, and downloading results.

- **Minimal CLI (headless)**  *(under development)*  
  A scriptable mode intended for automation, CI pipelines, and batch testing.


The GUI remain the best and easiest way to submit requests. The two other ways are more for intensive testing.


All client interfaces are **thin by design**:  
the server remains the authoritative source of truth for job state and execution.


---

## IVOA Compliance

CIANNA_OTF is designed with **Virtual Observatory interoperability** in mind.

Current compliance level:
- **UWS**: partial but functional (job lifecycle, phases, XML representation)
- **VOTable**: full (scientific results format)
- **PROV**: initial level (structured provenance metadata, server-side)

The compliance scope is intentionally incremental and documented in detail in the technical documentation.

---

## Documentation

- **README.md** (this file): functional overview
- **doc/server/**: server-side technical documentation ([here](./doc/CIANNA_OTF_Technical_Doc_Server.md))
- **doc/client/**: client-side technical documentation ([here](./doc/CIANNA_OTF_Technical_Doc_Client.md))
  - runtime API
  - TTY dashboard
  - GUI (work in progress)
  - CLI usage

---

## Project Status and Maturity

CIANNA_OTF is an **active research and engineering project**.

- Functional interfaces (job submission, polling, results) are stable
- Internal implementation may evolve
- Server framework (Flask) for development only

The project is suitable for:
- research deployments,
- shared laboratory services,
- controlled production environments.


---

## License

This code is under the APACHE LICENCE 2.0
The licence.md file is attached [here](LICENCE.md)

---

## Contact / Contribution

Contributions, issues, and discussions are welcome.
Please refer to the technical documentation before extending the system.
