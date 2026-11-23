# Chord and MIDI library

Static chooser for a chord progression or a MIDI song. The live site is [kemckai.github.io/ableton-live-library](https://kemckai.github.io/ableton-live-library/).

Chord progressions come from [Chordonomicon](https://github.com/opendining/chordonomicon) (Apache 2.0). This repo stores progression metadata and chord charts. The MIDI list is song titles and paths. The `.mid` files are not in this repo. **Open in the repo** points at the private repository `kemckai/ableton-live-mcp`, under `midi/`. Spot-checked paths from `data/midi.json`, including `.38 Special/Caught Up In You.mid` and `10cc/Dreadlock_Holiday.1.mid`, are files in that tree.

There is no build step and no package manifest. `index.html` is the whole app. It fetches JSON from `data/` with relative URLs.

## Run it

Serve the repo root over HTTP. Opening `index.html` as a file fails because `fetch` of the JSON is blocked:

```bash
python3 -m http.server 8000
```

Then open `http://localhost:8000`. GitHub Pages currently serves this same tree. `.nojekyll` is what keeps Pages from running Jekyll over `data/`.

The header counts (`679807 progressions · 17232 MIDI files` in the current data) come from `data/facets.json`, not from a live scan of the other files.

## Using the page

Two tabs share one search box.

**Progressions.** Nothing loads until a genre is chosen. `loadGenre` reads that genre's `files` list from `facets.json`, fetches every file in parallel, and concatenates them in that order. A successful response is ignored when a newer load has started (`state.req`). A failed older load still rejects the `onchange` promise, and that error handler replaces the list text even if a newer genre already loaded. Decade, mode, and search then filter the rows already in memory. The list shows 40 rows (`PAGE`). **Show more** appends the next 40.

**MIDI.** The first visit fetches `data/midi.json` once and keeps it in `state.midi`. Genre, decade, and mode controls are hidden on this tab. Search matches artist, title, and path. The variant field is not part of the search text.

Search waits 160ms, then requires every remaining word to appear (case-insensitive). It matches the preview column only, in both stored form (`Fsmin`) and display form (`F#m`). It does not read the full chart in `data/chords/`. On the progression tab, `queryParts` always pulls the words `major`, `maj`, `minor`, and `min` out of the text. Those words become the mode only while the mode buttons are still on **Any**. A pressed **Major** or **Minor** button wins, and the word is still not searched as text.

**Any** is the only way to see rows whose mode is `mixed` (16794 rows in the current files). There is no Mixed button.

**All decades** is the only way to see rows whose decade is `""` (257626 rows). The dropdown is the `decades` array in `facets.json` and does not include a blank. A chosen decade must match the stored label exactly. Those labels are the decade strings in `facets.json` (`"2020"`, `"2010"`, and so on), not a full release year.

Picking a row calls `choose`. Progression detail loads the full chart (see below). MIDI detail shows artist, title, optional variant, and the relative path. **Copy path** writes that relative path. **Open in the repo** links to `https://github.com/kemckai/ableton-live-mcp/blob/main/midi/` plus each path segment passed through `encodeURIComponent`, so spaces in names such as `.38 Special/Caught Up In You.mid` survive.

**Copy chords** writes the stored chart, including section tags and `s` for sharp, not the labels on the chips. Clipboard writes need a secure context (localhost or HTTPS). The handler does not report a failure.

The genre name `Untagged` is omitted from row metadata. Those titles read `Progression` rather than `Untagged`.

## Data layout

```
data/facets.json                 index: counts, genres, decades, bucket size
data/progressions/<file>.json    one genre, or one shard of Untagged
data/chords/<bucket>.json        full charts keyed by progression id
data/midi.json                   one array of MIDI rows
```

The client does not discover files. `facets.json` is the index.

```json
{
  "progressions": 679807,
  "midi": 17232,
  "decades": ["2020", "2010"],
  "genres": [
    { "name": "pop rock", "count": 39557, "files": ["pop-rock.json"] },
    { "name": "Untagged", "count": 327696, "files": ["untagged-0.json", "untagged-1.json", "untagged-2.json"] }
  ],
  "bucket": 3000
}
```

`name` is the `<select>` value, including the space in `pop rock` and the capital U in `Untagged`. `files` are paths under `data/progressions/`. The hyphen in `pop-rock.json` is not computed; it is whatever `files` says. `count` is display-only. `progressions` and `midi` are the header stats.

In the current files, genre counts sum to `progressions`, and the progression ids are a partition of `1..679807`: each id appears in exactly one genre file. Ids inside a file are not a contiguous range. `untagged-0.json` still contains ids from the high end of the catalog.

A progression row is a four-element array:

| Index | Meaning | Example |
| --- | --- | --- |
| 0 | id, shared with the chord chart | `5` |
| 1 | decade, or `""` | `"2020"` |
| 2 | `major`, `minor`, or `mixed` | `"major"` |
| 3 | short preview used by the list and by search | `"C G C G C F Dmin G"` |

A MIDI row is a five-element array:

| Index | Meaning | Example |
| --- | --- | --- |
| 0 | id | `5` |
| 1 | artist | `"10cc"` |
| 2 | title | `"Dreadlock Holiday"` |
| 3 | variant label, or `""` | `"1"` |
| 4 | path under the other repo's `midi/` directory | `"10cc/Dreadlock_Holiday.1.mid"` |

An empty variant is hidden. A non-empty one is shown as `variant 1`.

## Chord charts

`choose` loads `data/chords/${Math.floor(id / bucket)}.json` and reads the key `String(id)`. With `bucket` 3000, id `5` is `data/chords/0.json` key `"5"`, and id `642833` is `data/chords/214.json`. Published shards are `0.json` through `226.json`. `0.json` holds ids `1`–`2999`. Each later full shard holds 3000 ids. `226.json` holds `678000`–`679807`.

The bucket promise is cached on `state.chords`, including a rejected fetch, so a failed shard does not retry until reload. Progression file loads check `res.ok` and render `Could not load <file>`. Chord, MIDI, and facets loads do not; a bad status becomes a JSON parse error. A missing facets file renders `Could not load the library.`

A chart is one string. Section markers are `<name>` tokens. `sectionsOf` splits on those tags. A chart that does not start with `<` puts the leading chords in a section called `Progression`. Underscores in the tag become spaces, so `<verse_1>` is shown as `verse 1`. Many charts have no tags at all and show as that single section. The list preview is stored separately from the chart. On a sample of 600 rows spread across every genre file, the preview tokens were the chart, or a prefix of it, after section tags were removed.

`formatChord` rewrites each `/`-separated piece in `formatChunk`, then `chordKind` colors the chip:

- If the second character is `s` and the rest does not start with `sus`, that `s` becomes `#`. `Fsus4` stays, because the remainder really does start with `sus`. `Fs7sus4` becomes `F#7sus4`, and `Gssus4` becomes `G#sus4`.
- Every `min` substring then becomes `m`, including inside minor-major chords. Final labels: `Fsmin` → `F#m`, `Cminmaj7` → `Cmmaj7`, `Fsminmaj7` → `F#mmaj7`, `Daugmaj7/Fs` → `Daugmaj7/F#`.
- Color checks that display label in order: substring `dim`, then `aug`, then `maj`, then an `m` anywhere else, otherwise major. `Cmmaj7` and `F#mmaj7` are colored major because `maj` is tested first. `Daugmaj7/F#` is class `aug`.
- CSS only styles `.minor` and `.dim`. An `aug` chip uses the same background as a major chip.

## Updating the files

Replace the JSON in place. Keep these in agreement or the page lies or fails open:

- `genres[].files` must exist. A rename that is not listed in `facets.json` is invisible. A listed file that 404s fails that genre load.
- `genres[].count` should equal that genre's row count. The header uses the top-level `progressions` and `midi` numbers, so those should equal the real lengths (and the sum of genre counts).
- `decades` should stay the exact strings stored in column 1. A new decade string that is missing here cannot be selected.
- `bucket` must stay the divisor `index.html` uses. Changing it without rebuilding `data/chords/` sends ids to the wrong file.
- Progression ids should stay unique and present in `data/chords/`. A missing key renders an empty detail and copies an empty string.

## Troubleshooting

| What you see | What is going on |
| --- | --- |
| `Choose a genre to load progressions.` | Expected. The progression list stays empty until the genre `<select>` has a value. |
| `Could not load the library.` | `data/facets.json` failed to fetch or parse. Usually the page was opened from disk, or the site is not rooted at the repo. |
| `Could not load pop.json` | That genre's file is missing or not JSON. The name is the entry in `files`, not the genre name. |
| A genre loads, then the list becomes an error | An older `loadGenre` failed after a newer one succeeded. Successes check `state.req`. The `onchange` error handler does not, so it still overwrites the list. |
| Search for `minor` hides major rows and does not find the letters m-i-n | Those words are mode switches while **Any** is selected. Search `Amin` or `F#m` to match chord text. |
| Major is pressed, but the query says `minor` | The button wins. The word is discarded, not searched. |
| No `mixed` rows, or fewer rows than the genre count | **Major** / **Minor**, or a decade, is active. `mixed` and blank decades only show under **Any** and **All decades**. |
| Chords copied as `Fsmin` and `<verse_1>` | **Copy chords** returns the stored chart on purpose. The chips are display-only. |
| MIDI link 404s or asks for login | The button opens `kemckai/ableton-live-mcp` at `midi/<path>` on `main`. That repository is private. This repo does not contain the `.mid` file. |
| Detail stays on `Loading chords…` for one id range | That bucket fetch failed and is cached until reload. |
