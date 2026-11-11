# Backup folder layout and manifest

The export and recreate parts talk to each other only through this folder, often days apart and in different sessions or organizations. Write exactly this shape so the other part, or a resumed session, can trust it.

## Layout

```
<backup folder>/                      ← the folder Claude Code was opened in
├── My artifacts.csv                  ← one row per artifact the person owns, newest first
├── export-info.json                  ← where and when the export ran
├── New links.csv                     ← written by recreate only
├── 001 - <Artifact name>/
│   ├── NOTES.md                      ← plain-words note for the person
│   ├── manifest.json                 ← machine-readable record (below)
│   ├── page/                         ← published files at their published paths
│   ├── assets/                       ← uploaded images, logos, fonts, PDFs: <asset id>.<ext>
│   ├── data/                         ← the artifact's saved database, one JSON file per document
│   └── doc/                          ← Claude Docs / Sheets only
│       ├── tabs/<NN> - <tab name>/content.md   ← markdown export of the tab
│       ├── tabs/<NN> - <tab name>/content.xml  ← full structured read (chips, embeds, tables)
│       ├── widgets/<node id>.json    ← each chart/diagram node as read (its code and everything else)
│       ├── tabs/<NN> - <tab name>/<tab name>.docx  ← Word copy the person can open without Claude
│       ├── enums.json                ← every dropdown list: options, and which value each chip picked
│       ├── comments.json             ← the doc's comment threads, kept for reference (not recreated)
│       └── uploads/                  ← images and files placed in the doc
├── 002 - <Artifact name>/
└── …
```

Folder names: a three-digit number in the same order as `My artifacts.csv`, then ` - `, then the artifact name. Strip characters that Windows or macOS forbid (`<>:"/\|?*` and control characters), trim trailing dots and spaces, and cut to 80 characters. If two artifacts have the same name, the numbers keep them apart.

Leave out a subfolder that would be empty.

## My artifacts.csv

UTF-8 with a header row. Quote every field (names often contain commas):

```
"#","Name","Link","Type","Last updated","Folder"
"1","Team Onboarding FAQ","https://claude.ai/artifact/Yz98X…","Docs","2026-11-11","001 - Team Onboarding FAQ"
```

`Type` is the artifact type's name as the types listing shows it (`Docs`, `Slides`, `Design`, `Design System`, `Data`, `Sheets`, `Motion`, `Whiteboard`, …), or `Page` for a plain HTML artifact made without a type.

## export-info.json

```json
{
  "exported_at": "2026-11-11T14:02:00Z",
  "org_id": "3f1c9a2e-…",
  "org_name": "Acme Enterprise",
  "subscription": "enterprise",
  "scope": "all",
  "artifact_count": 37
}
```

`scope` is `"all"`, or `"subset"` when the person named particular artifacts.

Take the org fields from `claude auth status` when you can. Otherwise use `null` for org_name and subscription, and take `org_id` from the `artifact.org` field of a Claude Docs read, if the person has a Doc.

## manifest.json (one per artifact folder)

```json
{
  "number": 1,
  "name": "Q3 Customer Review",
  "link": "https://claude.ai/artifact/Ab12Cd34Ef56Gh78Ij90Kl",
  "type": "Slides",
  "type_url": "https://claude.ai/artifact/<slides-type-id>",
  "version": "1790000000-ab12",
  "status": "complete",
  "exported_at": "2026-11-11T14:05:12Z",
  "capabilities": { "…": "the stored declaration exactly as the read showed it, or null" },
  "files": [
    { "path": "project/deck.json", "bytes": 548, "saved": true }
  ],
  "skipped_type_files": 21,
  "assets": [
    { "id": "0a1b2c3d4e5f60718293a4b5c6d7e8f9", "url": "/_blob/0a1b2c3d4e5f60718293a4b5c6d7e8f9",
      "content_type": "image/png", "bytes": 40960, "sha256": "9f3e…", "local": "assets/0a1b2c3d4e5f60718293a4b5c6d7e8f9.png", "saved": true }
  ],
  "data": [
    { "collection": "datasets", "documents": 4, "saved": true }
  ],
  "doc": {
    "tabs": [ { "id": "5c8e1f0a-2d7b", "name": "Plan", "order": "V", "body": "b41d6e93-7a20", "folder": "doc/tabs/01 - Plan" } ],
    "widgets": [ { "id": "7e2a91c4-0b3d", "tab": "5c8e1f0a-2d7b", "caption": "roadmap · 3 phases", "saved": true } ],
    "enums": 2,
    "uploads": [ { "ref": "…", "local": "doc/uploads/…", "saved": true } ],
    "mentions_of_others": ["Sam K."]
  },
  "references_to_other_artifacts": ["https://claude.ai/artifact/Mn12Op34Qr56St78Uv90Wx"],
  "problems": []
}
```

- `status` is `complete` when every entry has `saved: true` and `problems` is empty. It is `empty` when the artifact turned out to hold no content of the person's: only type files, no assets, and every collection empty. Recreate skips `empty` artifacts and says why. Otherwise it is `partial`. A folder without a manifest is unfinished.
- `skipped_type_files` counts every type file left out, including the root `SKILL.md` and `index.html`.
- `data` lists every collection read, including those with `documents: 0`.
- `problems` holds short plain-words sentences ("The chart 'Spend by team' could not be read"). These go straight into the person's table.
- `references_to_other_artifacts` lists any claude.ai artifact links found in the saved page, project files or doc text. Recreate swaps them for the new links, which is how a deck keeps pointing at its design system.
- Leave out keys that don't apply (no `doc` on a Slides artifact, no `data` without a database).

## NOTES.md

A few lines for the person, not for Claude:

```
# Q3 Customer Review
Slides · https://claude.ai/artifact/Ab12Cd34Ef56Gh78Ij90Kl · saved 11 Nov 2026

Saved: 12 slides, the deck settings, 18 images.
Not saved: nothing — everything was copied.
Not carried over (by design): comments and past versions.
```

## New links.csv (recreate)

Same quoting rules. This file goes to colleagues so they can update their bookmarks, so keep it to these three columns:

```
"Name","Old link","New link"
```

## _rebuild/ and recreate-progress.json (recreate)

Recreate never edits the export. When files need changing, for example swapping old image links for new ones, it copies them into `_rebuild/<NNN>/` first and edits the copies, so the backup stays the original.

`recreate-progress.json` tracks each artifact's state, so a resumed session never creates a second copy:

```json
{
  "org_id": "…the new organization…",
  "artifacts": {
    "https://claude.ai/artifact/Ab12Cd34Ef56Gh78Ij90Kl": {
      "number": 3,
      "new_link": "https://claude.ai/artifact/AbC…",
      "state": "created | filled | verified",
      "asset_map": { "/_blob/0a1b…": "/_blob/9e1d…" },
      "differences": []
    }
  }
}
```

Write the entry with `state: "created"` and `new_link` **the moment** the create call returns, before filling anything. Then append the row to `New links.csv`.
