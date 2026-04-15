# Context Files

This directory contains the state files used by the agent-driven
development workflow described in AGENTS.md.

These files represent the current operational state of the platform.

**Aluna** umbrella docs (ontology + platform diagrams): [../docs/aluna-platform/](../docs/aluna-platform/). **Aluna Context** workload diagrams: [../docs/aluna-context/](../docs/aluna-context/).

They are intentionally committed to the repository to:

• ensure reproducibility <br/>
• allow agents to reload project state <br/>
• document architecture decisions <br/>
• support portfolio transparency

Sensitive data such as credentials or private tokens must never
be stored here.