# ddd-ai-skills-applied

Reusable **AI skills** for applying **DDD / CQRS / Event Sourcing** patterns in real codebases.

Maintained by **Piotr Wyczesany ([@pwyczes](https://bio.link/pwyczes))** — [Pattern Applied](https://patternapplied.com/en/).

> 🚧 Work in progress  
> Structure and conventions will evolve. Expect renames and reshuffles while the “skill format” stabilizes.

---

## What this repo is

This repository is a **skill library**: short, composable instruction packs you can feed into AI coding assistants
to get **consistent, production-grade** outcomes for:

- **DDD tactical patterns** (Value Objects, Factories, Aggregates, Domain Events, Policies…)
- **CQRS** (command model, query model, read models, messaging boundaries…)
- **Event Sourcing** (event design, versioning, projections, replay/reprocess…)
- **strategic DDD** (Bounded Contexts, context mapping, subdomains…)

The guiding idea: **make “how we do DDD” portable across agents**.

---

## Quick start

Add this repo to your workspace (or copy a single skill folder), then follow your agent’s docs for importing repo-based skills: 
 * [Cursor Skills](https://cursor.com/docs/context/skills),
 * [Claude GitHub integration](https://support.anthropic.com/en/articles/10167454-using-the-github-integration),
 * [OpenAI Codex Agent Skills](https://developers.openai.com/codex/skills/), 
 * [GitHub Copilot repository instructions](https://docs.github.com/en/copilot/concepts/prompting/response-customization). 

Once the files are available, open a `SKILL.md` and tell the agent to follow it.

---

## Skill usage pattern (the “3 inputs” that make it work)

When invoking any skill, provide these three things:

1) **Context**  
   Language/runtime/version, framework choices, packaging conventions, constraints.

2) **Target**  
   What you want built/changed (class names, packages, boundaries, API shape).

3) **Definition of done**  
   Tests required, compilation expectations, edge cases, style requirements.

Example instruction to an agent:

> “Apply `value-object-applied/SKILL.md`.  
> Implement `Money` and `Currency` value objects for a Java 21 project, package `com.acme.billing.domain`.  
> Provide JUnit tests, include invariants, and keep it immutable.”

---

## Status / roadmap

Initial focus:
- Tactical DDD skills for **Java** (Value Object, Factory, then Aggregates + Domain Events)

Planned expansion:
- CQRS and ES skills (read models, projections, event versioning)
- Strategic DDD skills (context mapping, boundaries, integration patterns)
- Agent-specific “glue” snippets where it helps, without locking the repo to one tool
