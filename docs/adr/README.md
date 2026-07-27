# Architecture decision records

Records of decisions that were **contested, expensive to reverse, or likely to be questioned later**. Not a log of every choice — an ADR that records the obvious is noise.

## Format

One file per decision: `NNNN-short-title.md`.

```markdown
# ADR-NNNN — Title

**Status:** Proposed | Accepted | Superseded by ADR-NNNN
**Date:** YYYY-MM-DD

## Context
The situation and constraints that forced a decision.

## Decision
What was chosen, stated plainly.

## Consequences
What this makes easy, what it makes hard, and what it rules out.

## Alternatives considered
What else was on the table and why it lost.
```

## Rules

- ADRs are **immutable once accepted**. A changed decision gets a new ADR that supersedes the old one; the original stays as the record of what was believed at the time.
- Consequences include the **bad** ones. An ADR that only lists benefits is marketing.
- Link the ADR from the doc it affects.

## Decisions recorded so far

None yet. Decisions made during the platform foundation phases — storage tiering, ext4 over ZFS, k3s over kubeadm, `nofail` on tier mounts — are documented inline in the phase documents and are candidates for retroactive ADRs.
