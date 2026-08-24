# CLAUDE.md

Context for AI assistants working on this repo. Read this before editing `index.html`.

## What this is

One self-contained HTML file that fetches a year of weather for a coordinate and renders a printable climate report. Hand-drawn SVG charts, no framework, no build step, no backend.

The audience is someone making a slow decision about a place — buying land, planning a build, choosing where to live. That framing drives everything: the page should be honest about uncertainty, should never overstate precision, and should show the ten-year normal alongside the single year so nobody mistakes one year for a climate.

## Hard constraints

Do not break these without a very good reason.

1. **Single file, no build step.** Everything lives in `index.html`. Someone should be able to download it, double-click it, and have it work. No bundler, no npm install, no transpilation.
2. **Must work from `file://`.** No same-origin assumptions, no fetch of local assets, no module scripts (which are blocked on `file://` in some browsers).
3. **No `localStorage` / `sessionStorage` / cookies.** State lives in the URL fragment. This is deliberate: it makes sites shareable and bookmarkable, works from `file://`, and keeps the page renderable inside sandboxed iframes.
4. **No API keys.** Every service used must work keyless.
5. **Graceful degradation everywhere.** If Leaflet won't load, say so and fall back to typed coordinates. If the geocoder is down, say so. If normals fail, render the page without them and state that they're missing. Never show a spinner forever, and never show a vague error.

## File map

`index.html` is sectioned with `/* ═══ heading ══ */` comment banners. In order:

| Section | What lives there |
|---|---|
| `config` | `SITE` (mutable lat/lon/name), `NORMAL_YEARS`, `DAILY_VARS`, `MON` |
| `tiny utils` | `iso`, `num`, `mean`, `sum`, `quantile`, `fmt`, `esc` |
| colour ramp | `RAMP`, `tColor()` — temperature → colour interpolation |
| svg builder | `e(tag, attrs, inner)` and `T(...)` — build SVG as **strings**, not DOM |
| `fetch` | `jget`, `buildURL`, `rowsFrom`, `loadData`, `buildNormals`, `byMonth` |
| `coordinates` | `parseCoord`, `validCoord`, `fmtCoord`, `readHash`, `writeHash` |
| `setup screen` | `showSetup()`, `loadLeaflet()`, map/geocoder wiring |
| `toolbar` | `wireTools()` |
| drawing | `drawContours`, `drawRibbon`, `drawTemp`, `drawRain`, `drawHum`, `drawDays`, `drawWind`, `drawRose` |
| `stats + prose` | `drawStats()`, `writeRead()` |
| CSV fallback | `parseCSV`, `showError` |
| `render` / `entry` | `render()`, `start()`, hash-or-setup bootstrap |

## Design system

The visual language is a **bathymetric nautical chart**: pale weathered chart stock, layered blues, a sun-ochre used only for heat. Don't drift toward generic dashboard styling.

CSS custom properties on `:root` — always use these, never hardcode a hex in new code:

```
--paper #E7ECE9   ground          --depth   #1B5C72  water, rain, primary
--salt  #FAFBF9   cards           --shallow #79AEBD  light water
--ink   #0F3038   text            --sun     #DFA01C  heat, sun
--rule / --ink-10/20/45/70        --pine    #3F6046  vegetation, comfort, wind
```

**Colour carries meaning.** Blue = water (rain, humidity). Ochre → ember = heat. Green = wind and vegetation. Keep it that way; it's why the charts read without a legend.

**Typography follows a cartographic convention.** Real nautical charts label hydrographic features in *italic* and topographic features upright. This project borrows that: water-related section headings use `.hydro` / `h2.water` (Spectral italic), land and temperature headings use upright IBM Plex Sans. Numbers are IBM Plex Mono. It's a small thing that makes the page feel designed rather than generated — preserve it when adding sections.

Fonts load from Google Fonts with real fallbacks (`Georgia`, `system-ui`, `ui-monospace`). The page must still look deliberate offline.

## Chart conventions

- Charts are built as **SVG strings** and assigned via `innerHTML`. Use the `e()` helper. Don't switch to DOM construction — string building keeps the drawing functions short and readable.
- Every chart has a fixed `viewBox` and scales fluidly. Pick a viewBox that matches its layout slot; don't rely on pixel sizes.
- Use CSS variables inside SVG (`fill="var(--depth)"`). This is what makes the print stylesheet and any future dark mode work.
- Margins are declared as `L/R/Tp/B` constants at the top of each draw function. Follow the pattern.
- Partial months get an `*` suffix and reduced opacity.

### Adding a chart

1. Add the section markup with `.sec-head` + `.card` + an `<svg id="...">`.
2. Write `drawX(months, normals)` following an existing function's shape.
3. Call it from `render()`.
4. Add it to the id list in the test harness (below) so it's checked for `NaN`.
5. Check `@media print` pagination still lands on three pages.

## Data pipeline

`loadData()` makes three parallel requests and merges them:

- **archive** (`archive-api.open-meteo.com/v1/archive`) — start-364 days to yesterday
- **forecast** (`api.open-meteo.com/v1/forecast`, `past_days=10`) — closes the tail gap, because some reanalysis models lag a few days
- **normals** (archive again) — ten full calendar years, aggregated per calendar month by `buildNormals()`

Merge prefers the archive where days overlap. `timezone=auto` — never hardcode a timezone.

`byMonth()` produces ordered month buckets carrying `coverage` (fraction of the calendar month present), which downstream code uses to identify partial months.

**Cost note:** one page load costs roughly 200–250 weighted API calls under Open-Meteo's formula, almost all of it the normals request. If you add caching, cache the normals — they change once a year. Any cache must not use browser storage (see constraints); an in-memory cache keyed by rounded coordinate is fine within a session.

## Known traps

Each of these was a real bug. Don't reintroduce them.

- **Leaflet double-init.** `MAP` and `MARKER` are module-scoped on purpose. Declaring them inside `showSetup()` means the second visit tries `L.map("map")` on a container Leaflet already claimed, which throws *"Map container is already initialized"* — and the exception lands in the `.catch()`, producing a misleading "library wouldn't load" message.
- **Partial months poisoning extremes.** A 12-month window clips a month at each end. Those buckets have artificially low rainfall and skewed averages, and with a window spanning two Augusts you get duplicate labels. `writeRead()` filters to `coverage >= 0.85` before picking warmest/coolest/wettest/driest, and de-duplicates labels.
- **Hemisphere assumptions.** Never infer seasons from calendar months. Sort months by temperature to find warm and cool seasons. The page is tested with an inverted-seasons dataset.
- **Location-specific prose.** `writeRead()` must derive every claim from the data. An earlier version asserted "a cold Atlantic", "inland Coimbra", "the *nortada*", "Portuguese housing stock" — all nonsense once the coordinate is user-supplied. If you add a sentence, it must hold at any latitude.
- **Unstyled form controls.** `#mapsearch input` shares a CSS rule with `.field input[type=text]`. Adding an input with a novel class and no matching rule leaves it on browser defaults, misaligned with adjacent buttons.
- **Toolbar wiring.** `wireTools()` is called from both `render()` and `showError()`. Binding handlers only in `render()` leaves "Change site" dead on the error screen, trapping the user.
- **Print colours.** `print-color-adjust: exact` on `*` is required or the charts print as grey boxes.
- **Overstated precision.** The footer once hardcoded "~9 km grid". That's ERA5-Land's resolution; ERA5-Land is land-only, so coastal and offshore pins fall back to ERA5 at ~31 km. Don't assert a resolution the page can't verify.

## Testing

There's no test framework, and adding one would break the no-build-step rule. Instead, extract the script and run it under Node with a stubbed DOM and a synthetic Open-Meteo response. This catches the bugs that matter (NaN in charts, throws in draw functions, prose that reads wrong).

```bash
# extract the inline script
python3 -c "
import re
s=open('index.html').read()
open('/tmp/app.js','w').write(re.findall(r'<script>(.*?)</script>', s, re.S)[-1])
"
node --check /tmp/app.js
```

Then prepend a harness that stubs `document`, `location`, `fetch` and `L`, generates a synthetic annual cycle, and asserts afterwards that every chart id has non-trivial `innerHTML` containing no `NaN`, `undefined` or `Infinity`.

Things worth asserting:

- Both hemispheres (flip the phase of the synthetic temperature cycle).
- The hash entry path *and* the no-hash setup path.
- The change-site round trip: render → setup → open map → rebuild → setup → reopen map. Stub `L.map()` to throw on a second call for the same container, mirroring real Leaflet.
- Generated SVG parses as well-formed XML (wrap each chart's `innerHTML` in an `<svg>` root and parse it).
- `parseCoord()` against decimal, DMS, loose DMS, both hemispheres, and garbage input.

## Prose style

The written analysis is a feature, not filler. Keep it:

- **Derived, never asserted.** Every number comes from the data.
- **Plain.** Short sentences, no jargon, no hedging pile-ups.
- **Honest.** The "Sample check" paragraph exists to tell people when the year was unusual. The "Caveats" paragraph exists to stop the charts being over-trusted. Don't soften either.

## Deliberately not done

Ideas that are reasonable but absent, roughly in order of value:

- **Unit toggle** (°C/°F, km/h/mph, mm/in). Fetch metric always and convert at display time — the thresholds (25 °C, the 18–26 comfort band, the 16/22/27 day buckets) are baked into the logic and should stay metric internally.
- **Compare two sites side by side.** The single most requested thing for property decisions.
- **Longer baselines** — a 30-year normal is the meteorological standard; ten years was chosen to keep the request affordable.
- Snow, solar radiation and growing-degree-day charts (the variables are already available from the API).
- Dark mode (the CSS variables make this cheap).
