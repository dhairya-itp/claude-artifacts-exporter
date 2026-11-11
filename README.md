# claude-artifacts-exporter

A [Claude skill](https://code.claude.com/docs/en/skills) that moves your claude.ai artifacts from one Claude organization to another, for example from Claude Enterprise to Claude Team.

It runs in Claude Code (the **Code** tab in the Claude desktop app, or the terminal) and works in three parts:

1. **Export.** Saves every artifact you own into a local folder. For each one it saves the published files, uploaded images and fonts, and the artifact's saved database. For Claude Docs it also saves each tab's structured content, the code of every chart and diagram, dropdown lists, and a Word copy. Each artifact gets a `manifest.json` and a plain-words `NOTES.md`, and the list goes in `My artifacts.csv`. Nothing on claude.ai is changed.
2. **Check the organization.** Confirms you've switched to the new organization before anything is recreated.
3. **Recreate.** Rebuilds each artifact in the new organization as the same type. Design systems go first, images are re-uploaded and their links rewritten, and charts are restored from their saved code. It writes `New links.csv` (name, old link, new link) and ends with a table showing which artifacts match the export.

It can also pick up where a stopped session left off, and fix a single recreated artifact that doesn't match its original.

## Install

- **claude.ai (recommended):** zip this folder (with `SKILL.md` inside a `claude-artifacts-exporter/` folder) and upload the zip in the skills settings on claude.ai or under **Customize** in the desktop app. Skills belong to the organization you're signed into, so upload it in both the old and the new organization. An organization admin can also add it for everyone.
- **One computer only:** copy this folder to `~/.claude/skills/claude-artifacts-exporter/`.

## Use

Make an empty folder, open it in Claude Code, and say:

- in the old organization: *"Export my artifacts"* (or name a few: *"export just these two: …"*)
- after switching: *"Which Claude organization am I using?"*, then *"Recreate my artifacts"*
- if a session stopped: *"My last session stopped before it finished. Carry on."*
- if a copy is off: *"The new copy of 'X' doesn't match the original: the logo is missing on slide 2."*

Click **Allow** when Claude asks to read artifacts or save files.

## What doesn't carry over

Each artifact gets a new link. Comments and version history stay behind (the export keeps a copy of Doc comments for reference). Sharing has to be set up again, and connectors such as Google Drive or BigQuery need connecting again in the new organization.

## Status

The export part has been tested on plain pages, Slides decks, Claude Docs with diagrams, and Data dashboards. The recreate part follows the same tool contracts but hasn't yet been run end to end against a second organization. Try it on a few artifacts before you rely on it, and keep your backup folder.

## Layout

```
SKILL.md                 when the skill triggers, which part to run, shared rules
references/export.md     Part 1, step by step for each kind of artifact
references/recreate.md   Part 3, plus fixing a single artifact
references/manifest.md   the backup folder layout and file formats both parts share
evals/                   test cases (claude plugin eval format; fill in your own links)
```
