---
name: anki-management
description: Safely manage Anki collections, decks, notes, cards, note types, templates, imports, and collection databases. Use when Codex needs to add or update Anki cards, edit collection.anki2 SQLite databases, generate import files or packages, work with .apkg/.colpkg files, format Anki card HTML/CSS/templates/media, preserve sync safety, create backups, or repair/validate Anki collection changes.
---

# Anki Management

## Overview

Manage Anki content with a safety-first workflow. Prefer Anki's public collection APIs, Anki's import/export paths, or AnkiConnect before raw SQLite writes; use direct database edits only when necessary and always with backups and validation.

## Operating Rules

- Treat Anki content as notes first, cards second. A note stores fields; one or more cards are generated from the note type templates.
- Always create a timestamped backup before any write. For live user collections, back up `collection.anki2` and `collection.media`; when possible, also create/export a `.colpkg` with media.
- Never copy, move, or write `collection.anki2` while Anki is open. Ask the user to close Anki or use an API path designed for a running Anki instance.
- Prefer working on a copy of the collection, then swap it in only after integrity checks and an Anki `Tools > Check Database` pass.
- Avoid changing Anki's table schema. Add-on-specific data belongs in supported config/add-on storage, not new columns on core tables.
- For version-specific behavior, inspect the target collection (`select ver from col`, `sqlite_master`) and check the current official Anki docs/source if available.
- If a task could be satisfied by generating CSV/TSV or `.apkg` for import, prefer that over editing a user's live database.

## Workflow

1. Identify the target:
   - Collection/profile path, deck name, note type, fields, templates, tags, media, and whether Anki is currently open.
   - User intent: create new material, update existing notes, change templates/styling, move cards, or repair data.

2. Choose the safest write path:
   - For new notes/cards: create CSV/TSV with Anki headers, use Anki's CSV import API, AnkiConnect, or Anki's Python `Collection.add_note()`.
   - For package generation: create `.apkg` with a reputable generator when the user wants an importable deck instead of direct profile edits.
   - For template/styling edits: use Anki's model/notetype APIs or generate clear front/back/CSS snippets for the user to apply.
   - For direct SQLite: use only after confirming Anki is closed, backups exist, schema is inspected, and no supported API/import route fits.

3. Back up and validate:
   - Before writes: copy `collection.anki2`, optionally copy `collection.media`, and run `PRAGMA integrity_check`.
   - After writes: run `PRAGMA integrity_check`, verify row counts and expected card generation, open in Anki, then run `Tools > Check Database`.

## Adding Notes and Cards

Prefer adding notes through Anki's collection API so Anki handles IDs, checksums, generated cards, sync state, and scheduling defaults:

```python
from anki.collection import Collection
from anki.decks import DeckId

col = Collection(r"path\to\collection.anki2")
try:
    notetype = col.models.by_name("Basic")
    if notetype is None:
        raise RuntimeError("Missing note type: Basic")

    deck_id = col.decks.id("Target Deck", create=True)
    note = col.new_note(notetype)
    note["Front"] = "Question text"
    note["Back"] = "Answer text"
    note.tags.append("imported")
    col.add_note(note, DeckId(deck_id))
finally:
    col.close()
```

For bulk text import, create UTF-8 CSV/TSV with Anki headers such as:

```text
#separator:Tab
#html:true
#notetype:Basic
#deck:Target Deck
#tags:generated reviewed
#columns:Front	Back
What is HTTP 404?	A "not found" response.
```

## Direct SQLite Edits

Use direct SQLite only for controlled, offline collection work. Read [direct-sqlite.md](references/direct-sqlite.md) before doing any raw insert/update/delete.

Minimum direct-edit procedure:

1. Confirm Anki is closed.
2. Create a timestamped backup of `collection.anki2` and media.
3. Inspect schema and collection version.
4. Use transactions and parameterized SQL.
5. Never alter core table definitions.
6. Keep local changed rows at `usn = -1` unless operating in a documented server/sync context.
7. Recalculate fields/checksums through Anki code when possible.
8. Run integrity checks and Anki's database checker afterward.

## Formatting Cards

Use Anki templates as HTML and Anki styling as CSS. Read [card-formatting.md](references/card-formatting.md) when creating or editing fields, templates, CSS, media, cloze notes, TTS, tables, code snippets, or platform-specific styling.

Keep formatting portable:

- Store field content as HTML when formatting is required; escape literal `<`, `>`, and `&`.
- Put repeated layout in templates and CSS, not duplicated into every note field.
- Reference media with `<img src="file.jpg">` or `[sound:file.mp3]`; place media in `collection.media` or package media correctly.
- Test on desktop and mobile when using JavaScript, custom fonts, platform CSS, or complex layouts.

## References

- [safe-workflows.md](references/safe-workflows.md): backups, imports, profile paths, and validation.
- [direct-sqlite.md](references/direct-sqlite.md): SQLite schema notes and raw editing guardrails.
- [card-formatting.md](references/card-formatting.md): Anki HTML/CSS/template/media formatting patterns.
