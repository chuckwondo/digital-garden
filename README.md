# 🌱 Digital Garden

A personal, evolving collection of interlinked notes and essays, organized by *idea* rather than by date, tended over time and kept in whatever state they happen to be in. Not a blog: notes are meant to grow rather than ever be "finished."

New here? Start with [How this garden works](content/how-this-garden-works.md) for the growth stages and how notes connect.

## Notes

- 🌿 [How this garden works](content/how-this-garden-works.md): how these notes are grown, staged, and linked.
- 🌿 [Correct by Construction](content/correct-by-construction.md): refined types, smart constructors, and where invariants come from. When only *some* values of a type are valid, make the guarantee come from *how the value is built*, not from checking it afterward. (In progress; see its STATUS block.)
- 🌿 [Python test suite structure](content/python-test-suite-structure.md): where tests live, whether `tests/` is a package, and how to share code across test files, all downstream of your pytest import mode.

## Running locally

Preview the site with live reload:

```sh
npx quartz build --serve
```

Then open <http://localhost:8080>; Quartz watches `content/` and rebuilds on save. On a fresh clone, run `npm run install-plugins` once first if a plugin is missing, and add `--port 8081` if 8080 is taken.

## Built with

[Quartz](https://quartz.jzhao.xyz), a static-site generator built specifically for digital gardens. It ships the features that make a garden a garden (`[[wikilinks]]`, automatic backlinks, a graph view, tag pages, and full-text search) as defaults rather than plugins, while notes stay plain Markdown that can also be authored in [Obsidian](https://obsidian.md) or [Foam](https://foambubble.github.io/foam/) over the same `content/` folder. The result is a self-owned static site deployed to GitHub Pages.

General-purpose generators (Astro, Eleventy, Hugo) can be bent to the same shape, but each garden feature then becomes something to wire up and maintain; a hosted option (Obsidian Publish) would trade away the self-owned static output and the custom theme. Quartz gives all of it out of the box, so it stays.
