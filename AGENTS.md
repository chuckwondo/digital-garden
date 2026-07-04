# AGENTS.md

Guidance for tending this garden (for AI coding agents and for me). The README is the reader-facing front door; this file is the source of truth for conventions, tooling, and workflow, so a fresh session starts warm. A shorter reader-facing summary of these conventions lives in the [[How this garden works]] note.

This is a **digital** garden: interlinked notes and essays, organized by *idea* rather than date, meant to grow rather than ever be "finished."

## Tooling and structure

- **Publisher: [Quartz](https://quartz.jzhao.xyz) v4**, a static-site generator built for digital gardens (wikilinks, backlinks, graph, tag pages, search).
- **Author** in plain Markdown with any editor: Foam in VS Code, or Obsidian pointed at the same folder.
- **Notes live in `content/`**, flat, as `content/<Title>.md`. Add deeper folders only when the link graph actually demands them.
- **Preview** locally with `npx quartz build --serve`.
- (The Quartz scaffold is being set up this session; until then the notes sit at the repo root.)

## Note conventions

- **One idea per note; the title is the idea** (a claim or concept, not a category).
- **Lightweight YAML frontmatter:**

  ```yaml
  ---
  title: Correct by Construction
  tags: [architecture, types]   # topic tags -> Quartz /tags pages
  stage: budding                # seedling | budding | evergreen
  planted: 2026-07-03           # first committed
  tended: 2026-07-04            # last meaningful update
  ---
  ```

  `title` and `tags` are Quartz-native; `stage`, `planted`, and `tended` are custom garden fields.
- **No `# H1` in the body:** Quartz renders the frontmatter `title`. Open with the *italic tagline*, then the prose.
- **`stage` is the single source of truth for maturity;** the README index and its 🌱 / 🌿 / 🌳 emoji derive from it.
- **Link liberally with `[[wikilinks]]`,** including to notes that do not exist yet: a dangling link is a planting marker for a future note, not an error. Wikilinks resolve by note title / filename.
- **In-progress notes** may carry a `## STATUS` block near the top recording what is settled and what is next.

## Maturity ladder

- 🌱 **seedling:** rough, just planted; may be incomplete or wrong.
- 🌿 **budding:** developing and taking shape; still evolving.
- 🌳 **evergreen:** mature and stable (but never truly done).

## Prose

- **Soft-wrap prose:** one line per paragraph, no hard line breaks mid-paragraph. Editors reflow visually, Quartz renders it identically, and diffs stay per-paragraph instead of churning on every rewrap.
- **Prefer a colon, parentheses, a comma, or splitting into shorter sentences over an em-dash.** If a dash is genuinely the right punctuation, use `--`, never the Unicode em-dash character.
- **`e.g.` and `i.e.` are always followed by a comma.**

## Workflow

- **Plant a note:** create `content/<Title>.md` with frontmatter (`stage: seedling`), write the tagline and body, add liberal `[[links]]`, add a line to the README index, and commit `Plant the garden: <Title>`.
- **Tend a note:** edit, bump `tended`, advance `stage` when earned, update the README hook if it changed, and commit `Tend the garden: <Title>`.
- **Structural changes** (Quartz config, new folders) go in their own commit, separate from note content.

## Backlog

Deferred and queued work. Once this repo has a GitHub remote, migrate these to GitHub issues and point here to them instead.

- **Phase 4: deploy to GitHub Pages.** Create the repo/remote, add the Pages Actions workflow, and resolve the base URL (the `cname` plugin emits a `CNAME` file, which a project page at `chuckwondo.github.io/digital-garden` does not want: disable that plugin or set a custom domain).
- **Tend [[Correct by Construction]]: body prose retrofit.** Reflow the body to soft-wrap and replace its em-dashes per the Prose conventions. Deliberately deferred as a large, voice-sensitive edit; the frontmatter is already aligned.
- **Spin off the [[Python test suite structure]] hub.** Its five Related wikilinks are dangling planting markers (`pytest import modes`, `src layout`, `sharing code between test files`, `tests as a package or not`, `pytest fixtures vs helper functions`). Plant each as its own note when it has enough real content, not as an empty stub.
