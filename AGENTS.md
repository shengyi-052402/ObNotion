# ObNotion Wiki Schema

This vault is maintained as a persistent wiki, not a loose note dump.
Default output language is Simplified Chinese unless the user explicitly asks otherwise.

## Architecture

There are three layers:

1. `raw/` contains immutable source material. The LLM may read these files but must not rewrite them in place.
2. `wiki/` contains LLM-maintained knowledge pages. The LLM owns summaries, synthesis, cross-links, and organization here.
3. `system/` contains process documents and templates that define how the vault is maintained.

## Directory Conventions

- `Clippings/`: newly captured sources waiting for processing
- `raw/sources/`: processed source notes kept as source-of-truth records
- `raw/assets/`: shared downloaded attachments and images
- `wiki/总览.md`: top-level map of the subject area
- `wiki/索引.md`: catalog of wiki pages with one-line descriptions
- `wiki/日志.md`: append-only operational log
- `wiki/entities/`: pages for people, organizations, tools, books, products
- `wiki/concepts/`: pages for concepts, methods, claims, themes
- `wiki/topics/`: broader syntheses, comparisons, and ongoing analyses
- `wiki/queries/`: answers worth preserving from chat sessions
- `system/templates/来源笔记模板.md`: template for source notes
- `system/templates/知识页模板.md`: template for wiki pages

## Operating Rules

- Prefer updating existing wiki pages over creating duplicates.
- Every substantial wiki page should link to at least one related page.
- When ingesting a source, touch all affected pages in one pass when practical.
- Keep raw-source facts separated from synthesized claims. If a statement is an inference, say so.
- Use markdown links with vault-absolute paths when possible because the vault is configured for absolute links.
- Do not delete user-authored raw material unless explicitly asked.
- Avoid empty stub pages. Create a page only when it can hold durable value.
- When creating new notes, prefer Chinese filenames and Chinese section headings.
- Summaries, indexes, and logs should be written in concise Chinese.

## Standard Workflows

### Ingest

1. Read a source from `Clippings/` or an explicitly provided URL/file.
2. If useful, create or update a source note in `raw/sources/`.
3. Update `wiki/总览.md` if the source changes the high-level picture.
4. Update relevant pages under `wiki/entities/`, `wiki/concepts/`, and `wiki/topics/`.
5. Update `wiki/索引.md`.
6. Append an entry to `wiki/日志.md`.

### Query

1. Read `wiki/索引.md` first to locate relevant pages.
2. Read the minimum set of pages needed to answer.
3. If the answer produces durable synthesis, save it under `wiki/queries/` and add it to the index.
4. Log the operation in `wiki/日志.md` when it materially changes the vault.

### Lint

Periodically check for:

- stale claims
- contradictory pages
- orphan pages
- concepts without dedicated pages despite repeated mentions
- missing cross-links
- repeated notes that should be merged

## Page Standards

- Use concise YAML frontmatter when it improves navigation.
- Keep the first paragraph readable as a standalone summary.
- End source-derived notes with a `## Sources` section when citations matter.
- Use `## Related` for important outgoing links if they are not obvious in body text.

## Index Format

`wiki/索引.md` is grouped by section. Each entry should look like:

- `[Page Name](path/to/page.md)` - one-line summary

## Log Format

Each entry in `wiki/日志.md` should begin with:

`## [YYYY-MM-DD] operation | title`

Accepted operations include `ingest`, `query`, `lint`, and `reorg`.
