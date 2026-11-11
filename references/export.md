# Part 1 — Export (save a copy)

Goal: every artifact the person owns (or every one they named) ends up in its own numbered folder. Each folder has enough content to rebuild the artifact elsewhere, a `manifest.json` that says exactly what was saved, and a `NOTES.md` for the person. Nothing on claude.ai changes.

Read `manifest.md` first for the folder layout. This file covers how to fill it.

## 0. Before starting

1. Check the organization (SKILL.md, *Which organization am I in*) and write `export-info.json`.
2. **All, or some?** The usual request is everything the person owns. If they name particular artifacts ("just these two", one link), export only those. Set `"scope": "subset"` in `export-info.json`, number them newest first among themselves, and check only those at the end. A later full export into the same folder keeps those numbers and appends the rest.
3. Look at the folder. If `My artifacts.csv` and numbered folders already exist, this is a **resume**. Keep the existing numbers. Skip a folder whose `manifest.json` says `"status": "complete"` (or `"empty"`). Redo folders that are `partial` or have no manifest; overwriting their files is fine. Append any artifacts that aren't in the CSV yet, using the next numbers.

## 1. Inventory

1. List the person's artifacts with the Artifact tool: `action: "list"`, default scope (mine), `limit: 200`. Follow `page_token` until there are no more. Keep only `(mine)` rows, which carry the name, link and "updated" date. Shared artifacts are not theirs to move.
2. Find each artifact's type. Call `action: "list", scope: "types"` for the type names and `type_url`s. Then, for each type, call `action: "list", type: "<type name>"` and keep only links that are in the person's own list from step 1. Type listings carry no dates and also show org-shared artifacts. Any artifact that appears under no type is a plain `Page`.
3. Sort newest first by the "updated" date, number from 1, and write `My artifacts.csv`. Tell the person how many artifacts you found and roughly how long it will take (a minute or two each). Then begin.

## 2. Getting tool results onto disk exactly

Some reads save straight to disk (`out_dir`). Others return their content into the conversation: a Doc's structured read, its exports, and chart/diagram nodes. Those must reach disk byte for byte, so never retype or summarize them.

- **Large results** come with a note like "Output too large… saved to `<path>`". Copy that file, or pull the text out of it with a short script. Don't paste it by hand.
- **Small results** shown inline: write them with the Write tool, copying the content exactly as returned.
- **Base64** (Doc exports): never decode it yourself. If it arrived in a saved file, decode from that file with a script. If it's inline and short, write the base64 text to a scratch `.b64` file and decode it with the shell (`base64 -d in.b64 > out`; on Windows PowerShell, `[IO.File]::WriteAllBytes(out, [Convert]::FromBase64String((Get-Content in -Raw)))`).
- Afterwards, check that the saved file's size matches what the tool reported, where it reports one.

## 3. Export each artifact

Work through the CSV in order. Finish one artifact (files, then manifest, then notes) before starting the next. Independent reads within one artifact (several assets, several widgets) can go out together in one turn.

### 3a. Every artifact (any type)

1. **Read it**: `action: "read", url` (no `out_dir`; a read without `path`/`paths` doesn't take one). From the header, take the version and file count. The `<artifact-stored-declaration>` block is the artifact's runtime capabilities: copy that JSON into `manifest.capabilities` exactly as shown, or `null` if none are declared.
2. **List its files**: `action: "list", scope: "files", url`.
3. **Save its files**: `action: "read", url, paths: [...], out_dir: "<folder>/page"`, with up to 256 paths per call. The files keep their published paths.
   - For a **typed** artifact, skip the type's own app files: everything under `artifact-type/`, the root `SKILL.md` and the root `index.html`. They are the type's viewer, the same for every artifact of that type, and the new organization provides its own. Count all of them in `skipped_type_files`. Save everything else, usually under `project/`.
   - For a **Page**, save every file.
4. **Save its uploaded assets** (images, logos, fonts, PDFs that the page references as `/_blob/<id>`). Call `action: "list", scope: "assets", url`, and follow `after` while the listing says there are more.
   - If the listing refuses because the artifact has no asset store (`capability_disabled`), there are simply no uploads: record `assets: []` and move on. The refusal may suggest republishing with the capability added. **Never do that**, because it would change the person's artifact.
   - Save each asset with its own call: `action: "read", url, path: "<asset id>", out_dir: "<folder>/assets"`. Asset ids don't work in a `paths` list. Send these calls together in one turn rather than one turn each. The read names each file `<id>.<ext>`.
   - Record each asset's id, url, type, size and sha256 from the listing. If `sha256sum` or `shasum -a 256` is available, check the saved files against those hashes.
5. **Save its database**, if the capabilities declaration has a `db` entry. No tool lists an artifact's collections, so collect the names from these sources:
   - **The type's own instructions**: the artifact's `SKILL.md`, read once per type, and its "store" or "database" section.
   - **The type's thumbnail settings**: `artifact-type/thumbnail/thumbnail.json`, which often names the content collections.
   - **The declaration**: the `db.rules[].path` and `db.audit` entries. A rule on the root path `""` is a permission, not a collection you can list.
   - **The page's code**: for a Page, the saved files. For a typed artifact, search the type's code without saving it, by reading it into a scratch folder.

   Known cases:
   - **Data** (dashboards) keeps everything in its store, including the page itself (`files/index.html`). The collections are `dash` (`meta`, `params`, `look`, `settings`), `datasets`, `files`, `shared-results`, `drafts`, `versions` and `version-docs`.
   - **Slides** keeps speaker-note data under `notes`.

   Read each collection with ArtifactData `action: "list", collection, query: {limit: 1000}, out_dir: "<folder>/data"`, and follow `next_cursor` until there is none. Paths alternate collection/document (`dash/meta` is document `meta` in collection `dash`). Look in saved documents and code for subcollections.

   Skip `data/users/…`. Other viewers' areas can't be read, and the person's own holds page state, not content. Save `data/users/me` only when the code shows real content there. Record every collection you read in the manifest, **including ones with 0 documents**: an empty list writes no files, and that's fine.
6. **References to other artifacts**: search everything you saved for `claude.ai/artifact/` and `claude.ai/code/artifact/` links. List them in `references_to_other_artifacts`.

### 3b. Claude Docs and Sheets (extra steps)

A Doc's or Sheet's content lives in the Claude Docs service, not in the artifact's files, so steps 3a.3 and 3a.5 find only type files there. Still run 3a.1, 3a.2 and 3a.4, then do the following, saving results as section 2 describes.

Load the Claude Docs instructions before the first docs call. Use the docs skill if one is listed; otherwise call `guide(items: ["topic.index"])`. Call `guide(items: ["topic.uploads"])` before saving images placed in a doc.

1. **Find the tabs**: `read(ref: {object: "project", id: "<the id at the end of the artifact link>"})`. `files[]` lists each tab: its id, name, and `content` (the body node id and engine). `value.tabs` gives the order. Every later call takes `container: {kind: "project", id: <frame.slug>}`. Note that this is the UUID in `frame.slug`, not the short id in the link.
2. **Full structured read of each tab** (the primary record): `read(ref: {object: "node", id: <body id>}, engine: <its engine>, container, payload: {kind: "view"})`. If the result says `complete: false`, repeat with the `next` keys added to the payload. Save it to `doc/tabs/<NN> - <tab name>/content.xml`. This copy keeps dates, mentions, dropdown chips, check lists, tables and where each chart sits.
3. **Markdown of each tab**: `export(container, file: <tab id>, format: "markdown", maxBytes: 11534336)`, saved as `content.md`. The export puts a text placeholder where each chart or diagram was and reports them as dropped. That's expected, because step 5 saves them. If the export is refused as too large, `content.xml` is enough. Never take the refusal's suggestion to "save the file into the doc's files", because that writes into the person's doc.
4. **A Word copy** of each tab: `export(…, format: "docx")`, saved as `doc/tabs/<NN> - <tab name>/<tab name>.docx`. It shows charts as pictures, so the person can open their doc without Claude. A file this size usually arrives as a saved result file (section 2).
5. **Charts and diagrams**: every `<embed … ref='node/<id>'>` in the XML is a chart or diagram. Read it with `read(ref: {object: "node", id}, container)` and save the whole `value` (its `code` and any `sources`) as `doc/widgets/<id>.json`. Record the embed's caption and tab. If a chart's sources cite an uploaded data file, save that file too.
6. **Dropdowns**: dropdown chips in the XML name an enum. Read each enum once (`read(ref: {object: "enum", id}, container)`). Save `enums.json` with each enum's name and options, and for every chip, the block it sits in and the option index it shows. Leave the file out if there are none.
7. **Images and files placed in the doc**: follow `topic.uploads` to read each one's bytes and save them under `doc/uploads/`.
8. **Mentions**: `self='true'` mentions are the person. Record other people's names in `mentions_of_others`.
9. **Comments**: list them with `query(object: "utterance", container, payload: {under: {object: "node", id: <body id>}})` and save `doc/comments.json`. They won't be recreated, because comments don't carry over, but the person keeps a copy of the discussion.

For **Sheets**, the tab's body engine may not be prose. Read it with the engine the doc read names, save the result as returned (`content.json`), and also try the markdown and docx exports.

### 3c. Write the manifest and notes

Write `manifest.json` (schema in manifest.md) and `NOTES.md`. Write each problem as a sentence the person would understand: "The image *logo-dark.png* could not be downloaded", not "read failed: 404".

**An empty artifact** (a dashboard or deck that was created but never filled in, so it holds only type files and no content anywhere) gets `"status": "empty"`. Its table row reads "yes — it was empty, nothing to save". Say this plainly, because the person may expect content there.

Every few artifacts, briefly tell the person how far you are ("12 of 37 saved so far").

## 4. Check everything, then report

1. List the person's artifacts again. For a full export, every one of their links must have a CSV row, and every row a folder with a manifest. For a subset, check the ones they asked for.
2. For each folder, check the saved files against the manifest by counting `page/` files, `assets/` files and `data/` documents on disk (0 is right where the manifest says 0). Check Docs folders for one `content.xml` per tab and one widget file per embed. If a check fails, try that piece again once before marking it.
3. Show the table:

| Artifact | Saved | Problems |
|---|---|---|
| Q3 Customer Review | yes | — |
| Dashboard | yes | It was empty: nothing to save |

Close with one line. If everything says yes: "Everything is saved in this folder. Next, switch to the new organization and start a new session here." Otherwise, say what's wrong and offer to try again.
