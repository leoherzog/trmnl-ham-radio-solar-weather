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
  identically to before. The map **grows into only the leftover height** so the top stats
  can never clip on the taller X screen: the content grid is `grow lg:grow-0` (fills the
  480px OG screen, but sizes to content on X), and the map band is `hidden lg:flex flex--row
  stretch` with the image `image--contain h--full w--auto`. The mechanism is load-bearing and
  easy to get wrong (all verified against the real framework CSS — see Common pitfalls below):
  `.layout` is `overflow:hidden` + `justify/align-center`, so any column taller than the screen
  is centered and clipped on BOTH ends — that's the top-stats-clipping bug. `stretch` is the
  real framework class (`flex:1 1 0%; min-height:0`) that lets the band both grow into the
  remaining space AND shrink below the image's intrinsic height, so it can never force overflow.
  `h--full` + `w--auto` size the image to that space keeping aspect (it's aspect-locked 2:1 and
  bounded by the leftover height, so it can't be widened without freeing vertical space).
  `flex--row` centers the image on both axes — do NOT use `flex--center-x`/`flex--center-y`
  here, they're direction-aware (map to `justify-content` vs `align-items` depending on a
  sibling `flex--row`/`flex--col`) and resolve to the wrong axis without one, leaving the map
  left-aligned. It's the one external/network image.
  (hamqsl's `solarmap.php` and NOAA GOES GeoColor full-disk were the
  rejected alternatives — too cluttered / not whole-earth.)
- `title_bar` — inline walkie-talkie SVG icon, instance name, localized "Updated" time.
- `hf_table` — HF band-conditions table; pairs each band's day rating (`band[i]`) with
  its night rating (`band[i + 4]`).
- `cond_badge` / `vhf_badge` — map a rating string to a styled label (Good = filled,
  Fair = outline, Poor / closed = gray).
- `stat_card` — one headline-index card (`item`/`meta`/`content` with a `value` +
  caption); params `val`, `cap`, `value_size`, `label_size`, `fit`, plus optional
  `lg_value_size` / `lg_label_size` that emit `lg:` variants to scale the card up on
  TRMNL X (≥1024px). Shared by every size's metric grid so the box markup lives in one
  place. The empty `<div class="meta"></div>` is **intentional, not dead markup** — `.meta`
  renders a fixed-width gray block (`width:var(--item-meta-width)`, gray-70 fill), which is
  the decorative bar beside each number. Don't "clean it up."
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
   Also tailor each size for TRMNL X: base classes target OG, and `lg:` variants scale
   content up at ≥1024px so the larger X screen isn't left with whitespace.
3. Guard every `{{ }}` against nil/empty (the feed can omit fields); `strip` padded values.
4. Keep the `title_bar` icon inline (no network URLs — they can fail on-device).
5. No inline styles, no emojis, no custom CSS — framework classes only.

## Common pitfalls (learned the hard way)

These are TRMNL-framework gotchas, all verified by grepping the compiled framework CSS
(`https://trmnl.com/css/latest/plugins.css`). **When a layout/sizing class isn't behaving,
verify it exists in that CSS before trusting the design-system guide** — the guide documents
intent, the CSS is ground truth, and several "obvious" class names below do not exist.

1. **`.layout` clips, it doesn't scroll.** It's `container-type:size; overflow:hidden;
   justify-content:center; align-items:center`. Any content taller than the screen is
   **centered and cropped on BOTH ends** (so vertical overflow shows up as the *top* getting
   cut off, not the bottom). There is no scrollbar to save you — content must fit.
2. **`min-h--0` / `min-w--0` / `overflow--hidden` do NOT exist.** Don't reach for a Tailwind
   name and assume it's there. The real way to get a grow-and-shrink flex item (flex-grow +
   `min-height:0`) is the **`stretch`** class (`flex:1 1 0%; min-height:0; min-width:0`), or
   `stretch-y` for the vertical-only variant. Min/max sizing utilities that DO exist:
   `h--min-[Ncqh]`, `h--max-[Ncqh]` (and `cqh`/`cqw` work because `.layout` is a size container).
3. **`flex--center-x` / `flex--center-y` are direction-aware, not absolute axes.** They map to
   `justify-content` vs `align-items` based on whether a `flex--row`/`flex--col` class is also
   present; with neither, they resolve to the wrong axis. To truly center a lone child on both
   axes, use `flex--row` (`flex-direction:row; align-items:center; justify-content:center`).
4. **`image--contain` is *only* `object-fit:contain`.** It sets no width/height — it does
   nothing until the `<img>` has an explicit size (`h--full`, `w--auto`, an `h--[Ncqh]`, etc.).
5. **Sizing utilities (`h--`/`w--`) only ship `sm:`/`md:` responsive prefixes — there is no
   `lg:h--`.** To affect only TRMNL X, gate the *element* with `hidden lg:flex` (etc.) and put
   the base (unprefixed) size class on it; it only renders at lg anyway.
6. **An empty `<div class="meta"></div>` is not nothing** — `.meta` paints a fixed-width gray
   block. Here that's a deliberate decorative bar (see `stat_card`); elsewhere it's a classic
   "why is there a gray box" surprise.
7. **No MCP/screenshot in this repo's workflow.** There's no TRMNL MCP server wired up, so
   markup can't be render-verified from here — reason from the CSS and verify on-device.
