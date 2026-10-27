# LINKa Extension Format (.linka)

> Russian version: [README.ru.md](README.ru.md)

## Purpose
`.linka` files package symbol-based communication boards for the LINKa.look AAC suite. Each file contains a board description (`config.json`) and the media files used by cards. This repository documents the container format so external tools can create, inspect, and validate LINKa boards.

## Container Layout
- The file is a ZIP archive. `config.json` is stored at the archive root next to referenced assets.
- Paths in `config.json` are ZIP entry names, normally plain file names such as `8f5a...png` or `speech.mp3`.
- The current editor stores newly added or generated assets as UUID-based file names with the original extension.
- When sets are merged and an asset name already exists, the copied asset is renamed to `uuid + extension` and the JSON reference is updated.
- On save, LINKa.look removes unreferenced assets and writes a fresh `config.json`.

```
example.linka
├── config.json
├── 9f1d2c6e-7a5b-4d9e-8d2e-a5d1b4f32b11.png
└── 4cc5e8b9-7d7a-43fd-a97b-2b8a44ffb610.mp3
```

## `config.json` Schema
`config.json` is UTF-8 JSON. The current LINKa.look editor writes version `"3.0"` and uses page-based layout.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `version` | string | yes | Current value is `"3.0"`. Loaded configs are normalized to the current version when saved. |
| `withoutSpace` | bool | yes | When `true`, card titles are appended without automatic spaces and `SpaceCard` can be part of the output. |
| `directSet` | bool | no (defaults to `false`) | Hides the output row and plays card audio immediately. |
| `quizAutoNext` | bool | no (defaults to `true`) | On a correct quiz answer, move to the next page automatically. |
| `quizReadQuestion` | bool | no (defaults to `false`) | Read quiz questions aloud when supported by the player. |
| `description` | string | no | Arbitrary text shown in LINKa.look's description dialog. |
| `pages` | array of `SetPage` | yes | Ordered pages of the set. This replaces the legacy top-level `columns`, `rows`, and `cards` layout. |

### Legacy compatibility
Older files can contain top-level `columns`, `rows`, `cards`, `quiz`, and `questions`. LINKa.look still reads those fields and migrates them into `pages`, but newly saved files use `version: "3.0"` with page objects.

## Pages
Each entry in `pages` has this shape:

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `id` | string | yes | Page identifier. LINKa.look uses UUID strings for generated pages. |
| `mode` | string enum | yes | `standard`, `quiz`, or `match`. Invalid or missing values are normalized to `standard`. |
| `columns` | integer | yes | Visible columns on the page. Values are normalized to integers `>= 1`. |
| `rows` | integer | yes | Visible rows on the page. Values are normalized to integers `>= 1`; `match` pages always use `2`. |
| `cards` | array of `Card` | yes | Cards in row-major order. On load/save, the page is trimmed or padded to `rows × columns`. |
| `question` | string | for `quiz` pages | Quiz question. It is omitted for non-quiz pages. |

### Page modes
- `standard` - regular communication board page.
- `quiz` - question page. Correct answers are marked with `card.answer: true`; `quizAutoNext` controls whether a correct answer immediately advances to the next page.
- `match` - matching game page. It always has two rows; pairs are linked with `matchId`, and `matchLane` is normalized to `top` or `bottom` from card position.

## Cards
Each card entry uses these fields:

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `id` | string | yes | Card identifier. LINKa.look uses UUID strings for generated cards. |
| `cardType` | integer enum | yes | `0` = `AudioCard`, `1` = `SpaceCard`, `2` = `EmptyCard`, `3` = `NewCard`. |
| `width` | integer | no (default `1`) | Card width in grid cells. Values <1 are coerced to 1. Supported on `standard` and `quiz` pages. |
| `height` | integer | no (default `1`) | Card height in grid cells. Values <1 are coerced to 1. Supported on `standard` and `quiz` pages. |
| `imagePath` | string | required for valid `AudioCard` | ZIP entry name of the image. PNG, JPG, JPEG, and GIF are supported. |
| `title` | string | required for valid `AudioCard` | Card caption and output text. |
| `audioPath` | string | no | ZIP entry name of an audio file. MP3, WAV, and OGG are accepted. |
| `audioText` | string | no | Text used to generate card audio, when the audio was created from text. |
| `audioVoice` | string | no | Voice identifier used with `audioText`, when available. |
| `answer` | `true` | for correct quiz answers | Only meaningful on `quiz` pages. It is removed from non-quiz pages during normalization. |
| `matchId` | string | for match pairs | Shared identifier for the two `AudioCard` entries that form one match pair. It is removed from non-match pages. |
| `matchLane` | `top` or `bottom` | generated for `match` pages | Normalized from card position. It is removed from non-match pages. |

### Card types
- `AudioCard (0)` - regular card with image, title, and optional audio. In the editor it is valid only when both `imagePath` and `title` are present.
- `SpaceCard (1)` - space output card. It participates in output when `withoutSpace = true`.
- `EmptyCard (2)` - exported empty placeholder that keeps the grid shape stable.
- `NewCard (3)` - editor placeholder for an empty editable slot. When the set is saved, `NewCard` entries are converted to `EmptyCard`.

## Behavioral Flags
- `withoutSpace` controls text composition. With `true`, LINKa.look appends `AudioCard` and `SpaceCard` output directly. With `false`, only `AudioCard` output is added in standard mode.
- `directSet` is intended for boards that should speak immediately. It hides the text output row and forces card audio playback.
- `quizAutoNext` controls whether a correct quiz answer advances automatically.
- `quizReadQuestion` stores whether quiz questions should be read aloud by supported players.

A card with `width` or `height` greater than `1` occupies a rectangular area starting at its row-major position. Covered positions in `cards` should be filled with `EmptyCard` entries so older app versions that ignore `width`/`height` safely degrade to a regular `1×1` grid instead of crashing. If a merged card would cross the page edge, loaders may clamp its span to the available space.

## Assets
- Images: PNG, JPG, JPEG, and GIF are supported.
- Audio: MP3, WAV, and OGG are accepted. Generated text-to-speech audio is stored as MP3.
- Only assets referenced by `AudioCard.imagePath` and `AudioCard.audioPath` are preserved during save.
- Keep custom file names ZIP-safe and unique. LINKa.look-generated names are UUID-based.

## Example `config.json`
```json
{
  "version": "3.0",
  "withoutSpace": false,
  "directSet": false,
  "quizAutoNext": true,
  "quizReadQuestion": false,
  "description": "Sample food board",
  "pages": [
    {
      "id": "b1f93341-7a7e-46df-b928-d49f9eecb57d",
      "mode": "standard",
      "columns": 3,
      "rows": 2,
      "cards": [
        {
          "id": "f4926d55-477f-4900-b016-3298b17b30ea",
          "cardType": 0,
          "width": 2,
          "imagePath": "apple.png",
          "title": "Apple",
          "audioPath": "apple.mp3"
        },
        {
          "id": "1aa5db1a-48f5-47d5-bcb4-9da5d5c54723",
          "cardType": 2
        },
        {
          "id": "8c299d04-cb51-4dbf-946f-ad5177535a47",
          "cardType": 1
        }
      ]
    }
  ]
}
```

## Validation Checklist
1. Confirm the file opens as ZIP and includes `config.json` at the root.
2. Parse `config.json` as UTF-8 JSON and normalize/check the schema above.
3. Ensure every `AudioCard.imagePath` and `AudioCard.audioPath` reference exists in the archive.
4. Ensure each page has valid dimensions and no more than `rows × columns` cards before normalization.
5. For `quiz` pages, ensure at least one intended answer has `answer: true`.
6. For `match` pages, ensure each `matchId` links exactly two `AudioCard` cards in different rows.
7. Test the board inside LINKa.look to verify standard output, quiz, match, and direct playback behavior.

## Лицензия

Собственный программный код LINKa в этом репозитории распространяется на условиях
GNU General Public License версии 3 или, по выбору получателя, любой более поздней
версии (SPDX: `GPL-3.0-or-later`). Полный неизменённый текст GPLv3: [LICENSE](LICENSE).

Разрешены использование, форки, изменения и коммерческое использование. При
распространении покрываемого GPL кода и производных программ необходимо соблюдать
GPL: сохранять уведомления об авторстве, лицензии и отсутствии гарантий, отмечать
изменения и их даты, предоставлять соответствующий исходный код получателям и
лицензировать покрываемую производную программу на условиях GPL. Личные изменения,
которые не распространяются, публиковать не требуется. Гарантии не предоставляются
в пределах разделов 15–16 GPLv3.

Авторы и правообладатели сохраняют права на свои вклады; существующие имена и годы
авторства сохраняются в исходниках, уведомлениях и истории Git. Поддержка или участие
АНО «Линка» сами по себе не означают передачу ей исключительных прав. Copyright FSF
в тексте GPL относится к тексту лицензии, а не к авторству программы. Ранее
предоставленные права на версии под MIT/BSD/ISC не отзываются.

Эта GPL-декларация относится только к собственному программному коду и не
перелицензирует сторонний код, SDK/DLL Tobii, Unity-компоненты, зависимости,
изображения, звуки, шрифты, бренды и учебные материалы. Их исходные лицензии,
атрибуция и ограничения сохраняются; права на товарные знаки не предоставляются.
Публичная доступность файла не заменяет разрешение правообладателя. Совместимость
всей готовой сборки с GPL и права на материалы без подтверждённой лицензии этим
изменением не устанавливаются.
