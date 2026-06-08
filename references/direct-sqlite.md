# Direct SQLite Editing

Direct SQLite editing is risky. Prefer Anki APIs because they maintain sync flags, checksums, card generation, note type behavior, and scheduling invariants.

## Core Tables

Common core tables:

- `col`: collection metadata, schema version, sync/schema timestamps, and legacy JSON blobs.
- `notes`: note rows with field content.
- `cards`: generated study cards, scheduling state, and deck placement.
- `revlog`: review history.
- `graves`: deletion records used for sync.

Modern collections may also include normalized `notetypes`, `fields`, `templates`, `decks`, `deck_config`, `config`, and `tags` tables. Older schema stores note types, decks, deck config, and tags as JSON in `col`.

Always inspect:

```sql
select ver from col;
select name, sql from sqlite_master where type in ('table', 'index') order by type, name;
```

## Notes Table Essentials

Typical `notes` columns:

- `id`: note ID, commonly epoch milliseconds and unique.
- `guid`: Anki note GUID. Let Anki create this when possible.
- `mid`: note type ID.
- `mod`: modified time in epoch seconds.
- `usn`: update sequence number; local unsynced changes are normally `-1`.
- `tags`: space-padded tags, for example ` tag1 tag2 `.
- `flds`: note fields joined by ASCII unit separator `\x1f`.
- `sfld`: sort field, usually derived from the first field with HTML stripped.
- `csum`: checksum of the sort field used for duplicate checks.
- `flags`, `data`: usually `0` and empty text for simple notes.

If you must create rows manually, use Anki's Python utilities or existing collection API code to generate `guid`, `sfld`, and `csum`. Do not guess these values when an API path is available.

## Cards Table Essentials

Typical `cards` columns:

- `id`: card ID, commonly epoch milliseconds and unique.
- `nid`: note ID.
- `did`: deck ID.
- `ord`: card template ordinal or cloze ordinal.
- `mod`, `usn`: modification and sync markers.
- `type`, `queue`, `due`, `ivl`, `factor`, `reps`, `lapses`, `left`, `odue`, `odid`: scheduling state.
- `flags`, `data`: user flags and extra data.

For new unscheduled cards, Anki's backend should create scheduling state. Manual card inserts are easy to get subtly wrong, especially across scheduler versions.

## Transaction Pattern

```sql
PRAGMA integrity_check;
BEGIN IMMEDIATE;
-- parameterized inserts/updates from code, not string-concatenated values
COMMIT;
PRAGMA integrity_check;
```

Use `ROLLBACK;` on any error. Never run migrations or `ALTER TABLE` on Anki core tables unless following official Anki source for that exact version.

## Raw Note Insert Guardrails

If direct insertion is unavoidable:

1. Determine target `mid` from `notetypes` or legacy `col.models`.
2. Determine `did` from `decks` or legacy `col.decks`.
3. Determine required fields and template ordinals.
4. Build `flds` with `\x1f` separators in exact field order.
5. Compute `sfld` and `csum` consistently with Anki.
6. Insert the note with `usn = -1`.
7. Generate cards only for non-empty fronts according to the note type templates.
8. Update collection modification metadata without marking schema modified unless the schema actually changed.
9. Validate with Anki's own database checker.

When in doubt, stop and generate an import file instead.

## Deletes and Updates

- Updating fields through raw SQL should be followed by Anki's card-generation/update logic. The public API exposes `after_note_updates()` for notes modified directly in the database.
- Deletions should normally use Anki APIs so `graves` and related cards/revlog state are handled.
- Do not delete individual generated cards as a content-management mechanism; change templates/fields and use Anki's Empty Cards workflow.
