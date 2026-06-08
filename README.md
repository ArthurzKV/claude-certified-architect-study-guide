# Claude Certified Architect Study Guide

An unofficial, community-oriented study guide for the **Claude Certified Architect - Foundations (CCA-F)** certification.

This repository is built for self-study: domain notes, scenario notes, decision frameworks, anti-patterns, API errata, and original practice questions grounded in public Anthropic documentation.

## What This Is

- A structured study guide for the CCA-F blueprint.
- Original notes and practice drills based on public documentation.
- A no-dumps, no-leaks resource for understanding the underlying architecture patterns.
- A public companion for people preparing through legitimate practice and review.

## What This Is Not

- Not affiliated with, endorsed by, or sponsored by Anthropic.
- Not a reproduction of official exam content.
- Not a collection of real exam questions.
- Not a proctored-exam assistance tool.
- Not a redistribution of third-party PDFs or proprietary training material.

## Exam Snapshot

Publicly documented CCA-F facts:

| Item | Detail |
| --- | --- |
| Format | 60 multiple-choice, scenario-based questions |
| Time | 120 minutes |
| Score | Scaled 100-1000 |
| Passing score | 720 |
| Scenarios | 4 randomly selected from 6 possible production scenarios |
| Domains | D1 27%, D2 18%, D3 20%, D4 20%, D5 15% |

Always verify against the official Anthropic/Skilljar materials before scheduling, because certification details can change.

## Study Path

1. Read the blueprint and strategy: [docs/reference/exam-blueprint-and-strategy.md](docs/reference/exam-blueprint-and-strategy.md)
2. Learn the anti-patterns: [docs/reference/anti-patterns-catalog.md](docs/reference/anti-patterns-catalog.md)
3. Study the five domains in order:
   - [D1 Agentic Architecture & Orchestration](docs/domains/d1-agentic-architecture-orchestration.md)
   - [D2 Tool Design & MCP Integration](docs/domains/d2-tool-design-mcp.md)
   - [D3 Claude Code Configuration & Workflows](docs/domains/d3-claude-code-config-workflows.md)
   - [D4 Prompt Engineering & Structured Output](docs/domains/d4-prompt-engineering-structured-output.md)
   - [D5 Context Management & Reliability](docs/domains/d5-context-management-reliability.md)
4. Read all six scenario notes:
   - [S1 Customer Support Resolution Agent](docs/scenarios/s1-customer-support-resolution-agent.md)
   - [S2 Code Generation with Claude Code](docs/scenarios/s2-code-generation-claude-code.md)
   - [S3 Multi-Agent Research System](docs/scenarios/s3-multi-agent-research-system.md)
   - [S4 Developer Productivity with Claude](docs/scenarios/s4-developer-productivity.md)
   - [S5 Claude Code for CI/CD](docs/scenarios/s5-claude-code-cicd.md)
   - [S6 Structured Data Extraction](docs/scenarios/s6-structured-data-extraction.md)
5. Drill with [original practice questions](docs/practice-questions/).
6. Use [docs/reference/current-api-errata-2026.md](docs/reference/current-api-errata-2026.md) for version-sensitive behavior.

## Repository Map

| Folder | Purpose |
| --- | --- |
| [docs/domains](docs/domains/) | One guide per exam domain |
| [docs/scenarios](docs/scenarios/) | The six scenario families the exam is organized around |
| [docs/reference](docs/reference/) | Strategy, anti-patterns, decision frameworks, glossary, errata |
| [docs/primary-source-synthesis](docs/primary-source-synthesis/) | Original synthesis of public Anthropic docs/research |
| [docs/practice-questions](docs/practice-questions/) | Original concept drills and flashcards |

## Core Mental Models

- Prefer the simplest architecture that solves the task.
- Use workflows for predictable control flow; use agents when the model must dynamically choose the path.
- Use `stop_reason`, not prose, to control agent loops.
- Enforce critical business rules in code/hooks, not only in prompts.
- Keep tool sets small and semantically distinct.
- Use MCP for clean integration boundaries, especially across teams/tools.
- Validate structured output with schemas and retry/repair logic.
- Measure reliability by slice, not only aggregate accuracy.

## Legal And Ethical Boundary

Use this repository only for self-administered study and practice. Do not use it during any proctored, graded, monitored, or official exam.

If you contribute, do not submit:

- Real exam questions.
- Screenshots or transcripts from official exams.
- Paid course material.
- Proprietary PDFs.
- Long copied passages from Anthropic or third-party docs.
- Anything you do not have the right to publish.

See [NOTICE.md](NOTICE.md) and [CONTRIBUTING.md](CONTRIBUTING.md).

## License

Study content in this repository is licensed under [CC BY-NC 4.0](LICENSE), unless a file says otherwise.

