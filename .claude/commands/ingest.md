---
description: Read a source file and create/update every wiki page it touches.
argument-hint: <path under sources/, e.g. sources/auth/session.py>
---

Run the **ingest workflow** defined in `CLAUDE.md` §6 on: `$ARGUMENTS`

Follow every step: read the source fully, discuss key takeaways with me first (high supervision),
search the existing wiki for pages it touches, then create or **integrate** (don't just append) the
relevant pages from the `templates/`. Add `[[wiki-links]]`, cite every claim with `[src: file:lines]`,
flag any contradictions for me instead of overwriting, update `wiki/index.md`, set `last_updated`,
append an entry to `wiki/log.md`, and finish with a `ingest: <path>` commit.

Never modify anything under `sources/`.
