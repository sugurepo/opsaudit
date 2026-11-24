# Project Rules
Read `PLAN.md` first—it is the source of truth for architecture, schemas, and milestones.

## Workflow & Code Standards
- Work on **one milestone at a time**, starting with the milestone specified in the prompt.
- Outline the plan and explain core concepts in plain language before writing code.
- Prioritize simple, readable, explicit code over clever abstractions.
- After completing each module, include a 3-sentence summary check to verify architectural understanding.
- Do not execute shell commands or tests automatically; provide the exact commands for the user to run locally.

## Data & Security Guardrails
- **Strict accuracy:** Never invent metrics, test results, or unbuilt features.
- **Synthetic data only:** Never process, read, or upload real documents.
- **Secrets protection:** Never commit `.env` or expose API keys.