---
name: codebase-diagram-explainer
description: Explain unfamiliar codebases with clear architecture diagrams, sequence/call workflows, dependency maps, and onboarding-oriented summaries. Use when a user asks to understand repository structure, data flow, request lifecycle, module relationships, or execution paths, especially when visual diagrams and stepwise workflows are requested.
---

# Codebase Diagram Explainer

## Outcome

Produce explanations that are both technically accurate and easy to scan:
- One short plain-language overview
- One architecture map (components and boundaries)
- One execution workflow (request/event path)
- One dependency or ownership map (optional, when useful)
- A "how to read this codebase" checklist for new contributors

Prefer Mermaid diagrams in Markdown unless the user requests another format.

## Workflow

1. Map the terrain quickly.
- Identify stack, entrypoints, runtime boundaries, and key directories.
- Use fast repo discovery commands (`rg --files`, `rg`, `ls`, `find`) and read only relevant files.

2. Locate the main lifecycle.
- For backend: trace request -> routing -> service -> data -> response.
- For event systems: trace producer -> transport -> consumer -> side effects.
- For frontend: trace route -> state -> API calls -> render/update.

3. Build three views.
- System view: major components and external dependencies.
- Flow view: ordered steps for one representative operation.
- Code ownership view: modules and responsibilities.

4. Explain at layered depth.
- Layer 1: 5-8 sentence summary for a new engineer.
- Layer 2: component-level details with file references.
- Layer 3: critical path and failure points (timeouts, retries, validation, caching, race risks).

5. Validate accuracy.
- Ensure every diagram edge corresponds to observed code paths.
- Flag uncertainty explicitly as assumptions.
- Do not invent services, queues, or tables not supported by code.

## Diagram Templates

Load [references/diagram-templates.md](references/diagram-templates.md) and adapt the closest template.

- `architecture`: component boundaries and integrations
- `sequence`: request/event execution order
- `dependency`: module or package relationships
- `state`: lifecycle/state transitions when behavior is stateful

## Output Contract

When presenting results, use this structure:
1. `Overview`
2. `Architecture Diagram` (Mermaid)
3. `Workflow Diagram` (Mermaid)
4. `Key Files and Responsibilities`
5. `Critical Path Notes`
6. `Contributor Onboarding Path`

Keep prose concise. Prefer direct statements over speculative commentary.

## Quality Bar

- Cite concrete files for each major claim.
- Keep diagrams readable (around 7-20 nodes each).
- Use consistent names between text and diagrams.
- Include at least one "start here" path for first-time contributors.
- If the repository is large, scope to one subsystem first, then expand.