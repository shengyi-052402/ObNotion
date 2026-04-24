# ObNotion

An Obsidian vault organized as a persistent LLM-maintained wiki.

## Structure

- `raw/inbox/`: unprocessed captures and incoming material
- `raw/sources/`: processed source notes
- `raw/assets/`: downloaded images and attachments
- `wiki/`: synthesized knowledge pages
- `system/templates/`: note templates
- `AGENTS.md`: maintenance rules for Codex and other LLM agents

## Core Files

- `wiki/overview.md`: high-level map of the vault
- `wiki/index.md`: catalog of maintained wiki pages
- `wiki/log.md`: chronological operation log

## Workflow

1. Add source material to `raw/inbox/`.
2. Process it into `raw/sources/` and relevant pages under `wiki/`.
3. Update `wiki/index.md` and append to `wiki/log.md`.
