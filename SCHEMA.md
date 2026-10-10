# Data format (schemaVersion 1)

Flesh and Blood card data taken from the official LSS Card Vault API
(`api.cardvault.fabtcg.com`) by fab-cardvault-extractor. The tool writes every file in this
repo except `README.md` and this file; do not edit them by hand, they are regenerated from `raw/`.

## Layout

```
raw/cards/{slug}.json        API response, re-indented (2 spaces) but otherwise unchanged
cards/{slug}.json            the card + its English printings ("base file")
cards/{language}/{slug}.json what differs in that language + that language's printings
sets.json                    every set seen in a printing
products.json                every product seen in a printing
vocabulary.json              class/type translations and value counts per field
index.json                   slug -> name, pitch, languages, print ids, source
```

`{slug}` is the Card Vault card id: `{name-slug}-{pitch}` for pitched cards (`snatch-1`),
`{name-slug}` otherwise (`malice-domina-of-the-dead`).

Language folders: `chinese`, `dutch`, `french`, `german`, `italian`, `japanese`, `korean`,
`polish`, `portuguese`, `russian`, `spanish`, `thai`. An unknown language code is used as the
folder name as-is.

## General rules

- Values are Card Vault's own words (`rainbow-foil`, `majestic`, `classic_precon`). Nothing is
  mapped to other codes. Unrecognized values are kept unchanged.
- Every field is present; an empty or missing source value is `null`. Exceptions (omitted when
  empty) are marked "optional" below.
- Files are UTF-8, 2-space indent, `\n` line endings, trailing newline. Rebuilding without new
  input produces identical bytes.
- Facts from the source and values we derive are separate: derived values live under `derived`
  (and `text.plain`).

## Base file: `cards/{slug}.json`

| Field | Meaning |
|---|---|
| `schemaVersion` | `1` |
| `id` | the slug |
| `name` | name of the front core |
| `objectType` | raw `object_type` (`deck-card`, `hero-card`, ...) |
| `cardType` | raw `card_type` |
| `cores[]` | the card's rules faces, front first (see below). Usually one. |
| `legality` | one entry per format, see below |
| `rulings[]` | raw `rulings_errata`, unchanged |
| `printings[]` | English printings, sorted by release date (unknown last), then id |
| `derived.keywords[]` | bold phrases in the front core's text, except ability labels (`**Action** --`). Sorted. |
| `derived.editions` | print id -> edition guess from the prefix: `U-` = `unlimited`, `R-` = `unlimited-revised`. Only prefixed ids. |
| `source.kind` | `api` (fetched), `pasted` (imported from a saved response) or `unknown` |
| `source.fetchedAt` | when the raw file was obtained (UTC) |
| `source.rawSha256` | SHA-256 of the response bytes as received (before re-indenting) |
| `source.updatedAt` | the card's `updated_at` in Card Vault |

The base file always exists, even when a card has no English printing (then `printings` is
empty); card-level data in Card Vault is in English.

### Core

| Field | Meaning |
|---|---|
| `position` | raw `layout_position` |
| `name`, `color` | as in the source |
| `pitch`, `chi`, `cost`, `power`, `defense`, `life`, `intellect` | numbers from the `*_value` fields; `null` when the card has none |
| `printedStats` | optional. Stat strings that are not a plain number (`X`, `*`, `3+`), by stat name |
| `typeLine`, `traitLine` | raw `typebox`, `traitbox` |
| `classes[]`, `talents[]`, `types[]`, `subtypes[]` | English names, in the order printed |
| `text.markup` | rules text in Card Vault markup: `**bold**`, `{br}` line break, `{r}` `{p}` `{d}` `{t}` ... icons |
| `text.plain` | derived: markup with `**` removed and `{br}` as `\n`; icon tokens stay |

### Legality

Fixed keys, always all present: `cc` (Classic Constructed), `blitz`, `commoner`, `silver-age`,
`living-legend`, `upf` (Ultimate Pit Fight). Each is `{ "status", "reason" }`; `status` is the
source value with spaces as dashes (`legal`, `not-legal`, ...). A format Card Vault adds later is
included under its slugified name.

Note: `not-legal` with reason `programmatic_legality` is computed by LSS (for example an adult
hero in Blitz); it does not mean banned.

### Printing

| Field | Meaning |
|---|---|
| `id` | raw `print_id` (`IAR115-RF`, `U-WTR167`, `JA_IAR053`) |
| `code` | printed collector code of the front face (`IAR115`) |
| `language` | raw `print_language` |
| `set`, `setNumber` | set code (key into `sets.json`) and number |
| `edition` | always `null` for now (see `derived.editions`) |
| `rarity` | raw `rarity`, with a `-marvel` suffix removed |
| `isMarvel` | `true` when the raw rarity ended in `-marvel` (`promo-marvel` -> `promo` + `true`) |
| `layout` | `regular`, `double-sided`, ... |
| `product` | product slug (key into `products.json`) |
| `releaseDate` | the product's release date |
| `isDefault`, `isPublished`, `publishedAt`, `notes[]` | as in the source |
| `faces[]` | physical faces, sorted by `layout_position` |

### Face

| Field | Meaning |
|---|---|
| `id` | raw `face_id` (`GEM184-MV`, `GEM184-MV_BACK`) |
| `side` | `front`, `back`, then `face-3`, ... by position |
| `position`, `orientation`, `language` | as in the source |
| `name`, `typeLine`, `traitLine` | as printed on this face |
| `text`, `flavorText` | `{ markup, plain, html }`; `html` is Card Vault's rendering |
| `stats` | `pitch`, `cost`, `power`, `defense`, `life`, `intellect` as printed (strings) |
| `printedCode`, `printedLanguageCode`, `printedYear`, `artist`, `rarityMark`, `message` | as printed |
| `finish`, `artType`, `border`, `frame`, `frameStyle`, `watermark[]` | as in the source |
| `images` | `{ small, normal, large }` URLs |

## Language file: `cards/{language}/{slug}.json`

Only exists for languages the card is printed in. Apply it on top of the base file:

1. `name` (optional) replaces the base `name`.
2. Each entry in `cores[]` is matched to the base core with the same `position`. Every property
   present in it replaces the base value; a missing property means "same as English".
   Possible properties: `name`, `typeLine`, `traitLine`, `classes`, `talents`, `types`,
   `subtypes`, `text` (`{ markup, plain, html }`).
3. `printings[]` are that language's printings (same shape as in the base file). They are added,
   not merged: they are different printings with their own ids.
4. `derived.editions` covers that language's printings.

Legality, stats and rulings never differ by language, so they are only in the base file.

Translated names and text come from the most recent printing in that language that has them.
Older printings keep their own wording on their faces. `classes`/`talents`/`types`/`subtypes` come
from Card Vault's translation table; they are left out when Card Vault has no translation.

## `sets.json`

`{ schemaVersion, sets: [{ code, name, type, parent }] }`, sorted by code.

## `products.json`

`{ schemaVersion, products: [{ slug, name, type, releaseDate, language, group }] }`, sorted by
slug. `group` is `{ slug, name, type, releaseDate }` or `null`.

## `vocabulary.json`

- `translations`: category (`classes`, `talents`, `types`, `subtypes`) -> English name ->
  language -> text.
- `values`: field -> value -> number of occurrences (printings, faces or cards depending on the
  field). Lists every value seen, including new ones.

## `index.json`

`{ schemaVersion, cards: { slug: { name, pitch, languages[], printIds[], source } } }`.
`source` (`kind`, `fetchedAt`, `rawSha256`) is the one thing that cannot be recomputed from
`raw/`, so the tool reads it back from here when rebuilding.
