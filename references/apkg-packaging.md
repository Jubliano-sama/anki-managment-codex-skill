# APKG Packaging with Embedded Media

Use this reference when creating importable Anki deck packages. The correct extension is `.apkg`; treat `.akpg` as a typo unless the user explicitly says otherwise.

## Choose a Packaging Path

1. Existing Anki deck: use Anki's export flow, select deck package `.apkg`, enable `Include Media`, and include scheduling only when the user wants learning progress transferred.
2. Generated deck: use `genanki` or Anki's own APIs/export tools rather than editing package internals directly.
3. Existing collection data: create a temporary/profile-safe deck, attach media through Anki, then export `.apkg` with media.

Avoid hand-rolling `.apkg` internals unless the user asks for package-format work specifically. Package internals and import behavior can change across Anki releases.

## genanki Pattern

Use unique, stable model and deck IDs. Generate them once, then hardcode or persist them for future updates so re-imports update the intended note type/deck.

```python
from pathlib import Path

import genanki

MODEL_ID = 1607392319
DECK_ID = 2059400110

model = genanki.Model(
    MODEL_ID,
    "Media Basic",
    fields=[
        {"name": "Prompt"},
        {"name": "Answer"},
        {"name": "Media"},
    ],
    templates=[
        {
            "name": "Card 1",
            "qfmt": "{{Prompt}}<br>{{Media}}",
            "afmt": "{{FrontSide}}<hr id=\"answer\">{{Answer}}",
        },
    ],
    css="""
.card {
  font-family: Arial, sans-serif;
  font-size: 20px;
  text-align: center;
}
img {
  max-width: 100%;
  height: auto;
}
""",
)

deck = genanki.Deck(DECK_ID, "Generated Media Deck")

media_path = Path("media/cell-diagram.png")
note = genanki.Note(
    model=model,
    fields=[
        "Identify this structure.",
        "Mitochondrion",
        '<img src="cell-diagram.png">',
    ],
    guid="anki-management-cell-diagram-001",
    tags=["generated", "media"],
)
deck.add_note(note)

package = genanki.Package(deck)
package.media_files = [str(media_path)]
package.write_to_file("generated-media-deck.apkg")
```

For audio, put `[sound:file.mp3]` in a field and include the source audio file in `package.media_files`.

## Media Rules

- Reference media by basename inside Anki fields: `cell-diagram.png`, not `media/cell-diagram.png`.
- Include the real file paths in the package's media list.
- Ensure every packaged media basename is unique. If two source folders contain `image.png`, rename or stage one before packaging.
- Do not put subdirectory paths in Anki field references.
- Use HTML media tags in fields, not template interpolation like `<img src="{{Image}}">`.
- Prefix static template assets such as fonts with `_`, for example `_inter.woff2`, so Anki's media checker does not remove them as unused.
- Prefer MP3 for audio and MP4 for video when portability to AnkiWeb/mobile matters.

## Validation

Before delivering an `.apkg`:

1. Confirm the output filename ends in `.apkg` and is not `collection.apkg` unless the user intentionally wants a legacy collection package name.
2. Import into a disposable Anki profile.
3. Confirm images render, audio/video play, and custom fonts/static assets load on the front and back templates.
4. Run `Tools > Check Media`; fix missing or unused media unless the unused media are intentional static assets prefixed with `_`.
5. Re-import the package once if the deck is meant to update previous notes; verify stable GUIDs update notes instead of duplicating them.
