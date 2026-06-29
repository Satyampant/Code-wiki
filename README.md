# LLM Wiki (codebase corpus)

An **agent-maintained wiki that documents a codebase**. You feed it source code; an AI agent reads it
and writes cross-linked, cited markdown pages explaining the architecture. The agent follows a precise
rulebook (`CLAUDE.md`) so it behaves like a disciplined maintainer, not a generic chatbot.

Built incrementally — see [`Docs/llm-wiki-roadmap.md`](Docs/llm-wiki-roadmap.md) and the per-phase
guides in [`Docs/phases/`](Docs/phases/).

## The three layers

| Layer | Path | Owner |
|---|---|---|
| **Sources** | `sources/` | You (read-only to the agent) |
| **Wiki** | `wiki/` | The agent (you read) |
| **Schema** | `CLAUDE.md` | You (the rulebook) |

## The three operations

- `/ingest <path>` — read a source file, create/update every wiki page it touches.
- `/query <question>` — answer from the wiki, with citations.
- `/lint` — audit the wiki for broken links, drift, duplicates, contradictions.

## Layout

```
CLAUDE.md            the schema (rulebook the agent obeys every session)
sources/             the codebase being documented (read-only)
templates/           page skeletons: entity, concept, comparison, synthesis
wiki/                agent-owned pages
  index.md           map of the wiki
  log.md             append-only history of every ingest/lint
  entities/ concepts/ comparisons/ syntheses/
.claude/commands/    /ingest, /query, /lint
```

## Status

Phase 0 complete: skeleton + schema in place, no content yet. Next: Phase 1 (pick 3–5 source files
and run the manual ingest → query → lint loop).
