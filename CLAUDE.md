# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A TRMNL plugin that displays ham-radio solar-terrestrial conditions from
[hamqsl.com](https://www.hamqsl.com/solarxml.php) (N0NBH's solar data). It renders the
headline solar indices, HF band conditions (day vs. night), and VHF propagation
phenomena across the four TRMNL layout sizes.

## Data Source

- **Strategy:** `polling`, single URL `http://www.hamqsl.com/solarxml.php`
- **Content type:** `text/xml`. TRMNL parses XML with Ruby's `Hash.from_xml`, so:
  - The root `<solar>` element is preserved — data lives under `solar.solardata.*`.
  - An element with attributes exposes its text under the **`__content__`** key
    (e.g. `band["__content__"]`, `source.url` + `source["__content__"]`).
  - Repeated sibling elements become **arrays** (`calculatedconditions.band` is an
    array of 8; `calculatedvhfconditions.phenomenon` is an array of 5).
- `conditions.xml` is a saved copy of the raw feed; `conditions.sample.json` is the
  parsed merge-variable shape (verified against a live plugin) for reference/testing.

### Key fields (under `solar.solardata`)

`solarflux`, `sunspots`, `aindex`, `kindex`, `xray`, `solarwind`, `magneticfield` (Bz),
`aurora`, `latdegree`, `geomagfield`, `signalnoise`, `updated` (a `DD Mon YYYY HHMM GMT`
string), plus the `calculatedconditions.band` and `calculatedvhfconditions.phenomenon`
arrays. Several numeric fields are space-padded in the feed, so the template `strip`s
them. `fof2` / `muffactor` can be `null`.

## Template Structure

All rendering lives in `shared.liquid`; the four size files are thin wrappers that
`{% render 'main', ... %}` with a `layout_size` argument so every size shares one source
of truth. `shared.liquid` defines these `{% template %}` partials:

- `main` — normalizes the indices, parses the `updated` stamp, branches per
  `layout_size`, and renders the `title_bar`. On `full`, a `hidden lg:flex` full-width
  band below the grid (image only, no header or divider) shows a clutter-free whole-earth day/night terminator
  map (`www.universemonitor.com/files/earth/terminator_720x360.png`, a 720×360 PNG
  refreshed every ~10 min; also offers 360×180 and 160×80). It appears **only on V2/X**
  (the `lg:` ≥1024px breakpoint) to fill the larger screen — OG (≤800px) renders
  identically to before. It's the one external/network image; `image-dither` +
  `image--contain` and a `h--[34cqh]` cap keep the wide 2:1 map e-ink-correct and
  banner-sized. (hamqsl's `solarmap.php` and NOAA GOES GeoColor full-disk were the
  rejected alternatives — too cluttered / not whole-earth.)
- `title_bar` — inline walkie-talkie SVG icon, instance name, localized "Updated" time.
- `hf_table` — HF band-conditions table; pairs each band's day rating (`band[i]`) with
  its night rating (`band[i + 4]`).
- `cond_badge` / `vhf_badge` — map a rating string to a styled label (Good = filled,
  Fair = outline, Poor / closed = gray).
- `stat_card` — one headline-index card (`item`/`meta`/`content` with a `value` +
  caption); params `val`, `cap`, `value_size`, `label_size`, `fit`. Shared by every
  size's metric grid so the box markup lives in one place.
- `stat_row` — a label/value space-weather row.
- `vhf_list` — VHF phenomena with friendly location names.

## The `updated` timestamp

hamqsl formats it as `DD Mon YYYY HHMM GMT` (e.g. `04 Jun 2026 1623 GMT`). The `HHMM`
has no colon, so `main` splits the string, reinserts a colon, then formats with
`l_date` (locale-aware) and the user's UTC offset — same display approach as the
`nws-severe-weather` plugin. Respects the `timemode` (12/24-hour) custom field.

## Development Guidelines

1. Keep all rendering logic in `shared.liquid`; size files stay one-liners.
2. Tailor each size — don't just shrink `full`. Smaller sizes drop secondary stats.
3. Guard every `{{ }}` against nil/empty (the feed can omit fields); `strip` padded values.
4. Keep the `title_bar` icon inline (no network URLs — they can fail on-device).
5. No inline styles, no emojis, no custom CSS — framework classes only.
