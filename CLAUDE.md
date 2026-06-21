# CLAUDE.md

Guidance for Claude Code working in this repository.

## What this repo is

A collection of **Claude Code subagent definitions** — not an application. Each `*.md` at the root is one agent: YAML frontmatter + a system prompt written in **Brazilian Portuguese**. There is no build/lint/test step — editing the prompts *is* the work. To use the agents, place the files in a `.claude/agents/` directory (user-level `~/.claude/agents/` or a target project's `.claude/agents/`).

## The SDD pipeline

Four agents implement one **Spec-Driven Development** flow. Each is a stage that consumes the previous stage's artifact and produces the next — they are not independent tools. The user invokes them in order; there is no orchestrator.

```
feature
   │
   ▼
prd-creator      (Product Owner)        ─►  prd.md
   │
   ▼
techspec-creator (arquiteto)            ─►  techspec.md
   │
   ▼
task-executor    (dev sênior)           ─►  tasks.md + código
   │
   ▼
task-reviewer    (qualidade)            ─►  codereview.md
   │
   ├─ REPROVADO ─►  volta ao task-executor
   └─ APROVADO  ─►  fim
```

- **prd-creator** — Product Owner. Asks clarifying questions, writes `prd.md` (the WHAT/WHY). Never touches code, architecture, or tech choices.
- **techspec-creator** — software architect. Explores the target project, writes a detailed `techspec.md` (the HOW/WHERE), with Mermaid flowcharts when a flow is non-trivial.
- **task-executor** — senior developer. Writes a flat `tasks.md` (3–5 items, no subtasks), implements item by item, then invokes `task-reviewer` itself.
- **task-reviewer** — quality engineer. Runs git diff + tests, emits `codereview.md` with status **APROVADO** or **REPROVADO**.

All four agents inherit every tool (no `tools:` key in frontmatter).

### The artifact contract

Every agent reads from and writes into one per-feature folder: **`./tasks/prd-[nome-da-feature]/`** (kebab-case), inside the *target* project being worked on — not this repo. The four filenames (`prd.md`, `techspec.md`, `tasks.md`, `codereview.md`) are a hard interface — don't rename one in a single agent without updating the others. The set of files present is the pipeline's state machine: only `prd.md` → next is the Tech Spec; `prd.md` + `techspec.md` → next is implementation; an APROVADO `codereview.md` → done.

### Two load-bearing concepts — preserve when editing any stage agent

1. **"Mapeamento de camadas"** — a mandatory section in `techspec.md` dictating which layer each kind of code lives in. `task-executor` follows it literally; `task-reviewer` treats "business logic in the wrong layer" as a blocking failure. The single most referenced concept across techspec/executor/reviewer.
2. **The APROVADO/REPROVADO gate** — no middle ground. Any blocking finding (failing test, layer violation, serious rule violation, security issue, tech debt) = REPROVADO, looping back to `task-executor`. A folder existing ≠ done; only an APROVADO `codereview.md` ends the cycle.

## Conventions shared across all agents

- **Persona-first prompt.** Each agent opens by stating its role (Product Owner / arquiteto / dev sênior / engenheiro de qualidade) and its single output artifact.
- **Frontmatter:** `name` + `description` only. Add `tools:` solely to restrict (omit = inherit all).
- **`description`** is a usage trigger and follows one pattern: `"Use este agente para … no fluxo Spec-Driven Development. Atua como [persona]: …"`. Keep all four parallel — Claude Code uses it to decide when to delegate.
- **`<critical>…</critical>` tags** at the top of the body hold the non-negotiable invariants.
- **Respect the target project's `.claude/rules/` and `.claude/skills/`** — techspec/executor/reviewer must read them first and treat violations as blocking. Keep this clause in those three. `prd-creator` stays product-only: no code, no rules.
- **Language is Portuguese**, prompts terse, word budgets explicit (PRD/techspec ~1500 words). Don't pad.
- **Context7 MCP** — techspec/executor/reviewer use it to verify library/framework API versions. `prd-creator` does not.

## Working in this repo

Changes are prose edits to prompt files; nothing runs. Validate by re-reading the affected agent(s) and checking that shared concepts (the four artifact filenames, folder naming, "Mapeamento de camadas", the APROVADO/REPROVADO vocabulary, the persona/description pattern) stay consistent across **all** files that reference them. Cross-file consistency is the main correctness concern.
