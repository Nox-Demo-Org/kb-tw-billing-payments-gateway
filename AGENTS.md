---
type: Agent Guide
title: AGENTS.md — how to use this knowledge base
description: This repository is the knowledge base for tw-billing/payments-gateway, maintained by NoX.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-payments-gateway/blob/main/AGENTS.md
tags:
- payments-gateway
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:45:47Z'
---

# AGENTS.md — how to use this knowledge base

This repository is the knowledge base for **tw-billing/payments-gateway**, maintained by NoX. It is an **Open Knowledge Format (OKF) bundle**, following Google Cloud's open specification for knowledge that agents and people share (https://github.com/GoogleCloudPlatform/open-knowledge-format), extended by NoX for knowledge that spans applications.

## Layout
- `index.md`: the architectural overview; its frontmatter declares `okf_version`, and it ends with the contents.
- `summaries/`: interface and API references (e.g. `api-spec.md`).
- `concepts/`: flows, lifecycles and cross-cutting mechanisms.
- `entities/`: components, services and data models.
- `decisions/`: architecture decision records (ADRs).
- `*/index.md`: each folder's pages with their one-line descriptions (OKF progressive disclosure).
- `log.md`: every generation and sync, grouped by date, newest first.
- `.nox/brief.md`: a compact brief (under 4,000 characters) for coding agents.

## Rules for agents
1. **Frontmatter**: every page starts with OKF YAML frontmatter. `type` is required; `title`, `description`, `resource`, `tags`, `sources` and `generated` describe the page. Keep any keys you don't recognise.
2. **Links**: pages link with `[[path/slug|Title]]` wikilinks, and to other applications' knowledge bases with `[[kb:other-app/path|Title]]` (NoX's cross-application extension). Folder indexes use OKF's bundle-absolute Markdown links (`/entities/x.md`).
3. **Log changes**: when you change or add a page, add an entry to `log.md` under today's date.
4. **Preserve anchors**: never delete `<!-- anchor: ... -->` comments; they tie a page to the source lines it describes.
