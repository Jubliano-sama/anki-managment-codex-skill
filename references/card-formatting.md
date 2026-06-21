# Card Formatting

## Field Content

Anki field content can be plain text or HTML. If HTML is enabled during import, escape literal characters:

- `<` as `&lt;`
- `>` as `&gt;`
- `&` as `&amp;`

Use `<br>` for intentional line breaks in imported HTML fields.

## Templates

Basic front:

```html
{{Front}}
```

Basic back:

```html
{{FrontSide}}
<hr id="answer">
{{Back}}
```

Field names are case sensitive. `{{Front}}` and `{{front}}` are different.

Use conditional replacements to control display and card generation:

```html
{{#Example}}
  <div class="example">{{Example}}</div>
{{/Example}}
```

On standard note types, Anki creates cards only when the front side renders as non-empty. Put card-generation conditions on the front template.

## Styling

Put reusable style in the note type's Styling section:

```css
.card {
  font-family: Arial, sans-serif;
  font-size: 20px;
  text-align: center;
  color: #111;
  background: #fff;
}

.hint {
  color: #666;
  font-size: 0.9em;
}

.nightMode .hint {
  color: #bbb;
}
```

Style individual fields with wrappers:

```html
<div class="term">{{Term}}</div>
<div class="hint">{{Context}}</div>
```

## Media

Use:

```html
<img src="diagram.png">
```

or:

```text
[sound:audio.mp3]
```

Place referenced files in `collection.media` or package them with the deck. Do not rely on subdirectories in media. If using custom fonts in templates, prefix font filenames with `_` so media cleanup does not delete them.


For `.apkg` generation, the media reference inside a field should use the media basename:

```html
<img src="cell-diagram.png">
```

or:

```text
[sound:pronunciation.mp3]
```

The package builder must separately include the actual file path. Do not put `<img src="{{Image}}">` in a template and a bare filename in the field; put the full media tag in the field instead.

If source media files live in subdirectories, stage or rename them so their basenames are unique before packaging. Anki fields should not reference `images/cell-diagram.png`.

## Cloze

Use Anki's cloze note type for deletion cards:

```text
{{c1::Rome}} is the capital of {{c2::Italy}}.
```

Cloze ordinals are generated from cloze numbers. Do not manually add cloze cards with raw SQL unless reproducing Anki's card-generation rules exactly.

## Code and Tables

Inline code:

```html
<code>console.log("hello")</code>
```

Code block:

```html
<pre><code>function square(n) {
  return n * n;
}</code></pre>
```

Tables are regular HTML:

```html
<table>
  <tr><th>Concept</th><th>Meaning</th></tr>
  <tr><td>HTTP 404</td><td>Not found</td></tr>
</table>
```

Keep table CSS mobile-friendly and test in Anki desktop, AnkiMobile, AnkiDroid, and AnkiWeb if the deck is intended to be portable.

## JavaScript

Avoid JavaScript unless the user explicitly needs it. Anki clients render cards differently, and long-lived web views can break scripts that assume a fresh page load. If JavaScript is necessary, test across target clients and keep it isolated.
