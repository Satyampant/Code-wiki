---
description: Answer a question from the wiki, with citations.
argument-hint: <your question>
---

Run the **query workflow** defined in `CLAUDE.md` §7 to answer: `$ARGUMENTS`

Find candidate pages via `wiki/index.md` and search, read them, follow `[[links]]` as needed, and
synthesize an answer **only from the pages** — never invent facts. Cite which wiki pages you used. If
the wiki can't support a good answer, say so and tell me what to ingest next to close the gap. Don't
write pages during a query unless I ask.
