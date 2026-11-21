# AI usage declaration

> **Mandatory.** If you did not use any AI tool, write "No AI tools were used." under *Summary* and leave the tables empty.

## Summary

AI assistance was used extensively as a strategic pair architect and accelerator to analyze the assessment repository, structure decision frameworks, synthesize incident logs, draft Markdown deliverables with Mermaid diagrams, and validate OpenAPI and YAML syntax. All architectural decisions, trade-offs, calculations, and code review verdicts were rigorously verified against the scenario constraints.

## Tools

| Tool (name and version or plan) | Where it ran (web, IDE, CLI) | Data you shared with it |
|---|---|---|
| Google Gemini 3.8 Flash (High) via Antigravity Agent | IDE / Workspace Agent | Assessment scenario, tasks, context files, incident logs/metrics, and PR-388 diff from the public assessment repository. No confidential or personal real-world data was shared. |

## Uses

| Part | What you asked the tool to do | What you kept | How you validated it | What you discarded, and why |
|---|---|---|---|---|
| **A — Architecture & Governance** | Analyze constraints (C1–C9), propose C4 diagrams in Mermaid, design OpenAPI 3.0 schema, and draft ADR-0001 and ADR-0002. | C4 context and container diagrams, tenant bulkhead model, OpenAPI 3.0 schema, and ADR structures. | Verified OpenAPI schema validity against OpenAPI 3.0.3 specifications. Validated that C1, C2, and C6 constraints were met. | Discarded an initial suggestion for a single shared regional database with row-level security (RLS) because it violates sovereign data protection isolation (C5). |
| **B — Quality & Automation** | Draft test strategy, quality gates, DORA targets, and an Azure DevOps multi-country pipeline template. | Testing pyramid, Pact contract testing flow, SonarQube/Gitleaks quality gates, and progressive canary rollout YAML. | Verified YAML syntax and ensured stages adhere to progressive country rollouts (MX -> CO -> CL). | Discarded an all-in-one simultaneous multi-country deployment pattern because it creates severe cross-country blast-radius risks. |
| **C — Reliability & Operations** | Analyze incident INC-0907 logs, CSV metrics, and chat logs to reconstruct the timeline, root causes, and postmortem. | Complete incident timeline with exact timestamp and log line citations, 5 Whys, and blameless action items. | Manually cross-referenced log timestamps in `quote-xapi.log`, `TIMELINE.md`, and `channel-metrics.csv` to confirm exact cause of failure. | Ruled out the database CPU load and IDP key rotation hypotheses after cross-referencing timeline timestamps. |
| **D — PR Review & Mentoring** | Review PR-388 diff, identify architectural and security violations, and draft mentoring feedback (< 400 words). | Findings table ordered by severity (Critical, High, Medium, Low), refactoring proposal, and empathetic mentoring message to Diego R. | Verified exact line numbers in PR-388 diff and verified that hardcoded client secret, PII logging, and breaking schema changes were accurately identified. | Discarded harsh or punitive phrasing in the developer message; ensured the tone remained encouraging and focused on teaching architectural principles. |
| **E — Roadmap & Feasibility** | Prioritize tech debt items using RICE/WSJF, structure Q1/Q2 roadmap based on team capacity, and analyze GenAI feasibility. | RICE prioritization table, team capacity calculations (58 pw internal, 48 pw Nimbus, 6 pw borrowed), and 1-page executive sponsor memo. | Calculated mathematical person-week allocations against team capacity in `TEAM.md` to ensure roadmap feasibility without overcommitting. | Rejected immediate Q1 production release of GenAI assistant due to regulatory advice restrictions and security constraints. |
| **F — Responsible AI** | Draft enterprise AI guidelines for developers and vendors. | Clear guidelines covering approved tools, data classification boundaries, mandatory review, and disclosure rules. | Ensured guidelines provide concrete enforceable checks, including SonarQube, Gitleaks, and contract clauses for vendor renewal. | Discarded vague generic AI policy statements in favor of actionable, measurable rules. |

## Reflection

Using an agentic AI assistant dramatically accelerated the synthesis of complex multi-file operational evidence (cross-referencing logs, metrics, and chat transcripts during INC-0907). It also helped generate clean Mermaid diagrams and structured tables rapidly. 

However, AI tools require active human oversight: initial suggestions tended towards generic enterprise patterns (such as shared multi-tenant database clusters) that would have violated Meridian's sovereign data segregation laws (C5) and cross-country availability isolation (C1). Deep domain understanding of insurance regulations, Salesforce governor limits, and vendor dynamics was essential to guide the AI and produce a sound, defensible technical solution.
