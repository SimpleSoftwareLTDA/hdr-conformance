# Agent Guide

Read `.agents/skills/hdr-conformance/SKILL.md` before any HDR task. It contains the normative
rules, test matrix, golden values, and pitfalls.

## Subagents

| Agent | File | Use |
|---|---|---|
| `hdr-spec-reviewer` | `.agents/agents/hdr-spec-reviewer.md` | Review HDR changes against the rule set. Read-only. |
| `hdr-benchmark-runner` | `.agents/agents/hdr-benchmark-runner.md` | Run live probes and produce a report. Does not change system state. |
| `hdr-test-author` | `.agents/agents/hdr-test-author.md` | Add tests, fixtures, and golden vectors. |
| `hdr-standard-editor` | `.agents/agents/hdr-standard-editor.md` | Maintain rules, profiles, and versioning. |

## Repository rules

- English for code, tests, docs, and reports.
- RFC 2119 keywords in normative text. `MUST`, `SHOULD`, and `MAY` have defined meanings.
- Every rule has a test. Every test maps to a rule.
- Evidence over claims. A conformance result without a report and raw evidence is not a result.
- Platform-specific behavior stays in adapters. The core has no vendor assumptions.
- Core and pipeline tests run without HDR hardware. Hardware and measured tests are separate tiers.
- Do not change display state, restart a compositor, or toggle HDR during a read-only probe.
- Keep expected values in one place. Do not duplicate golden values across tests.
