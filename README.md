This is the stateful platform and infrastructure repo for the Aletheos ecosystem.

This repository is the operational source of truth for the homelab platform:
storage, Kubernetes, networking, observability, deployment patterns, and deployed workload state.

It is not the primary home of product source code. Product code lives in separate workload repos.
This repo owns how workloads are deployed, operated, documented, and evolved on the platform.

## The Purpose

The platform exists to provide a production-like environment for:

- running real workloads
- validating platform engineering decisions
- preserving infrastructure truth in-repo
- supporting the Aletheos ecosystem as it grows

The first major workload under this model is **Aletheos Access**.

## Core principle

**The repo is the memory, not the chat.**

Project state, architecture, tasks, lessons, and deployment truth are written here so future sessions and agents can reload context from files.

## What lives here

- platform foundation:
  - k3s
  - networking
  - storage
  - observability
- workload deployment state:
  - manifests
  - environment wiring
  - runbooks
- platform and architecture documentation
- roadmap, tasks, lessons, and ADRs

## What you won't find here

- primary application source code for Aletheos products
- SDK source code
- control-plane UI source code
- runtime service source code

Those live in separate solution repos such as `aletheos-access`.

## Repo layout

- `context/`  
  machine-readable and operational project state
- `docs/`  
  architecture, deployment, security, ADRs, engineering logs
- `tasks/`  
  scoped execution queue, backlog, and lessons
- `platform/`  
  platform primitives and infrastructure implementation
- `workloads/`  
  workload-specific deployment state and runbooks
- `deployments/`  
  cluster deployment artifacts and shared manifests

## Workload model

Each workload tracked here should have:

- deployment notes
- manifests or deployment references
- environment expectations
- operational runbooks

This repo treats workloads as deployed systems, not as source-code monoliths.

## Current strategic role

The platform supports the Aletheos ecosystem:

- **Aletheos Access** → first workload / first revenue-oriented product
- **Aletheos Data** → future flagship data platform
- **Aletheos Ops** → future observability/AI workload
- **Mini-LLM** → future internal/private inference support

## Operating model

This repo follows the stateful development workflow defined in `AGENTS.md`.

Meaningful infrastructure work should move through:

context → task → implementation → verification → documentation/state update

## Near-term focus

Support the launch of **Aletheos Access** with:

- stable app hosting
- PostgreSQL
- Redis
- domain + TLS
- minimal deployment reliability
- backups
- just-enough observability

The platform should support the product, not block it.

## Notes

- Keep platform work proportional to current workload needs.
- Do not overbuild infrastructure before the workload proves what is needed.
- Let real workloads drive platform maturity.