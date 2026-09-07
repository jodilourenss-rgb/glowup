# Glow-Up

A personal Summer 2026 planner (September to December 2026) for training, beauty
routines and monthly reflection. Built for one person, on one phone, installed
to the home screen as a PWA. Not a general-purpose app and not intended for
other users.

## Running it

There is no build step and no dependencies. To run locally, serve the folder
with any static file server and open `index.html`, for example:

```
python3 -m http.server 8000
```

Opening `index.html` directly via `file://` mostly works, but the service
worker only registers over `http(s)`, so offline caching will not activate.

## Deploying

The site deploys via GitHub Pages from this repository (no build step, no
GitHub Actions workflow). Push to the branch Pages is configured to serve
(check the repo's Settings → Pages) and the static files go live as-is.

### Cache-busting after a deploy

`sw.js` pre-caches the app shell under a named cache, currently `glowup-v2`.
The cache name is versioned, and the comment at the top of `sw.js` is a
reminder to bump it:

```js
const CACHE = "glowup-v2";
```

**Bump this string on every deploy that changes `index.html` or any cached
asset.** If you don't, the service worker's fetch handler will keep serving
the old cached copy first (it revalidates in the background, but the page
already executing in memory won't pick that up).

Even with the bump, the installed PWA needs a **full close and reopen**
(not just tapping back to the home screen) to pick up the new version,
because the currently running page keeps executing the JS it already loaded
until it is relaunched. This is expected behaviour for the current service
worker, not a bug.

## File inventory

| File | Size | Purpose |
|---|---|---|
| `index.html` | 122 KB | The entire app: HTML, CSS and JS inline. |
| `manifest.json` | 425 B | PWA manifest (name, icons, colours, start URL). |
| `sw.js` | 1.4 KB | Service worker: caches the app shell, stale-while-revalidate fetch. |
| `icon-192.png` | 11 KB | PWA icon, 192×192, referenced by `manifest.json` and the `<link rel="icon">`. |
| `icon-512.png` | 20 KB | PWA icon, 512×512, referenced by `manifest.json` and pre-cached by `sw.js`. |
| `apple-touch-icon.png` | 11 KB | iOS home screen icon, referenced by `<link rel="apple-touch-icon">`. |
| `hero-today.jpg` | 114 KB | Background image for the Today view hero, referenced in CSS (`.hero::before`). |
| `hero-beauty.jpg` | 68 KB | Background image for the Beauty view banner, referenced in CSS (`.banner::before`). |
| `bg-sand.jpg` | 56 KB | **Not referenced anywhere.** The page background is a base64 blob embedded in `index.html` instead. Orphaned. |
| `icon-alt-sage.png` | 19 KB | **Not referenced anywhere** (no manifest entry, no HTML reference). Orphaned. |
| `icon-alt-umber.png` | 20 KB | **Not referenced anywhere** (no manifest entry, no HTML reference). Orphaned. |

Everything `index.html` references (`hero-today.jpg`, `hero-beauty.jpg`,
`icon-192.png`, `apple-touch-icon.png`, `manifest.json`) is present in the
repo. Nothing is missing. Three files are unreferenced dead weight:
`bg-sand.jpg`, `icon-alt-sage.png`, `icon-alt-umber.png`.

## Map of `index.html`

### CSS, in section order (banner comments in the file)

1. `:root` design tokens, resets, base type
2. `type` — heading and label classes (`.h1`, `.h2`, `.h3`, `.lbl`, `.sm`, `.xs`)
3. `chrome` — top bar, page wrap, section spacing, `.card`
4. `hero` — full-bleed photo header used on Today
5. `rows` — the tappable list row used across Today/Beauty/Reset, plus the exercise list and lift log
6. `buttons` — `.btn-1` (filled), `.btn-2` (outline)
7. `tabs` — month/track/reset pill tabs
8. `calendar` — the Month grid, day cells, marker dots, legend
9. `rings` — the Track view progress rings
10. `fields` — text inputs and dot-rating inputs
11. `banner` — the Beauty view photo banner
12. `sheet` — the bottom sheet (day detail) and its backdrop
13. `home arc` — the scroll-snap arc nav on Home
14. `back` — the back button/pill used on every non-home view

There is no "home masthead" section and no glass/frosted-glass tokens
(`--glass`, `--glass-on`, `--glass-line`) in the current file — see
**Gotchas** below.

### The six views

All are `<section class="view">` elements inside `#app`, shown/hidden by
toggling the `.on` class in `go(v)`:

- `v-home` — the arc navigation (default view on load)
- `v-today` — today's plan, session checklist, daily-care habits, month focus
- `v-month` — calendar grid for a selected month, tap a day to open the sheet
- `v-track` — progress rings, strength progression, energy/reflection fields
- `v-beauty` — daily-care habits, weekly habit grid, teeth-whitening cycles, hair/nail fields
- `v-reset` — monthly checklist by group, close-the-month reflection fields, backup/restore

### Data constants

- `PLAN` — seven entries, Monday through Sunday, each a day template
  (`k` kind, icon, title, sub, when, goal, exercise list). Thursday's entry
  (`alt:true`) carries both `exA` and `exB` for the A/B week split.
- `MONTHS` — four entries, September through December 2026, each with
  `i` (zero-based month index), `name`, `short`, `focus` and `note`.
- `HABITS` — seven daily-care items, each `[key, label, icon]`.
- `RESET` — the monthly reset checklist, grouped under `Fitness`, `Beauty`, `Lifestyle`.
- `DAYNAMES`, `MONTHNAMES` — display strings for dates.

### State shape (`S`)

Persisted as JSON to `localStorage` under the key `glowup:v1` (falls back to
`window.storage` if `localStorage` is unavailable):

| Key | Shape | Stores |
|---|---|---|
| `sessions` | `{ [dateISO]: true }` | Which days had a session marked done |
| `ex` | `{ [dateISO]: { [exerciseIndex]: true } }` | Ticked exercises within a day's session |
| `habits` | `{ [dateISO]: { [habitKey]: true } }` | Daily-care habit ticks (used by Today and Beauty) |
| `notes` | `{ [dateISO]: string }` | Free-text note entered in the day sheet |
| `teeth` | `{ [cycleIndex]: { [applicationIndex]: true } }` | Teeth-whitening cycle progress |
| `reset` | `{ "monthIndex:group:itemIndex": true }` | Monthly reset checklist ticks |
| `fields` | `{ [fieldKey]: string }` | All free-text/number inputs bound via `data-f`, keyed by that attribute |
| `swap` | `{ [dateISO]: planIndex }` | Days where the scheduled session was swapped for a different one |
| `lifts` | `{ [dateISO]: { [liftName]: { w, r } } }` | Logged weight/reps per lift per day |

### Render functions

`renderAll()` calls, in order: `renderToday`, `renderMonth`, `renderTrack`,
`renderBeauty`, `renderReset`, `renderHome`. Each reads from `S` and rebuilds
its view's DOM; most user interactions call `save()` then re-render the
affected view(s) directly rather than going through `renderAll()`.

## Design tokens

| Token | Hex | Used for |
|---|---|---|
| `--cream` | `#E9E2D7` | Page base background, sheet background |
| `--surface` | `#F5F1E9` | Card backgrounds, active row/arc-item backgrounds |
| `--sand` | `#DED5C6` | Subtle fills: tabs, chips, "on" row background |
| `--umber` | `#52483B` | Primary dark: headings, gym accent, "done" button |
| `--caramel` | `#A89672` | Warm accent: run accent, primary button, focus outline on inputs |
| `--sage` | `#99ABA6` | Training/pilates accent |
| `--storm` | `#6E777B` | Rest accent, secondary text |
| `--t1` | `#52483B` | Primary text |
| `--t2` | `#6E777B` | Secondary text |
| `--t3` | `#9A948A` | Tertiary text (labels, hints) |
| `--line` | `#D8CFC0` | Borders and dividers |
| `--r` | `14px` | Large corner radius (cards, hero, sheet top) |
| `--rs` | `10px` | Small corner radius (rows, buttons, inputs) |
| `--pad` | `20px` | Standard horizontal page padding |

No glass/translucency tokens exist yet (see Gotchas).

## The A/B week rule

`ANCHOR` is fixed at `2026-08-31`. `weekLetter(dateKey)` finds the Monday of
that date's week, measures how many whole weeks have passed since `ANCHOR`
(`Math.floor((mondayOfWeek - ANCHOR) / 604800000)`), and calls the week "A"
if that count is even, "B" if odd. `planFor()` uses this only for the
Thursday interval-run template (the one entry with `alt:true`), picking
`exA`/"Interval run" on A weeks and `exB`/"Steady run" on B weeks. Every
other day of the week ignores the letter entirely.

## Gotchas

- **The background image is a base64 blob, not a file.** `body::before` in
  the CSS embeds the page background as a ~76,000-character base64 JPEG data
  URI on a single line, not a reference to `bg-sand.jpg` (which sits in the
  repo unused). This was done because the image previously 404'd when
  referenced by path. Do not open `index.html` in a naive editor or reader
  without stripping this line first, for example:
  ```
  sed 's/base64,[A-Za-z0-9+/=]*/base64,<BLOB>/g' index.html
  ```
  Edit the file with targeted string replacement, never a full rewrite.
- **Orphaned files.** `bg-sand.jpg`, `icon-alt-sage.png` and
  `icon-alt-umber.png` are committed but referenced nowhere in `index.html`
  or `manifest.json`.
- **No masthead, no glass nav, no `inner` body class in the current code.**
  Some prior context describes a home-screen masthead (date + month focus)
  and frosted-glass styling on the arc navigation as already shipped, and
  describes `go(v)` as toggling an `inner` class on `<body>`. None of that
  is present in the current `index.html`: `v-home` renders only the arc,
  the arc items use the plain `--surface` token (not a translucent glass
  token), there are no `--glass*` tokens defined, and `go(v)` only toggles
  `.on` on the view sections. Treat that as aspirational or from a version
  that didn't make it into this file, not as the current state.
- **Service worker caching is aggressive by design.** See the Deploy section
  above: bump `CACHE` in `sw.js` on every asset change, and expect to fully
  close and reopen the installed app afterward.
- **Data is on-device only.** Everything in `S` lives in `localStorage`
  under `glowup:v1`. There is no sync and no server. The Reset view's
  export/import buttons are the only backup mechanism, and they are manual.
