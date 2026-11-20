# Article Graph

My evolving practice for working with AI. Article notes are the evidence; `playbook.md` is the product: what I actually do now.

The rules for filing notes (template, lenses, index layout, playbook layout, graph rules, staleness) live in the project's **`conventions.md`**. The `cliff-notes` skill reads it before filing, so this README only describes the layout and workflow.

## Lenses

- `sdlc`: my software development lifecycle for building with AI agents.
- `evals`: how an AI product manager designs evaluation systems.
- `industry`: where the field is heading (models, vendors, adoption, economics).

## Layout

| Path | What it holds |
|---|---|
| `articles/` | One note per article, `YYYY-MM-DD-slug.md`. |
| `articles/sources/` | Downloaded originals, only for paywalled pieces and PDFs. |
| `index.md` | The graph: every article, plus concepts, frameworks and topics linking to notes. |
| `playbook.md` | Current practice (SDLC, evals), the assumptions about the field it rests on, and a changelog. |

## Workflow

1. **Add an article.** Share a link (or a PDF / paywalled copy) and run `/cliff-notes`. In this project it files the note and updates `index.md` by default.
2. **Review the proposed playbook changes.** Each note's "What this changes" section says what it Confirms, Adds, Contradicts or Supersedes. Nothing in `playbook.md` changes until I approve a specific change; each applied change cites its note and gets a changelog row.
3. **Supersede, never delete.** A replaced note gets `status: superseded` and `superseded_by: <slug>`.

## Finding what's gone stale

```sh
# Notes that push back on current practice
grep -l -E '^- \*\*(Contradicts|Supersedes)\*\*' articles/*.md

# Notes already replaced
grep -l '^status: superseded' articles/*.md

# Industry notes that may shift the assumptions section
grep -l '^lens:.*industry' articles/*.md
```
