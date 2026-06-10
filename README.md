# Homelab platform (Aluna)

This is the stateful **platform and infrastructure** repository for the **Aluna** ecosystem (control, context, awareness, and surface layers, see [docs/aluna-platform/platform-ontology.md](docs/aluna-platform/platform-ontology.md)).

This repository is the operational source of truth for the homelab platform: storage, Kubernetes, networking, observability, deployment patterns, and deployed workload state.

It is not the primary home of product application source code. Product code lives in separate workload repos. This repo owns how workloads are deployed, operated, documented, and evolved on the platform.

## Purpose

The platform exists to provide a production-like environment for:

- running real workloads  
- validating platform engineering decisions  
- preserving infrastructure truth in-repo  
- supporting the **Aluna** product stack as it grows  

Ontology mapping (high level):

- **Ceiba** — first enforceable control-plane boundary under Aluna (keys, policies, quotas).  
- **Aluna Context** — meaning / geo-aware data plane (APIs, Postgres, ingestion).  
- **Aluna Mira** — awareness (observability, interpretation, AI-assisted ops).  
- **Aluna Terra** — surface (dashboards, maps, analytics UI).  

## Core principle

**The repo is the memory, not the chat.**

Project state, architecture, tasks, lessons, and deployment truth are written here so future sessions and agents can reload context from files.

## What lives here

- Platform foundation: k3s, networking, storage, observability  
- Workload deployment state: manifests, environment wiring, runbooks  
- Platform and architecture documentation ([docs/aluna-platform/](docs/aluna-platform/) umbrella; [docs/aluna-context/](docs/aluna-context/) for the **Aluna Context** workload)  
- Roadmap, tasks, lessons, and ADRs  

## What you will not find here

- Primary application source code for Aluna products  
- SDK source code  
- Control-plane UI source code  
- Runtime service source code  

Those live in separate solution repos (naming may still reflect legacy project names in remotes).

## Repo layout

- `context/` — machine-readable and operational project state  
- `docs/` — architecture, deployment, engineering logs; **`docs/aluna-platform/`** (ontology + platform diagrams), **`docs/aluna-context/`** (Aluna Context runtime diagrams)  
- `tasks/` — scoped execution queue, backlog, lessons  
- `platform/` — platform primitives and infrastructure implementation  
- `workloads/` — workload-specific deployment state and runbooks  
- `deployments/` — cluster deployment artifacts and shared manifests  

## Workload model

Each workload tracked here should have deployment notes, manifests or references, environment expectations, and operational runbooks. Workloads are deployed systems, not a single monolithic source tree.

## Operating model

This repo follows the workflow in [context/AGENTS.md](context/AGENTS.md):

`context` → `task` → implementation → verification → documentation / state update  

## Near-term focus

Support **Ceiba** and the **Aluna Context** data plane with:

- stable app hosting  
- PostgreSQL  
- Redis  
- domain + TLS  
- minimal deployment reliability  
- backups  
- just-enough observability (foundation for **Aluna Mira**)  

## Notes

- Keep platform work proportional to current workload needs.  
- Do not overbuild infrastructure before workloads prove what is needed.  
- **Hostname:** docs use **aluna-node-01**; align `/etc/hostname` and DNS when you rename the machine.  
- **Git remote:** may remain `https://github.com/davidpal3c/alethos_platform.git` until the remote is renamed; content in this repo describes the **Aluna** platform.  
