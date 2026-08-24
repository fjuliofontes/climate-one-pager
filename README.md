# Climate One-Pager

**A single HTML file that turns any coordinate on earth into a printable one-page climate report.**

Pick a spot on a map, get twelve months of daily weather rendered as charts, measured against a ten-year normal so you can tell an ordinary year from a fluke. No build step, no dependencies to install, no API key, no backend.

🔗 **[Live demo](https://example.pages.dev)**

<!-- Add a screenshot here once deployed:
![Screenshot](docs/screenshot.png)
-->

---

## Why

Weather sites tell you what tomorrow looks like. This answers a slower question: *what is it actually like to be in this place, all year round?* It was built to evaluate a rural property — how wet the winters are, whether the summer wind makes outdoor space unusable, how many days sit in a comfortable range — but it works anywhere.

The ten-year normal is the part that matters most. A single year is a weak sample. Every chart carries a dashed marker showing the ten-year average behind it, and the page states plainly whether the year you're looking at was typical.

## What you get

| | |
|---|---|
| **Year ribbon** | 365 columns, one per day, spanning that day's low to its high, coloured on a cold-to-hot ramp. Rainfall hangs from the top edge. The whole year at a glance. |
| **Monthly temperature** | Floating bars from average low to average high, whiskers to the most extreme day, and a shaded 18–26 °C comfort band. |
| **Rainfall** | Monthly totals in mm with a wet-day count overlaid — 90 mm over eighteen drizzly days is a different climate from 90 mm in two storms. |
| **Humidity** | Monthly mean with the 10th–90th percentile of daily values shaded around it. |
| **Wind** | Monthly average daily peak with gust whiskers, plus a 16-point wind rose where petal length is frequency and shade is strength. |
| **Day types** | Every month split by share of days by afternoon high, with a rule showing the wet-day share. |
| **Written read** | Several paragraphs of plain-language analysis, generated entirely from the data. |

Everything is hand-drawn SVG. No charting library.

## Using it

**Setup screen.** On first load you're asked for a name and a coordinate. The coordinate box accepts:

```
40.098, -8.7724                  decimal pair
40.098 -8.7724                   space separated
40°05'52.45"N 8°46'18.01"W       degrees–minutes–seconds
40 05 52.45 N 8 46 18.01 W       loose DMS, no symbols
-33.8688, 151.2093               southern / eastern hemisphere
```

It parses as you type and shows you what it read, so a bad paste is caught before anything is fetched.

**Map picker.** "Pick on a map" opens a Leaflet map — click to drop a pin, drag to fine-tune. There's a place-name search too. The map loads lazily, only when asked for, and degrades to manual entry if it can't reach the CDN.

**Every site has its own URL.** Settings live in the fragment:

```
index.html#lat=40.098&lon=-8.7724&name=Alqueid%C3%A3o
```

Opening that link goes straight to the report with no setup. Bookmark one per site and you have a comparison set. This also means no cookies and no browser storage — the URL *is* the save file.

**Save as PDF.** The print stylesheet forces colour output, avoids splitting charts across page breaks, and paginates deliberately to three A4 pages. Use your browser's "Save as PDF" destination. Firefox users need to tick *Print backgrounds*; Chrome, Edge and Safari handle it automatically.

For a fully headless export:

```bash
chrome --headless --disable-gpu --no-pdf-header-footer \
  --virtual-time-budget=25000 \
  --print-to-pdf=site.pdf \
  "file:///path/to/index.html#lat=40.098&lon=-8.7724&name=Alqueidao"
```

The `--virtual-time-budget` matters — without it you get a PDF of the loading screen.

## Deploying

It's one static file. Any host will do.

**Cloudflare Pages, via the dashboard:**

1. Push this repo to GitHub.
2. Cloudflare dashboard → Workers & Pages → Create → Pages → Connect to Git.
3. Pick the repo. Leave the build command **empty** and set the output directory to `/` (or whichever folder holds `index.html`).
4. Deploy.

**Cloudflare Pages, via Wrangler:**

```bash
npm install -g wrangler
wrangler pages deploy . --project-name climate-one-pager
```

There is no build step, no environment variable, and no server-side code. All API calls happen in the visitor's browser.

## Running locally

Open `index.html` in a browser. That's it — no server needed, it works from `file://`.

If you'd rather serve it:

```bash
python3 -m http.server 8000
# then visit http://localhost:8000
```

## How it works

Three requests on load, all client-side:

| Purpose | Endpoint |
|---|---|
| The last ~year of daily data | `archive-api.open-meteo.com/v1/archive` |
| Recent days the archive hasn't settled yet | `api.open-meteo.com/v1/forecast` with `past_days=10` |
| Ten calendar years for the normals | `archive-api.open-meteo.com/v1/archive` |

The archive and forecast results are merged and de-duplicated, preferring the archive where they overlap. This is what keeps the series running right up to today: some reanalysis models lag by a few days, and the forecast endpoint's `past_days` closes the gap.

Timezone is resolved by the API from the coordinates (`timezone=auto`), so it's correct anywhere without configuration.

### API budget

Open-Meteo's free tier allows fewer than 10,000 API calls per day, 5,000 per hour and 600 per minute, counted per IP. Because this app runs entirely in the browser, **each visitor spends their own quota, not yours** — a static host never makes a request.

Be aware the calls are weighted, not counted one-for-one. Per Open-Meteo's [pricing page](https://open-meteo.com/en/pricing), requests covering more than 10 weather variables or more than 2 weeks for a single location count as multiple calls, using fractional counts — a 2-week request with 15 variables counts as 1.5 calls. Applying that formula, one page load costs roughly **200–250 API calls**, dominated by the ten-year normals request. That's fine for individual use (a few dozen loads per visitor per day) but worth knowing before you point a crawler at it.

If that becomes a problem, the obvious fix is caching the normals — they change once a year at most. See `CLAUDE.md`.

## Third-party services

This project makes free use of infrastructure other people pay for. Please respect their terms.

| Service | Used for | Licence / terms |
|---|---|---|
| [Open-Meteo](https://open-meteo.com) | All weather data | Data under [CC BY 4.0](https://open-meteo.com/en/license), attribution required. Free tier is **non-commercial only** — see [terms](https://open-meteo.com/en/terms) |
| [OpenStreetMap](https://www.openstreetmap.org) | Map tiles in the picker | Data © OpenStreetMap contributors, [ODbL](https://www.openstreetmap.org/copyright). Tiles are subject to the [tile usage policy](https://operations.osmfoundation.org/policies/tiles/) |
| [Nominatim](https://nominatim.openstreetmap.org) | Place-name search | Subject to the [usage policy](https://operations.osmfoundation.org/policies/nominatim/) |
| [Leaflet](https://leafletjs.com) | Map rendering | BSD-2-Clause |
| IBM Plex Sans / Mono, Spectral | Typography | SIL Open Font License |

Two things to flag if this gets popular:

- **Open-Meteo's free tier is non-commercial.** Putting ads or a subscription on a site using it makes the use commercial and requires a paid plan.
- **The OSM tile and Nominatim services run on donated infrastructure.** They're fine for a personal tool. If traffic grows, move to your own tile source or a commercial provider — the search is deliberately button-triggered rather than type-ahead for exactly this reason.

## Limitations

Worth being blunt about these, because the charts look more authoritative than the data warrants.

- **This is reanalysis, not observations.** Values are modelled averages over a grid box — roughly 9 km where ERA5-Land covers the location, coarser elsewhere. There is no weather station at your pin.
- **Temperature and wind hold up well at that scale. Rainfall is shakier.** Daily precipitation totals in particular can differ from a local gauge.
- **Local effects vanish.** Fog, cold-air pooling, valley shading, coastal microclimate — all smoothed away by the grid.
- **A 12-month window clips a month at each end.** Partial months are marked with `*` on the charts and excluded from the written analysis, so the month you generate the page in may be missing from the prose.
- **Climate is not risk.** This says nothing about flood risk, fire risk, erosion, or the specific ground you're standing on.

## Contributing

Issues and PRs welcome. The whole thing is one file with no build step — open it, edit it, reload.

If you're using an AI assistant to make changes, `CLAUDE.md` documents the architecture, design system, testing approach, and the specific bugs that have already bitten this codebase once.

## License

MIT License

Copyright (c) 2026 <Fernando Fontes>

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.


---

This license covers the source code in this repository only.

Data and services obtained through this software are licensed separately by
their providers and are not sublicensed here:

  - Weather data from Open-Meteo is licensed CC BY 4.0 and requires
    attribution. Open-Meteo's free API tier is limited to non-commercial use.
    https://open-meteo.com/en/license

  - Map data from OpenStreetMap is licensed ODbL. Tile and geocoding services
    are subject to the OpenStreetMap Foundation usage policies.
    https://www.openstreetmap.org/copyright

  - Leaflet is licensed BSD-2-Clause. IBM Plex and Spectral are licensed under
    the SIL Open Font License. Both are loaded from third-party CDNs and are
    not redistributed in this repository.

Retain the attribution line in the page footer.
