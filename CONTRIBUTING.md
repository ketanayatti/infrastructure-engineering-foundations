# Contributing

Thank you for contributing to Infrastructure Engineering Foundations. This repository is a long-term operational engineering lab. Contributions should improve accuracy, clarity, or operational usefulness.

## Contribution Philosophy

- Prefer small, reviewable changes over large rewrites.
- Ground changes in evidence, reproducible experiments, or verifiable references.
- Treat documentation and automation as production-grade artifacts.

## Engineering Standards

- Reliability-first: consider failure modes, rollback paths, and operational impact.
- Security-aware: avoid introducing secret exposure or unsafe examples.
- Operational correctness is more important than cleverness.
- Include verification steps for any change that affects behavior.

## Documentation Standards

- Write for operational reuse and incident context.
- Capture assumptions, constraints, and verification steps.
- Keep diagrams aligned with architecture notes and experiments.
- Prefer short, structured sections that scan well during incidents.

## Commit Conventions

Use a clear, consistent format:

```
<type>(<scope>): <summary>
```

Types: feat, fix, docs, chore, refactor, test, build, ci

Examples:

```
docs(linux): add journald troubleshooting notes
feat(cicd): add rollout verification checklist
fix(k8s): correct service discovery diagram
```

## Pull Request Guidelines

- Provide a concise summary and the operational intent.
- Identify architecture and operational impact.
- Include verification steps or testing results.
- Update related diagrams, notes, learnings, and mistakes when needed.
- Keep PRs focused on a single topic or module.

## Issue Reporting Guidelines

- Use the appropriate issue section in the issue template.
- Include environment details, logs, and reproduction steps.
- Redact secrets and sensitive data.
- Describe impact, scope, and any attempted mitigations.

## Experimentation Guidelines

- Run experiments in safe, isolated environments.
- Document hypotheses, inputs, and measurable outcomes.
- Time-box experiments and record cleanup steps.
- Capture failure modes and operational insights.

## Operational Engineering Mindset

- Treat system state as evidence, not opinion.
- Prefer repeatable diagnostics over one-off fixes.
- Make rollback and recovery paths explicit.
- Keep access and change history auditable.

## Respectful Collaboration Principles

- Be direct, professional, and constructive.
- Assume good intent and focus on the work.
- Keep discussions technical and evidence-based.
- Respect different experience levels and learning paths.
