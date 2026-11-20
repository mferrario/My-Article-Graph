# Article Graph

My evolving practice for working with AI, through two lenses:

- **sdlc**: my own software development lifecycle for building software with AI agents.
- **ai-pm**: how an AI product manager designs systems, evals included.

Article notes are the evidence. `playbook.md` is the product: what I actually do now.

## Layout

| Path | What it holds |
|---|---|
| `articles/` | One note per article, `YYYY-MM-DD-slug.md`, made from `articles/_template.md`. |
| `articles/sources/` | Downloaded originals, only for paywalled pieces and PDFs. |
| `index.md` | The graph: every article, plus concepts, frameworks and topics linking to notes. |
| `playbook.md` | Current practice for both lenses, split into sections with stable IDs, plus a changelog. |

## Workflow

1. **Add an article.** Send a link (or drop a PDF/paywalled copy into `articles/sources/`) and run `/cliff-notes`. It writes the note, updates `index.md`, and proposes playbook changes.
2. **Tag it against the playbook.** The note's `playbook:` frontmatter lists the section IDs it `confirms`, `adds` to, `contradicts` or `supersedes`. The "What this changes" section explains each one.
3. **Decide on the playbook.** Nothing in `playbook.md` changes until I say which proposed changes to apply. Each applied change cites its note and gets a changelog row.
4. **Supersede, never delete.** When a newer note replaces an older one, the old note gets `status: superseded` and `superseded_by: <slug>`.

## Querying the graph

What challenges a section (the "what's gone stale" check):

```sh
grep -lE "(contradicts|supersedes): \[.*sdlc-verify" articles/*.md
```

Everything that touches a section:

```sh
grep -l "sdlc-verify" articles/*.md
```

Sections no note has touched yet: compare the IDs in `playbook.md` against `grep -ho "\b\(sdlc\|pm\)-[a-z]*" articles/*.md | sort -u`.

## Conventions

- Plain Markdown, no build step.
- Links over downloads. Download only what a link can't reliably reach.
- Date anything that can expire ("as of <month year>").
- Concept slugs are lowercase, hyphenated, singular, and reused before new ones are created.
