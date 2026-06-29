# LLM Wiki — Schema & Operating Rules

> This file is the **schema**: the rulebook the agent obeys on every session. It is loaded
> automatically by Claude Code. Read it fully before performing any operation.
>
> **The schema is the product.** When the wiki misbehaves, the fix is almost always a sharper
> rule *here* — not a hand-edit to a wiki page.

---

## 1. Mission & layers

This repo is an **agent-maintained wiki describing a codebase**. There are three layers, and the
rule about *who edits what* is absolute:

| Layer | Path | What it is | Who edits it |
|---|---|---|---|
| **Sources** | `sources/` | The codebase being documented (read-only to the agent). | The human. **Never the agent.** |
| **Wiki** | `wiki/` | Markdown pages explaining the code. | **The agent, only.** |
| **Schema** | `CLAUDE.md` (this file) | The rulebook. | The human, only. |

**Hard rules:**

- **Never modify anything under `sources/`.** It is read-only input. You read it; you never write it.
- **The human does not hand-edit `wiki/` pages.** If a page is wrong, the human fixes a *rule* in
  this file and asks you to re-run. (You, the agent, are the only writer of `wiki/`.)
- **Everything is git.** Each ingest and each lint pass should end as a commit, so knowledge changes
  are diff-able and reversible.

---

## 2. Page types

Every wiki page is exactly one of four types. Pick the narrowest type that fits.

| Type | Answers | Examples (codebase) | Folder |
|---|---|---|---|
| **entity** | "What *is* this thing?" | A class, module, package, service, endpoint, DB table, config object | `wiki/entities/` |
| **concept** | "How does this *work* across files?" | Auth flow, caching strategy, retry logic, build pipeline, a design pattern in use | `wiki/concepts/` |
| **comparison** | "How do these two differ?" | Two similar services, old vs new code path, two libraries doing a similar job | `wiki/comparisons/` |
| **synthesis** | "Give me the whole story." | End-to-end request lifecycle, "everything that happens on checkout" | `wiki/syntheses/` |

**The ladder:** entities are atoms → concepts group atoms into ideas → syntheses weave concepts into
stories. Comparisons reference entities/concepts. Prefer creating entity pages first; let concept and
synthesis pages grow out of them.

**One page = one subject.** If a page starts explaining two distinct things, it must be split (lint
will catch this). Keep entity pages tight; push cross-cutting explanation up into concept pages.

---

## 3. The `[[wiki-link]]` convention

Pages reference each other with double brackets. This is what turns pages into a navigable graph.

- A link uses the target page's **canonical name** = its filename without `.md`.
  Example: a file `wiki/entities/AuthService.md` is linked as `[[AuthService]]`.
- **Filenames are the canonical id.** Use the natural name of the code thing for entities
  (`AuthService`, `User-model`, `payments-module`). Use `kebab-case` for concepts/comparisons/
  syntheses (`authentication-flow`, `rest-vs-graphql`, `request-lifecycle`).
- A page may declare `aliases` in its frontmatter so alternate spellings resolve to it. A `[[link]]`
  may target a canonical name **or** any alias.
- **Every link must resolve** to an existing page. Don't link to pages you haven't created. If a page
  should exist but doesn't yet, create at least a stub, or note the gap in `log.md` — don't leave a
  dangling `[[link]]`. (Lint enforces this.)
- Link generously but meaningfully: link the first mention of a related entity/concept in a page.

---

## 4. Citation format (codebase-specific)

**Every factual claim in a wiki page must be traceable back to the code that justifies it.** This is
what makes the wiki trustworthy instead of confident guessing.

A citation points at a **file and line range** inside `sources/`:

```
The session token is refreshed on every authenticated request. [src: auth/session.py:40-72]
```

- Format: `[src: <path-relative-to-sources>:<Lstart>-<Lend>]`. A single line is `[src: file.py:66]`.
- Paths are **relative to `sources/`** (write `auth/session.py`, not `sources/auth/session.py`).
- Put the citation at the **end of the sentence or bullet** it supports.
- A claim that draws on several places may carry several citations.
- **Prefer precise line ranges over whole-file citations.** Precision is what makes Phase 4's
  incremental `git diff` re-ingest possible: when a file changes, we want to know which pages depended
  on which lines.
- If you cannot cite a claim to the source, either don't make the claim or mark it clearly as an
  inference (e.g. `(inferred)`), never as fact.

Also record, in each page's frontmatter, the list of source files the page draws on (`sources: [...]`).

---

## 5. Page structure & frontmatter

Every page begins with YAML frontmatter:

```yaml
---
title: <human title>
type: entity | concept | comparison | synthesis
aliases: [alt name, another alt]      # optional; [] if none
sources: [auth/session.py, ...]       # source files this page draws on, relative to sources/
last_updated: YYYY-MM-DD              # date of the last ingest/lint that touched this page
---
```

The **body** uses the fixed section set for its type. Start from the matching file in `templates/`:

- `templates/entity.md` — sections: *What it is · Responsibilities · Key collaborators · Gotchas/notes · Related*
- `templates/concept.md` — sections: *Summary · How it works · Key pieces · Edge cases · Related*
- `templates/comparison.md` — sections: *What's being compared · Side-by-side · When to use which · Related*
- `templates/synthesis.md` — sections: *Question · The story · Step-by-step · Pages this draws on · Related*

Keep the fixed sections even if a section is brief; this lets lint check completeness and keeps every
page of a type reading the same way.

---

## 6. Ingest workflow

Triggered by `/ingest <path>` (one source file or a small set). Steps:

1. **Read** the source file(s) fully.
2. **Discuss** the key takeaways with the human first (high supervision in early phases). Don't write
   pages until the human is aligned, unless they've asked for batch/low-supervision mode.
3. **Search the existing wiki** (`wiki/index.md` + filename/heading/grep search) for pages this source
   touches. Decide what already exists vs. what's new.
4. **For each thing the source describes:**
   - If **no page exists** → create one from the right `templates/` skeleton.
   - If a **page exists** → **update and reconcile** it. *Integrate*, don't merely append: rewrite so
     the new knowledge is woven in and the page still reads as one coherent explanation.
5. **Link** related pages with `[[wiki-links]]`.
6. **Cite** every claim with `[src: file:lines]` and update each page's `sources:` frontmatter.
7. **Flag contradictions:** if the source conflicts with an existing page, surface it to the human and
   note it — do **not** silently overwrite. (Wrong claims propagate fast through links.)
8. **Update `wiki/index.md`** so new pages are discoverable.
9. **Set `last_updated`** on every page touched.
10. **Append an entry to `wiki/log.md`** (see §9 format).
11. **Commit** with a message like `ingest: <source path>`.

**Append vs. integrate** is the core discipline: a weak wiki tacks facts onto the bottom of a page; a
strong wiki reconciles them into a coherent whole.

---

## 7. Query workflow

Triggered by `/query <question>`. Steps:

1. **Find candidate pages** via `wiki/index.md` plus filename/heading/grep search.
2. **Read** those pages.
3. If they're not enough, **follow `[[links]]`** to related pages and read more.
4. **Synthesize an answer *from the pages*** — never invent facts. If the wiki can't support an
   answer, say so plainly.
5. **Cite the wiki pages** the answer used (e.g. "from [[authentication-flow]], [[AuthService]]").
6. If the wiki was **thin** on the topic, note what to ingest next to close the gap.
7. **Don't write pages during a query** unless explicitly asked. (Standing synthesis pages for
   recurring questions come later, in Phase 3.)

---

## 8. Lint rules

Triggered by `/lint`. Audit the whole wiki against this checklist; fix the safe items, flag the
judgment calls for the human:

- **Broken links** — `[[X]]` where no page (or alias) `X` exists. → create a stub or remove the link.
- **Orphan pages** — pages nothing links to. → link them in, or question whether they should exist.
- **Bloated/scope-drifted pages** — one page covering two subjects. → **split** into separate pages.
- **Duplicate/overlapping pages** — two pages for the same thing. → **merge**, leave a redirect note,
  add the merged-away name as an `alias`.
- **Contradictions** — two pages that disagree. → **flag for the human**, don't auto-resolve.
- **Stale citations** — `[src: file:lines]` pointing at code that has moved or changed. → re-check
  against `sources/` and fix.
- **Missing structure** — page missing required frontmatter fields or its type's required sections.
- **Stale `last_updated`** — dates that don't reflect reality.

End a lint pass by reporting findings, applying safe fixes, listing flagged items, and committing with
a message like `lint: <summary>`.

---

## 9. The log (`wiki/log.md`)

Append-only journal. Every ingest and lint appends one entry, newest at the bottom:

```
## YYYY-MM-DD — ingest: <source path>
- Pages created: [[A]], [[B]]
- Pages updated: [[C]]
- Contradictions flagged: <none | description>
- Notes: <anything notable>
```

```
## YYYY-MM-DD — lint
- Fixed: <broken links, splits, merges...>
- Flagged for human: <contradictions, judgment calls>
```

---

## 10. Conventions summary (quick reference)

- Never write `sources/`. Only the agent writes `wiki/`. Only the human writes this schema.
- Entity filenames = the code thing's name; concept/comparison/synthesis = `kebab-case`.
- Links: `[[canonical-name]]`, must resolve. Citations: `[src: file:lines]`, relative to `sources/`.
- Integrate, don't append. Flag contradictions, don't overwrite. One page = one subject.
- Every operation ends in a `log.md` entry and a git commit.
