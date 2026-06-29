---
description: Audit the wiki for broken links, drift, duplicates, and contradictions.
---

Run the **lint workflow** defined in `CLAUDE.md` §8 across the whole `wiki/`.

Check for: broken `[[links]]`, orphan pages, bloated/scope-drifted pages (split them), duplicate or
overlapping pages (merge + redirect note + alias), contradictions (**flag for me, don't auto-resolve**),
stale `[src: file:lines]` citations, missing frontmatter/sections, and stale `last_updated` dates.

Report your findings, apply the safe fixes, list anything you've flagged for my judgment, append a
`lint` entry to `wiki/log.md`, and finish with a `lint: <summary>` commit.
