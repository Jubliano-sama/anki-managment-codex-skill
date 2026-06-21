# Safe Anki Workflows

## Prefer Supported Paths

Use the safest route that meets the request:

1. CSV/TSV import for bulk note creation or updates.
2. Anki Python `Collection` API for local programmatic edits while Anki is closed.
3. AnkiConnect or add-on code for edits while Anki is running.
4. `.apkg` generation for portable decks, with embedded media when the deck references images, audio, video, fonts, or static assets.
5. Raw SQLite only when the task cannot be done safely through the above paths.

Authoritative references to re-check when exact behavior matters:

- Anki Manual, files/backups/importing/templates: `https://docs.ankiweb.net/`
- Anki add-on docs: `https://addon-docs.ankiweb.net/`
- Anki source: `https://github.com/ankitects/anki`
- genanki package generation docs: `https://github.com/kerrickstaley/genanki`

## Locate Collection Data

Recent default profile locations:

- Windows: `%APPDATA%\Anki2\<Profile>\collection.anki2`
- macOS: `~/Library/Application Support/Anki2/<Profile>/collection.anki2`
- Linux: `~/.local/share/Anki2/<Profile>/collection.anki2` or `$XDG_DATA_HOME/Anki2/<Profile>/collection.anki2`

Profile folders also commonly contain `collection.media` and `backups`.

## Backup Contract

Before any write:

1. Confirm Anki is closed unless using AnkiConnect or add-on APIs.
2. Copy `collection.anki2` to a timestamped file, for example `collection.anki2.backup-YYYYMMDD-HHMMSS`.
3. Copy `collection.media` when changing media references or packaging a full restore point.
4. Prefer a manual `.colpkg` backup with media for user-facing safety when Anki can perform the export.
5. Run `PRAGMA integrity_check;` on the source and stop if it is not `ok`.

After any write:

1. Run `PRAGMA integrity_check;`.
2. Compare expected note/card counts and sample rows.
3. Open Anki and run `Tools > Check Database`.
4. If templates changed, run `Tools > Empty Cards` and inspect generated cards before deleting anything.
5. Sync only after verifying the collection behaves correctly.

## Bulk Import File Pattern

Use UTF-8 text. Put headers at the top to remove import ambiguity:

```text
#separator:Tab
#html:true
#notetype:Basic
#deck:Target Deck
#tags:generated reviewed
#columns:Front	Back
Capital of Argentina	Buenos Aires
```

Use `#guid column:N` only when updating notes previously exported by Anki. For custom stable IDs, prefer a normal first field and let Anki use duplicate matching.

## Collection API Pattern

```python
from anki.collection import Collection
from anki.decks import DeckId

col = Collection(r"path\to\collection.anki2")
try:
    nt = col.models.by_name("Basic")
    deck_id = col.decks.id("Target Deck", create=True)
    note = col.new_note(nt)
    note["Front"] = "Prompt"
    note["Back"] = "Answer"
    note.tags.extend(["generated", "reviewed"])
    col.add_note(note, DeckId(deck_id))
finally:
    col.close()
```

Use `col.update_note(note)` for field edits and `col.set_deck(card_ids, deck_id)` for moving cards. These methods mark changes for sync and trigger Anki's card-generation logic.

## Package Export Pattern

When creating a shareable deck package:

1. Use `.apkg`, not `.akpg`.
2. Include media when notes reference local files.
3. Exclude scheduling/review history unless the user is transferring their own learning state.
4. Validate the package by importing it into a temporary profile before sharing.
