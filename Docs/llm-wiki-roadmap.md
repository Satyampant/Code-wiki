# LLM Wiki — Incremental Implementation Roadmap

A hands-on plan for building Andrej Karpathy's "LLM Wiki" pattern from scratch, one working slice at a time. The goal isn't to ship a product — it's to understand the pattern by building every layer yourself.

---

## The idea in one screen

**Three layers:**

- **Sources** — your curated input documents (articles, papers, transcripts, code). Immutable. The agent reads them but never edits them.
- **Wiki** — a folder of markdown pages the *agent* owns: summaries, entity pages, concept pages, comparisons, syntheses, plus an index and a log. Pages link to each other with `[[wiki-links]]`. You read; the agent writes.
- **Schema** — a config / rulebook telling the agent how the wiki is structured (page types, link and citation conventions, ingest workflow, query behavior, lint rules). This is what makes the agent a disciplined maintainer instead of a generic chatbot.

**Three operations:**

- **Ingest** — read a source, discuss takeaways, write or update every page it touches (one source can ripple through 10–15 pages), append an entry to the log.
- **Query** — find the relevant pages, read them, synthesize a cited answer.
- **Lint** — a periodic audit: fix broken links, split pages that have grown to cover two concepts, flag contradictions, kill drift.

**Why not just RAG:** RAG re-retrieves and re-synthesizes raw chunks on every question. The wiki *pre-compiles* knowledge into structured pages, so understanding compounds across sessions instead of being rebuilt each time.

---

## Design principles (read before you start)

1. **The agent owns the wiki; you own the sources and the schema.** Don't hand-edit wiki pages — fix the schema and re-run, or you'll never learn whether the agent can maintain it on its own.
2. **Start as just markdown + a coding agent.** Add infrastructure only when scale forces it. No vector DB, no API, no UI on day one.
3. **Stay in the loop early.** Ingest one source at a time and read every page it produces. Automate later, once you trust the workflow.
4. **The schema is the real product.** Most of your iteration time goes here. When the agent misbehaves, the fix is almost always a sharper rule.
5. **Everything is git.** Every ingest and every lint pass is a commit you can diff and roll back. This gives you a free audit trail of how knowledge evolved.

---

## Pick your corpus

The pattern is identical regardless of topic, but a *topically related* corpus produces a far richer link graph than scattered topics. Two good options:

- **A domain-knowledge corpus (recommended for you):** your trading domain — options Greeks / IV / theta decay, F&O mechanics, expiry and ban playbooks, factor investing, strategy and platform comparisons. You're an expert here, so you'll instantly spot wrong or shallow pages, which is exactly what trains the lint instinct. It also becomes a grounded knowledge layer a Tradl agent could query later.
- **A codebase corpus:** point it at a repo and let it build and maintain a wiki of the architecture, using git diffs to update incrementally. A strong variant given how much you live in a coding agent.

You can also start on a deliberately throwaway corpus (a handful of papers on one subject) just to learn the mechanics, then redo it on something you care about. Either way the roadmap below applies unchanged.

---

## Phase 0 — Substrate & schema (≈ an evening)

**Goal:** a repo that a coding agent can operate, with the rulebook written down. No code, no automation — just structure.

- [ ] Create a git repo. Lay out folders: `/sources` (immutable inputs), `/wiki` (agent-owned pages), and inside `/wiki` plan for `index.md`, `log.md`, and subfolders like `entities/`, `concepts/`, `comparisons/`, `syntheses/`.
- [ ] Write the **schema** — the heart of the whole thing. If you're using Claude Code, put it in `CLAUDE.md` (or a referenced `SCHEMA.md`) so it loads every session. Specify: the page types and what each is for; the `[[wiki-link]]` convention; the citation format (how a wiki page points back to the source that justifies a claim); the step-by-step ingest workflow; how to answer queries; and the lint rules.
- [ ] Define **page templates** with frontmatter (`title`, `type`, `aliases`, `sources`, `last_updated`) and a consistent body structure per page type.
- [ ] Decide the agent's entry points. In Claude Code, set up `/ingest`, `/query`, and `/lint` as slash commands or a single skill that reads the schema. (Worth studying for structure: the `llm-wiki` Claude Code plugin and the `nashsu/llm_wiki` desktop app — read them, don't just install them.)

**You'll learn:** that the "system" is mostly a folder layout plus a precise rulebook — and how much the schema's precision determines the agent's behavior.

**Done when:** the repo is empty of content but fully specified, and the agent reads your schema on every session.

---

## Phase 1 — The manual loop on a tiny corpus (≈ a weekend, and this is where the real learning lives)

**Goal:** get ingest → query → lint working by hand-guiding the agent on a small, related source set.

- [ ] Pick **3–5 topically related sources** (related sources produce a richer graph than scattered ones). Save them as markdown in `/sources`.
- [ ] **Ingest #1, high supervision.** Have the agent read one source, talk through the key takeaways with you, write a summary page, create the relevant entity/concept pages, update the index, and append to the log. Watch exactly which pages it touches.
- [ ] **Refine the schema** based on what went wrong — pages too broad, links inconsistent, citations missing, naming sloppy. Re-run. This iteration *is* the lesson; expect several rounds.
- [ ] **Ingest the rest, one at a time.** Watch cross-links form and — importantly — watch existing pages get *updated and reconciled*, not just appended to. Note when the agent flags a contradiction between a new source and an existing page.
- [ ] **Query.** Ask 5–10 real questions. Check that answers are synthesized from cited pages, not hallucinated. Where the wiki is thin, that tells you what to ingest next.
- [ ] **Lint pass #1.** Have the agent audit for contradictions, duplicate or overlapping pages, broken links, and single pages that have drifted into covering two concepts (split them). Fix and commit.

**You'll learn:** the difference between *appending* and *integrating* knowledge, why the schema must be explicit, and what a good entity vs. concept vs. synthesis page actually looks like.

**Done when:** you have a 15–40 page wiki that answers questions better than re-reading the raw sources, and a schema you trust.

---

## Phase 2 — Robust ingest & scaling the corpus (≈ a part-time week)

**Goal:** turn ingest from a manual conversation into a repeatable pipeline.

- [ ] **Source intake.** Write a small CLI to pull a URL or PDF into clean markdown in `/sources` (Readability + a HTML→markdown converter for the web; a PDF→markdown step for papers). Strip nav, ads, and boilerplate.
- [ ] **Source registry.** Keep a manifest of every source (id, title, url, ingested-at, status) so you never double-ingest and you know your coverage.
- [ ] **Two ingest modes.** Interactive (one source, high supervision) vs. batch (many sources, low supervision). Document in the schema when to use which.
- [ ] **Link integrity & dedup conventions.** Canonical page per entity, aliases, and a redirect note when two pages should merge.
- [ ] **Scale to ~50–100 pages.** Observe where the agent starts struggling to find the right page to update or cite — that struggle is what justifies Phase 3.

**You'll learn:** the unglamorous-but-essential plumbing of knowledge intake, and where naive "agent reads the folder" starts to break.

**Done when:** ingest is one command and the corpus is meaningfully larger.

---

## Phase 3 — Strong query / retrieval over the wiki (≈ a part-time week)

**Goal:** fast, accurate, cited answers at scale.

- [ ] **Baseline retrieval first.** Let the agent find candidate pages via the index plus filename/heading search (grep). Measure how good this actually is — it's often enough up to ~100 pages.
- [ ] **Add a search layer only when page count warrants it.** A local full-text/semantic search (e.g. `qmd`) or simple embeddings over page chunks. Benchmark it against the grep baseline so you know whether it earned its complexity.
- [ ] **Citation discipline.** Every answer cites the wiki pages it used, and every wiki page cites its sources — so any claim is traceable end to end.
- [ ] **Synthesis pages.** For recurring questions, have the agent write a standing synthesis/comparison page, so repeat queries are answered from a compiled page instead of re-synthesized from scratch. This is the compounding payoff made concrete.
- [ ] **Build a small eval set** — ~20 question/expected-fact pairs. Re-run after changes to catch regressions.

**You'll learn:** when retrieval infrastructure is actually necessary (and when it's premature), and how "compiling" answers into pages beats re-deriving them.

**Done when:** queries are fast and trustworthy, and you have a repeatable eval.

---

## Phase 4 — Automated maintenance: lint + incremental sync (≈ a part-time week)

**Goal:** the wiki stays correct as it grows without you babysitting it.

- [ ] **Lint as a standing job.** Run it on a schedule or as a pre-commit/CI check: broken `[[links]]`, orphan pages, stale `last_updated`, pages over a size/scope threshold, unresolved contradiction flags.
- [ ] **Contradiction log.** When ingest finds a claim that conflicts with an existing page, record it for human review instead of silently overwriting. Wrong claims propagate fast through links.
- [ ] **Incremental ingest via diffs.** Record the last-ingested state per source and re-ingest only the delta when a source changes. For a *codebase* corpus, store the last-ingested git commit and ingest only `git diff` since then — git becomes the maintenance engine while HEAD stays the source of truth.
- [ ] **Versioning.** Since the wiki is git, every ingest/lint is a commit — diff knowledge over time, and roll back a bad ingest cleanly.

**You'll learn:** how to keep a growing knowledge base from quietly rotting, and the elegant trick of using version control as the incremental-update mechanism.

**Done when:** the wiki self-maintains and you can audit how it changed.

---

## Phase 5 — Make it useful / connect it (optional, most Tradl-relevant)

**Goal:** the wiki feeds real consumers instead of being a side experiment.

- [ ] **Expose it as a queryable tool** — a thin API or an MCP server so other agents and apps can query the wiki.
- [ ] **Wire it into your stack.** Let a Tradl agent (e.g. the screening agent or position copilot) read the relevant concept/entity pages before it reasons, so domain knowledge is explicit and maintained rather than buried in prompts.
- [ ] **Optional viewer.** Skip building a UI — open the `/wiki` folder in Obsidian. It renders `[[links]]` and the knowledge graph for free.
- [ ] **Multi-corpus separation** if you end up wanting more than one wiki (e.g. a trading-knowledge wiki and a codebase wiki).

**You'll learn:** how an LLM wiki stops being a notebook and becomes a grounded, maintainable knowledge layer for downstream agents.

**Done when:** something other than you is consuming the wiki.

---

## Recommended minimal stack

- **Storage:** git + plain markdown. That's the substrate.
- **Agent:** Claude Code (or any coding agent) driven by `/ingest`, `/query`, `/lint` commands or a single skill, all reading your schema.
- **Viewer:** Obsidian (optional, free graph + backlinks).
- **Search:** none at first; add a local full-text/semantic layer (`qmd` or simple embeddings) only at ~100+ pages.
- **Distribution (Phase 5):** a small MCP server to expose the wiki to other agents.

## References worth reading (study, don't just install)

- Karpathy's original gist — the canonical spec for the pattern.
- `nashsu/llm_wiki` — a desktop-app implementation; good for seeing the ingest/query/lint pipeline made concrete.
- `llm-wiki.net` — a Claude-Code-native take (plugin + portable `AGENTS.md`); useful since you already work in a coding agent.
- The "LLM Wiki for a codebase" write-up (dev.to) — the git-diff incremental-maintenance idea in Phase 4.

---

## Suggested order if you only have a weekend

Do Phase 0 and Phase 1 end to end on 3–5 related sources. That alone gives you the full ingest → query → lint loop and 80% of the conceptual understanding. Phases 2–5 are about scale, robustness, and integration — add them once the core loop feels natural.
