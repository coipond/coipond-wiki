# CoiPond Wiki (source)

This repository is the **source** for the [CoiPond](https://coipond.ai) documentation wiki. The pages here are authored, reviewed, and linted as normal files, then published to the rendered wiki.

CoiPond is self-hosted infrastructure for orchestrating autonomous AI coding sessions: it triggers coding-agent runs from events, isolates each one in its own Incus container, and chains them into multi-step workflows.

## Reading the docs

Start at [Home](Home). Every page links to its neighbors by bare page slug, which resolves in the rendered wiki.

## Editing

- Edit the Markdown files in this repository and open a pull request. Do not edit the rendered wiki directly - changes there are overwritten on the next publish.
- Follow the existing style: HTML tables (not Markdown tables), fenced code with a language, Title Case headings capped at H3, US English, and plain hyphens (never em-dashes).

## Layout

- `Home.md` - the wiki landing page and table of contents.
- `_Sidebar.md` - the navigation sidebar.
- One Markdown file per page, named by its slug (for example `Getting-Started.md`).
