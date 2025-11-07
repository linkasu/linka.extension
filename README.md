# LINKa Extension Format (.linka)

> 🇷🇺 Russian version: [README.ru.md](README.ru.md)

## Purpose
`.linka` files package symbol-based communication boards for the LINKa.look AAC suite. Each archive carries a board description (`config.json`) plus the media that cards rely on. This repository defines the container so that alternative tooling can reliably produce, inspect, or validate LINKa boards.

## Container Layout
- The file is a ZIP archive. `config.json` sits at the root next to every referenced asset. There are no nested folders.
- Every media file is stored exactly once. When two cards point to the same asset, LINKa.look stores a duplicated copy with an underscore-prefixed name to keep entries unique before updating the JSON references.
- Paths inside `config.json` are file names only. The player extracts an archive into a temporary directory and resolves the file names relative to that directory at runtime.

```
example.linka
├── config.json
├── apple.png
├── apple.ogg
└── space.png
```

## `config.json` schema
`config.json` is UTF-8 JSON with the following top-level properties:

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `version` | string | yes | Free-form semantic tag. The current editor writes `"1.3"` but the loader accepts earlier values as long as the schema matches. |
| `description` | string | no | Arbitrary text shown in LINKa.look’s description dialog. |
| `directSet` | bool | no (defaults to `false`) | Hides the output row and forces immediate audio playback when a user selects a card. |
| `columns` | integer | yes | Visible columns per page (>=1). |
| `rows` | integer | yes | Visible rows per page (>=1). |
| `withoutSpace` | bool | yes | `true` means card captions are concatenated character-by-character (no automatic spaces) and audio controls stay hidden in the editor. |
| `cards` | array | yes | Ordered list of card objects. Order is row-major and drives pagination. |

### Cards
Each card entry uses these fields:

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `id` | integer | yes | Stable identifier. LINKa.look keeps the original number when re-saving sets. |
| `width` | integer | no (default `1`) | Reserved for future multi-cell cards. Values <1 are coerced to 1 when the editor loads the file. |
| `height` | integer | no (default `1`) | Same as `width`. |
| `imagePath` | string | required for `Common`/`Space` cards | File name of the card image (PNG/JPG/GIF). GIFs stay animated in player mode. |
| `title` | string | required for `Common`/`Space` cards | Card caption rendered below the image and sent to the text bar unless the card is a `Space`. |
| `audioPath` | string | optional | File name of an `.ogg`, `.mp3`, or `.wav`. For speech boards (`withoutSpace = false`) the editor prompts creators to provide audio. |
| `cardType` | integer enum | yes | `0` = `Common`, `1` = `Space`, `2` = `Fake`. |

### Card types
- `Common (0)` – regular card with caption, image, and optional audio. Selecting the card either types the caption or enqueues it for speech.
- `Space (1)` – represents a blank character. The editor auto-generates a ␣ image and caption and disables manual editing. When activated it inserts a literal space even if `withoutSpace = false`.
- `Fake (2)` – invisible placeholder that keeps pagination stable (for example, to push cards onto the next page). Fake cards must have empty `title`, `imagePath`, and `audioPath` values.

## Behavioral flags
- **`withoutSpace`** – Governs how the runtime composes utterances. When `true`, LINKa.look appends card titles directly (no leading spaces), deletes characters one by one, and speaks the accumulated text buffer instead of individual cards. Editors also hide audio controls for such boards because per-card speech is meaningless.
- **`directSet`** – Intended for training boards that should speak instantly. When the flag is `true`, the runtime hides the text output row and forces “play audio from card” mode regardless of the user’s global settings.

## Grid semantics
Cards are consumed in the order they appear in `cards`. The player slices the list into pages of `rows × columns` entries, fills each grid row from left to right, and automatically adds more pages when needed. Any trailing slots on the last page stay empty unless you add `Fake` cards to preserve alignment.

## Assets
- Images: PNG, JPG, and GIF are supported. Choose square canvases where possible so the runtime does not pillarbox content.
- Audio: OGG is the native format, but MP3/WAV files are accepted and converted on save.
- File names should be ASCII-safe; the save routine already strips directories, but unique names prevent automatic underscore prefixes.

## Example `config.json`
```json
{
  "version": "1.3",
  "description": "Sample food board",
  "directSet": false,
  "columns": 5,
  "rows": 3,
  "withoutSpace": false,
  "cards": [
    {
      "id": 0,
      "imagePath": "apple.png",
      "title": "Apple",
      "audioPath": "apple.ogg",
      "cardType": 0
    },
    {
      "id": 1,
      "imagePath": "space.png",
      "title": "\u2423",
      "audioPath": "",
      "cardType": 1
    }
  ]
}
```

## Validation checklist
1. Confirm the archive opens as ZIP and includes `config.json` at the root.
2. Validate `config.json` against the schema above and ensure all referenced assets exist.
3. Ensure every asset is referenced by at least one card (or remove it to keep archives lean).
4. Test the board inside LINKa.look to verify pagination, pronunciation mode, and `directSet`/`withoutSpace` behavior.
