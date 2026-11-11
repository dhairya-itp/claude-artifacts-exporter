# Part 3 — Recreate in the new organization

Goal: each exported artifact exists again in the new organization, as the same type, looking and working the same. The person gets a `New links.csv` and a table saying which ones are all good.

Read `manifest.md` first: it describes the export you are reading from and the progress file you write.

## 0. Before creating anything

1. **Check the organization.** Run `claude auth status` and compare its `orgId` with `export-info.json`. If they match, stop: you are still in the old organization, and creating anything would only make duplicates there. Explain how to switch (SKILL.md). If the command isn't available, list the person's artifacts. If the old links show up as theirs, you're still in the old organization.
2. **Read the export**: `My artifacts.csv`, `export-info.json` and every `manifest.json`. Note any artifacts whose export is `partial` or has no manifest. Recreate what was saved and carry the gaps into the final table. Don't recreate `empty` ones (blank dashboards, untouched decks). List them in the table as "skipped: it was empty", and offer to make a blank one if the person wants it.
3. **Resume?** If `recreate-progress.json` exists, artifacts in state `verified` are done. Finish `created` or `filled` ones, filling the existing new artifact rather than making another. Check the progress file's `org_id` too. If it differs from the current organization, ask the person which organization they mean before going on.
4. **Check types.** Call `action: "list", scope: "types"` in this organization. Every type the export needs (other than `Page`) must be listed by name. If any is missing, **ask the person before doing anything else**. Name the artifacts affected and offer options in plain words: rebuild them as plain pages (they look the same but lose the type's editing tools), skip them for now, or wait while an admin enables the type. Recommend one.
5. **Check connections.** Some artifacts call connectors. Look at each manifest's `capabilities.mcp.servers`, and at the data sources in a dashboard's saved datasets. List the connectors involved (Google Drive, BigQuery, …). These need connecting again in the new organization. Don't rewrite queries to avoid them. Mention them now, and again in the final table.
6. **Mentions of colleagues** in Docs (`mentions_of_others`) can't carry over as they are: people's ids differ between organizations. The default is to keep them as plain names. If there are any, ask once whether the person wants them turned back into @-mentions (that needs each colleague's email, which they can provide).

## 1. Order

Design systems first, because decks and designs point at them. Then the rest in `My artifacts.csv` order. Keep a map of old link → new link as you go (it lives in `recreate-progress.json`).

## 2. Recreate each artifact

General rules:

- **Rebuild from the export, don't redesign.** The person wants the same artifact. Don't "improve" layouts, rewrite text, change fonts or fix typos you notice.
- **Work on copies.** Copy the files you'll change into `_rebuild/<NNN>/` (with the same relative paths) and edit those. Never edit the export.
- **Record the new link immediately** (manifest.md, *recreate-progress.json*), then append to `New links.csv`.
- **Uploaded assets get new urls.** Upload them to the new artifact, then replace every old `/_blob/<old id>` (and any absolute form of it) in the rebuilt files and data with the url the upload returned. Use the url exactly as returned. Keep the map in `asset_map`. Upload with `action: "publish", url: <new>, asset: true, file_paths: [...]`, up to 25 per call. CSV, Markdown, JSON and plain-text assets go one per call with `file_path`.
- **Links to other migrated artifacts** (`references_to_other_artifacts`): swap in the new link if that artifact already exists here. Leave the rest for the fix-up pass (step 3).

### 2a. Plain pages (`Page`)

1. Copy `page/` to `_rebuild/<NNN>/`.
2. First publish: `file_path: _rebuild/<NNN>/index.html`, with every other file in `files` (same published paths, `root: _rebuild/<NNN>`), plus an `icon` (one generic word) and a short `description`. If the manifest has `capabilities`, load the `artifact-capabilities` skill first and pass the same declaration. If the new organization refuses a capability, tell the person what that part of the page did and ask whether to publish without it.
3. If there are assets: upload them, rewrite the `/_blob/` references in the rebuilt files, and publish again with the same `file_path` (same url).
4. If there is saved data: write it back with ArtifactData `action: "batch"` (up to 50 writes, 1 MiB per batch, no `if_version` because these are new documents). Look at one saved document file first. If it wraps the fields with read metadata (such as `version` or an id), write only the document's own fields, preparing those files in `_rebuild/<NNN>/data/`. Rewrite `/_blob/` references inside the data too.

### 2b. Typed artifacts (Slides, Design, Design System, Data, Motion, Whiteboard, …)

1. Create from the type in this organization: `action: "publish", type_url: <this org's url for the type name>, title: <name>, auto_open: "after_first_write"`. Use the type_url from this organization's types listing, not the old one in the manifest. Don't pass `icon`, because the type sets its own.
2. Read the instructions the create result returns. They say where the type keeps content (data files published to the artifact, documents in its store, or both) and which reference pages to read first. Follow them, using the export as the content: the files in `page/` (usually `project/…`) and the documents in `data/`.
3. Publish the content files to the new url with `files` and `root: _rebuild/<NNN>/page`, so each file lands at the same path it had. Never publish the type's own files (`artifact-type/…`, `SKILL.md`, `index.html`), because the type supplies them.
4. Upload assets, rewrite their references in the rebuilt files and data, and publish again.
5. Write store documents with ArtifactData batches, as for pages. Follow the type's rules for its store. For example, the Data type says never to write who shares a connector, and to stamp the datasets you write. Its instructions win over a literal copy.
6. **Design systems**: the same steps, then record old → new so the decks and designs that follow can point at the new one. If a deck or design names its design system (in `project/deck.json`, `canvas.json` or similar), swap in the new link.
7. A dashboard whose datasets query a connector won't show numbers until that connector is connected here. That's expected: report it, don't work around it.

### 2c. Claude Docs and Sheets

Load the Claude Docs instructions first. Use the docs skill if one is listed; otherwise call `guide(items: ["topic.index"])`. Before the calls they cover, also load `topic.tabs` (more than one tab), `topic.charts` and `topic.diagram` (any widgets), and `topic.uploads` (any images or files). Load `topic.charts` in a call of its own.

1. **Create the doc** with the connector's `batch` call, named as before. Build each tab's content from the export. `content.md` gives the text, with a placeholder where each chart was. `content.xml` is the authority on what the markdown can't carry, and where it sits:
   - date chips → `{"type":"date","value":"YYYY-MM-DD"}`
   - the person's own mention → `{"type":"mention","user":"me"}`
   - colleagues' mentions → plain names, unless the person asked for @-mentions and gave emails
   - dropdowns → create each enum from `enums.json`, with the same name and options, in the same batch before the prose. Then one dropdown chip per cell with the same `index`.
   - check lists keep their ticked state, and tables keep their columns and rows
2. **Tabs**: recreate every tab with its name and order (`topic.tabs`).
3. **Charts and diagrams**: for each saved widget, create the widget node with the saved `code` (and `sources`) as `topic.charts`/`topic.diagram` describe, and put its embed chip back in the same place with the same caption. Don't redraw a chart from scratch: the saved code is the chart.
4. **Images and files**: upload each one from `doc/uploads/` and place it where it was (`topic.uploads`).
5. Don't recreate comments (`comments.json` is the person's reference copy). The table's last column notes that old comments stayed behind.
6. Write each tab in as few calls as the connector allows. The docs guide's advice to fill section by section is for a doc someone watches being written; here the person wants the whole batch done. Don't open every new doc on screen, because the final table carries the links.

## 3. Fix-up pass

When every artifact exists, go back to any whose `references_to_other_artifacts` point at artifacts created later. Swap in the new links, republish (or update the doc), and record it.

## 4. Check each one against the export, then report

For each artifact, read the new one and compare it with its manifest:

- **Pages and typed artifacts**: list its files and assets. The same content files exist, the asset count matches, and no rebuilt file still contains an old `/_blob/` id or an old artifact link that has a new one. Database collections have the same document counts.
- **Docs and Sheets**: read the doc and each tab's outline. Tab count and names match, and the embeds, tables, dropdowns and images per tab match the counts in the export's `content.xml`.

Set `state: "verified"` when it matches. Otherwise record the differences in plain words. Then show:

| Artifact | New link | All good | What's different |
|---|---|---|---|
| Acme brand (design system) | https://claude.ai/artifact/… | yes | — |
| Dashboard | https://claude.ai/artifact/… | no | Needs BigQuery connected before its numbers show |

Close with what the person does next, in a few short lines: open a few new links and compare them with the originals; share each one again with the same people (Share button on claude.ai); send `New links.csv` to colleagues; reconnect any connectors named above. Don't delete anything in the old organization; keep the backup folder. Comments and past versions don't carry over.

## Fixing one artifact

The person says a new copy doesn't match ("the logo is missing on slide 2"):

1. Find the artifact's row in `My artifacts.csv` (by name; ask if two match), its folder and manifest, and its new link in `recreate-progress.json` or `New links.csv`.
2. Read the new artifact (and, for the slide, page or tab in question, the file or section) and compare it with the export on exactly the point raised. Look for the real cause: an asset not uploaded, a `/_blob/` link not rewritten, a widget not embedded, a file at the wrong path.
3. Fix the **new** artifact only: republish it to the same url, or update the doc. Never touch the old one.
4. Tell the person in a sentence or two what was wrong and what you changed. Update the progress file.
