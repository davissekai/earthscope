# EarthScope UI audit and remediation plan

Diagnosis only. This branch does not change application code. The dashboard was run from an external import harness so the current tree could be exercised. That harness is not part of the commit.

The visual identity stays. Later fixes repair behaviour inside the existing palette, type, and layout. Items that would change the look are flagged, with a style-preserving alternative.

## Challenge

The repository never names a Space Apps challenge. The only in-repo mention is the README line that final Space Apps submission materials are not implemented.

The owner’s brief for this audit is the 2026 NASA Space Apps challenge **Be An Earth System Trend Detective!** (Advanced; Earth Science and Software):

https://www.spaceappschallenge.org/2026/challenges/earth-system-trend-detective/

The challenge asks for a tool that selects Earth-science variables, shows a long time series, and says what is changing, where, by how much, and whether that change is statistically significant. The same variable may trend in opposite directions in different regions. Named inputs include MODIS, OMI, AIRS, GPM, MERRA-2, and GLDAS. Linear trends are in scope. The audience includes students and the public, so variable names need plain-language definitions.

Judging is scored 1–5 on Impact, Creativity, Validity (science, feasibility, usability), Relevance (the challenge itself, with NASA data at the core), and Presentation (a working demo and a project page). A submission needs a summary, a public demo, a video of at most 30 seconds or 7 slides, a project link, a description, NASA sources, references, and an AI-use disclosure. At least one NASA data source is required. The event is 14–15 November 2026.

## Fork and upstream

`davissekai/earthscope` `main` and `Mikeswr/earthscope` `main` are the same commit:

`b149bf806ee97ea5e0893e5424eb8a91deb94edf` (12 November 2026), “Standardize displayed units and clarify spatial and forecast data notes”.

Ahead/behind is 0/0. Upstream has no fix that this fork’s `main` lacks. Upstream has no other branches.

The fork has two unmerged branches that `main` does not:

| Branch | What it changes | Does it fix the audit findings? |
| --- | --- | --- |
| `claude/modest-goldberg-cl2t7v` | `tests/test_dashboard.py` expects 8 tabs instead of 10 | Matches the current UI. Does not fix the import failure, so the suite still does not collect on that branch’s CI run. |
| `claude/relaxed-wozniak-jxdh6p` | A few `spatial.*` imports become `earthscope.spatial.*` | Does not remove the empty `earthscope/` directory that shadows the real package. CI on that branch still failed the same import. |

`earthscope_project.zip` (10 November) is an older snapshot. In that snapshot the Python package lived at `earthscope/earthscope/`, and `docs/CREDENTIALS.md` existed. The current tree lifts the modules to the repository root and leaves `earthscope/` containing only `.gitkeep`. That is the import break below.

## Architecture

One Streamlit page, `dashboard/app.py` (about 1,160 lines). There is no separate JavaScript frontend.

| Piece | Role |
| --- | --- |
| `config.py` | Locations, NASA POWER parameter list, units, `OPENWEATHER_API_KEY` and `AGROMONITORING_API_KEY`, data directory. Loads `.env` from the repo root. |
| `data/processed/` | Committed monthly NASA POWER table (18,900 rows), trend results (35 location-variable rows), anomalies, and provenance JSON. Five cities: Accra, Kumasi, Takoradi, Tamale, Wa. Seven variables, 1981–2025 (solar from 1984). |
| `trends/` | Mann-Kendall, Sen’s slope per decade, autocorrelation adjustment, Benjamini-Hochberg FDR, anomalies against 1991–2020. |
| `forecasting/` | Seasonal-naive, trend-plus-season, and damped additive exponential smoothing. Rolling-origin backtest. Empirical 80% and 95% bands. |
| `spatial/power_regional.py` | Live NASA POWER monthly regional API for a Ghana bounding box, tiled, no interpolation. |
| `spatial/map.py` | Folium map of native cells, clipped to geoBoundaries Ghana ADM2. |
| `ingest/openweather.py`, `ingest/agromonitoring.py` | Optional live weather. Not mixed into the NASA trend tables. Agromonitoring NDVI helpers exist and are not shown in the UI. |
| `locations.yaml` | The five city coordinates. |

The page is one script. A hero, three controls (location, variable, chart period), three metric cards, then a Folium overview map, then eight tabs:

Ghana-wide map, Explore time series, Compare trends, Future outlook, Unusual conditions, Data & methods, Weather now, Agromonitoring.

Default selection is alphabetical: Accra, and `ALLSKY_SFC_SW_DWN` (solar energy at the surface). That happens to be the README’s featured series. It is not called out as a featured finding.

### How to run it

Documented command, from the README:

```bash
pip install -r requirements.txt
streamlit run dashboard/app.py
```

That command does not work in a normal checkout. See UI-01. Requirements are unpinned. This audit installed, on Python 3.12.3:

| Package | Version used for the baseline |
| --- | --- |
| streamlit | 1.67.0 |
| pandas | 3.1.1 |
| numpy | 2.5.4 |
| plotly | 7.1.0 |
| folium | 0.20.0 |
| streamlit-folium | 0.27.4 |
| geopandas | 1.2.0 |
| scipy | 1.18.1 |
| pytest | 9.2.0 |

GitHub Actions uses Python 3.11. There is no lockfile, no `pyproject.toml`, and no linter or type-checker configuration.

### Environment

| Name | Required for | Present in this run |
| --- | --- | --- |
| `OPENWEATHER_API_KEY` | Weather now tab | No. `.env` is absent. `.env.example` has an empty value. |
| `AGROMONITORING_API_KEY` | Agromonitoring tab | No. |
| `EARTHSCOPE_DATA_DIR` | Override the data directory | Unset. The committed `data/processed/` files loaded. |
| NASA POWER | Point archive is already committed. The maps call the public regional API and need no key. | Reachable. A one-year sample returned HTTP 200. |
| geoBoundaries Ghana ADM2 | District overlay | Reachable (HTTP 200). |

`docs/CREDENTIALS.md` is referenced from `config.py` and the weather clients, and it is in the November 10 zip, but it is not in the current tree. The UI copy only tells people to add keys to Streamlit Cloud secrets. Local `.env` already works in `config.get_key` and is not mentioned on screen.

No key values were invented for this audit. Weather actions were not called.

## Baseline

Commands below were run on 13 November 2026.

### Boot

From the repository root:

```text
from earthscope import config
ImportError: cannot import name 'config' from 'earthscope' (unknown location)
```

`pytest.ini` puts the repository root on `sys.path`. The empty `earthscope/` directory becomes a namespace package, so `earthscope.config` does not exist. The same traceback is the GitHub Actions failure on `main`:

- Forecasting tests and real-data validation, run 43434232292, failed while collecting tests.
- NASA POWER regional grid smoke test, run 43434231999, failed in `scripts/fetch_power_regional.py` on `from earthscope import config`.
- Real-data forecast validation, run 43434232067, succeeded. That workflow does not import `earthscope`.

`dashboard/app.py` inserts `Path(__file__).parents[2]` on `sys.path`, which only helps when the checkout directory is named `earthscope` and that parent directory is searched before the empty inner folder. The Actions checkout path is `earthscope_f/earthscope_f`, so the hack does not save CI either.

To exercise the UI without editing the tree, the audit symlinked every repository entry except the empty `earthscope/` directory into `/tmp/es-pkg/earthscope` and set `PYTHONPATH=/tmp/es-pkg`. Processed data then loaded: 18,900 monthly rows, 35 trend rows, 5 locations.

`python -m compileall` of the Python packages completed with no errors.

### Lint and types

Ruff 0.17.5, with no project configuration, reported 153 findings. Most are import order, `dict()` calls, blind `except`, and `datetime.UTC` style. One is a real defect: `dashboard/app.py:324` redefines `build_location_boundary_map` while the copy at line 238 is unused (F811).

Mypy 2.4.1, pointed at the repository root, stops immediately because `spatial/power_regional.py` is seen as both `workspace.spatial` and `spatial`. Run against the import harness with `--explicit-package-bases`, it reports 7 errors in 2 files: the duplicate function, an untyped list in `scripts/validate_forecasts.py`, and the Agromonitoring history label dictionary (values are strings where an earlier dictionary in the same file used pairs).

### Tests

With the import harness, excluding the dashboard UI tests:

**173 passed** in 17.8 seconds.

Dashboard AppTests (`tests/test_dashboard.py`):

| Test | Result |
| --- | --- |
| `test_empty_state_shows_instructions_and_no_charts` | Passed |
| `test_short_record_shows_reason_instead_of_a_number` | Passed |
| `test_synthetic_data_renders_with_warning` | Failed: `assert 8 == 10`. The page has 8 tabs. The test still expects 10. |

The synthetic-data run did render, showed the synthetic-data error, and did not raise an app exception. The failure is the stale tab count.

### Browser pass

Local server: `streamlit run` on 127.0.0.1:8501, Chrome 1440×900 and 390×844 (plus 360×740). Every tab was opened. Location was changed to Tamale, variable to corrected precipitation, then back to Accra solar. The statistics expander was opened. The Ghana-wide map was loaded for temperature, mean and trend evidence. Screenshots are in `docs/audit-screenshots/`.

What worked:

- Hero, sidebar, metrics, and the significant-trend banner for Accra solar.
- Location and variable controls, including the “possible trend” warning for Tamale precipitation.
- Comparison table, including Sen’s slope −0.0514 and the FDR p-value 0.0008 for Accra solar.
- Forecast chart, uncertainty bands, and the sentence that the selected method cut backtest MAE by 10.6%.
- Anomaly chart with |z| ≥ 2 markers.
- Data & methods completeness table.
- Overview solar grid (cells, district lines, Accra marker, trend legend).
- Ghana-wide temperature grid: 63 cells, 34,020 monthly records, native spacing 0.500° × 0.625°. Trend evidence shows different classes in different parts of the country.
- Missing-key states for OpenWeather and Agromonitoring, without a crash.

Server logs showed no Python traceback during the session. The browser console had no application exceptions. It did have feature-policy warnings, a Folium iframe sandbox warning, and four OpenStreetMap tile requests aborted with `net::ERR_ABORTED` while the map was re-rendering. Tiles were visible in the saved map screenshots. Details are in `docs/audit-screenshots/console-and-failed-requests.txt`.

The public URL `https://earthscope.streamlit.app/` responds with a redirect into Streamlit’s app gate. This audit did not complete a pass of the deployed site. The findings below are from the local tree.

## What the screen is using

Colours and type already in the page, so later fixes can stay inside them:

- Page `#F7F8F6`, sidebar `#EEF3F1`, body text `#203744`, secondary text `#526875`.
- Hero gradient `#102F40` to `#176E73`, white title, mint eyebrow.
- Chart series `#2386C8` (monthly), `#F2A65A` (annual), `#D95745` (Sen line and forecast).
- Map evidence greens `#16856B` / `#238B57`, reds `#D45B4C` / `#C84C4C`, gold `#D69A32` / `#D99B32`, grey `#87949B`.
- Teal accents `#1C7293` and `#238B83`, note panel `#EAF3F0`.
- White metric cards, 13px radius, hairline `#E2EAE6`.

The hero, cards, and chart colours rendered as designed. The mean-value grid uses a purple-to-yellow ramp that is not in this set (UI-12).

## Issue catalogue

Severity is about a judge or a first-time reader on 14 November, not about code style.

| ID | Sev. | Where | What you see | Cause |
| --- | --- | --- | --- | --- |
| UI-01 | Blocker | `earthscope/` (only `.gitkeep`), `pytest.ini`, `dashboard/app.py` lines 1–23, `spatial/power_regional.py`, `scripts/fetch_power_regional.py` | `streamlit run dashboard/app.py` and `python -m pytest` fail immediately. CI on `main` is red for the same import. | Modules live at the repo root and are imported as `earthscope.*`. The empty `earthscope/` directory is the package Python actually finds. The `parents[2]` path insert only works when the checkout folder is named `earthscope`. |
| UI-02 | High | `dashboard/app.py` time-series trace, around the Sen’s slope `Scatter` | The chart caption and the legend promise a dashed Sen’s slope. No dashed line is drawn. Screenshot `04` and `05`. | The trace is a two-point dashed line. Its computed values sit inside the axes (about 5.02 kWh/m²/day in 1984 and 4.81 in 2025 for Accra solar). Plotly 7.1 under Streamlit 1.67 paints the legend swatch and does not paint the segment. |
| UI-03 | High | `dashboard/app.py` chart-period slider versus `row` from `trend_results.csv`; overview caption uses `display_years` | Moving “Chart period” filters the time-series drawing and the overview map window. The three headline numbers, the evidence banner, the anomaly chart, and the forecast stay on the full record. The side caption still prints the slider years next to that full-record evidence. | `display_years` is applied to `m_display` / `ann_display` and to `build_regional_grid_map`. Metrics read the precomputed full-record row. The slider key changes with location and variable, so each series starts at its full span and the mismatch appears only after the user moves it. |
| UI-04 | High | Overview call in `dashboard/app.py` (`zoom` via `focus_coordinates`); Ghana-wide tab, both `with sptab` blocks; `st_folium(..., returned_objects=[])` | Default solar map is zoomed on southern Ghana and is almost entirely “decreasing, significant”. The national temperature map, which does show different regions, is on another tab, under a five-point chart, behind Load. The tab says “select a cell”. Clicking a cell does nothing. Screenshot `03` versus `15`. | Overview passes the selected city’s coordinates and `spatial/map.py` uses zoom 8. The unfocused map already uses zoom 6 and center `[7.95, -1.05]`. `returned_objects=[]` discards Folium clicks. The Ghana-wide tab is filled twice: first the five-point Plotly map, then the regional explorer. |
| UI-05 | High | `load_regional_grid_cached` and `load_ghana_district_geojson` in the overview, `show_spinner=False`; `fetch_tile` timeout `(20, 180)` and 3 retries, 4 tiles | First paint is a blank white page (`00-blank-first-paint.png`). Nothing in the eight tabs can run until the overview’s full-archive regional request and the district file return. This run came back in seconds and the map rendered. A slow or failing NASA response can hold the whole script for many minutes, because each tile may wait 180 seconds and retry. | The fetch sits in the top-level script, not inside the Ghana-wide tab’s button. The cache hides the spinner. The button path a few hundred lines later is the one that already fails softly. |
| UI-06 | High | `dashboard/app.py` CSS, especially the hero and the `@media (max-width: 768px)` rules; Streamlit header | At 390px the hero eyebrow is cut off under the toolbar (“HANA · EARTH…”), the variable select truncates `ALLSKY_SFC_SW`, tab labels clip (“ditions”), the chart axis title clips, and the map legend covers the map. Screenshot `17`, `18`, `19`. | The custom CSS restacks columns, which works, and does not clear the Streamlit header over the hero. Long labels are the full plain-language name plus the NASA code. The Folium legend is `position:fixed` inside the iframe. |
| UI-07 | High | `spatial/map.py` legend HTML (`position:fixed; bottom; left`) | On desktop and mobile the legend sits on top of the coast and the scale bar. A clipped fragment (“E” or “1”) shows beside the scale. Screenshot `03` and `15`. | The legend is a fixed-position element in the iframe, not a Leaflet control, so it does not reflow with the map. |
| UI-08 | Medium | `number()` uses 3 decimals; the comparison table uses `round(4)` | Accra solar is **−0.051** in the metric and **−0.0514** in the table. The README’s featured figure is −0.0514. | Two formatters. The underlying value is −0.051390. |
| UI-09 | Medium | Comparison table column `evidence_class`; forecast metric `chosen.replace("_", " ").title()`; forecast chart title | The table shows `significant_after_fdr`. The forecast method reads “Trend Seasonal”. The chart title is `Accra · ALLSKY_SFC_SW_DWN statistical outlook`. The banner above the chart is already plain language. | Internal tokens are title-cased or printed raw. `trend_seasonal` is the linear-trend-plus-calendar-month model in `forecasting/engine.py`. |
| UI-10 | Medium | `spatial/grid.py` `point_trend_map`; first block of `with sptab` | The five-point map is a small Africa view. Takoradi (not significant, open circle, slope near zero) reads as the darkest marker because of the dark stroke. The colorbar says `kW-hr/m^2/day/decade` while the rest of the page says `kWh/m²/day`. The hover says to select a point. Selection does not change the study location. This chart is the first thing in the Ghana-wide tab, above the grid that actually answers the question. Screenshot `08` and `14`. | Separate Plotly traces; the not-significant symbol is `circle-open` with a `#333` stroke. The colorbar uses `trends["unit"]` (the NASA spelling) rather than the display unit in `INFO`. `plotly_chart` uses the default `on_select="ignore"`. The session key `overview_location_map` is read at the top of the script and never written. |
| UI-11 | Medium | Landing metrics, success banner, and the “Explore the selected area” column | The same evidence is stated twice: the green banner, then “Evidence: Significant After Fdr” beside the map. | Two summaries were both left in place when the landing page was narrowed around the map. |
| UI-12 | Medium | `spatial/map.py` mean-value `LinearColormap` | The temperature mean layer uses a purple–blue–green–yellow ramp. The rest of the product is teal, sand, and coral. | The colormap is the default Viridis stops, not the colours already used for evidence and charts. |
| UI-13 | Medium | Weather tabs; `config.get_key`; `ingest/agromonitoring.py`; missing `docs/CREDENTIALS.md` | Both weather tabs explain that a key is missing and mention only Streamlit Cloud secrets. With a key, Agromonitoring’s current temperature always subtracts 273.15. The client file still says it has not been verified against a live response. The README says the live panels were tested. | No `.env` in this run, by design. UI copy is narrower than `get_key`, which already reads the environment, Streamlit secrets, and `.streamlit/secrets.toml`. The current-weather path does not use the history parser’s Kelvin check. `VERIFIED_AGAINST_LIVE_API` is `False`. |
| UI-14 | Medium | `tests/test_dashboard.py` | `test_synthetic_data_renders_with_warning` fails: 8 tabs, assertion expects 10. | The page was reduced to 8 tabs. The assertion on `main` was not updated. The unmerged branch `claude/modest-goldberg-cl2t7v` updates only that number. |
| UI-15 | Medium | Data & methods completeness table | Solar rows show 504 of 540 months (93.3%) and timestamps `1981-01-01 00:00:00`. A reader can think solar data is gappy rather than starting in 1984. | `checks.completeness` counts from the file span. The 36 missing solar months are 1981–1983. The table prints full datetimes. The explanation exists in `METHODS.md` and is not next to the table. |
| UI-16 | Medium | Time-series `yaxis_title=f"{name(var)} ({unit})"` | The y-axis title is the long plain-language name. On the page it collides with the plot edge, and on a phone it is cut off. Screenshot `05` and `18`. | Plotly margin does not reserve room for a title of this length. The layout sets height and legend and leaves the default margin. |
| UI-17 | Medium | `build_location_boundary_map` defined at lines 238 and 324; `build_leaflet_location_map`; the `overview_location_map` block | None of these map builders are called. The second definition shadows the first. The click handler at the top of the script looks for a widget that the page never creates. | An earlier district-click design was replaced by the regional grid and left in the file. |
| UI-18 | Low | `fetch_openweather_history_probe` | The tab says “latest four hours” and “four hourly observations”. The request is `now` floored to the hour, minus 3 hours. | Copy and window do not match. Whether the API returns 3 or 4 stamps depends on inclusive endpoints. This could not be checked without a key. |
| UI-19 | Low | Map legends and Plotly charts | Trend class is colour only (green, red, gold, grey). Charts have no text alternative. Map features are not keyboard operable. | Folium and Plotly marks. `st.plotly_chart` is not passed `alt`. The comparison table beside the map does carry the same numbers in text. |
| UI-20 | Low | The large `<style>` block at the top of `dashboard/app.py` | A broad `color: #203744 !important` rule fights the white hero type. Later rules put the hero back. In this run the hero title stayed white and readable. | The override is load-bearing. A future Streamlit markup change can make the title dark-on-teal again. The repair is to narrow the override, not to restyle the hero. |
| UI-21 | Low | Browser console during the session | Feature-policy warnings, one Folium sandbox warning, four aborted OpenStreetMap tile requests. No app exception. Tiles were visible after the map settled. | Browser and iframe noise, plus tile requests cancelled when Streamlit re-ran the script. |
| UI-22 | Low | `README.md` “Not implemented” | The README still says continuous gridded spatial trend maps are not implemented. The running app draws them. | The README was not updated when the regional explorer landed. A judge reading the repo gets the wrong status. |

## Competitive gaps

Checked against this tree and the local session. The owner’s list is kept in their order, then corrected, then ranked again by judging impact against the size of the change. Every approach keeps the current visual identity.

### 1. Lead with the Accra solar finding

**Partly there.** The default controls land on Accra and surface solar energy. The page shows 42 valid years, −0.051 kWh/m²/day per decade, FDR p 0.0008, and “Statistically significant decreasing trend”. The comparison table has the README figures at four decimals (−0.0514, interval −0.0716 to −0.0285, p 0.0008). The hero does not tell that story. It says “Environmental change, made understandable.” Nobody is named (solar planning in Ghana is only in the README). The slope line that would show the decline is not drawn (UI-02), and the metric rounds the featured number (UI-08).

**Judging:** Impact and Presentation. **Effort:** small.

**Approach, same look:** Add one `.note` under the hero, in the existing mint panel, naming Accra, the 1984–2025 decline, the four-decimal slope, and why a solar planner would care. Show four decimals in the existing metric. Leave the gradient, the eyebrow, and the current hero sentence alone. Rewriting the hero would be a visible identity change. The note is the alternative.

### 2. Show where it changes, including different directions

**The grid is real. The default view does not tell this part of the challenge.**

Confirmed in the running app:

- Overview, Accra, solar, trend evidence: native 1.000° × 1.000° cells, clipped to Ghana, almost all “decreasing, significant after FDR”, zoomed on the south. Screenshot `03`.
- Ghana-wide tab, temperature, mean: 63 cells, 34,020 monthly records, spacing 0.500° × 0.625°. Screenshot `16`.
- Ghana-wide tab, temperature, trend evidence: the north around Tamale is increasing and significant (green), a middle band is raw p < 0.05 only (gold), and much of the south, including Accra, is not significant (grey). Screenshot `15`. That is a regional difference. It is not a clean pair of significant opposite signs.
- Among the five cities, no variable has both a significant increase and a significant decrease. Precipitation is the closest: Wa increases and stays significant after FDR; Tamale decreases and does not stay significant after FDR.

**Judging:** Relevance, Impact, and Presentation. This is the challenge sentence. **Effort:** medium. The map code already does the hard part.

**Approach, same look:** On the overview, use the national camera the unfocused map already uses (center about 7.95°N, 1.05°W, zoom 6) so the first screen can show more than one region. Keep the same Folium tiles, cell colours, and legend. In the Ghana-wide tab, put the regional map first and the five-point chart second, still inside the same tab. Default that tab to Trend evidence. Either implement cell click or change the sentence that says “select a cell”; the table under the map already lists the cells. Do not add a new basemap or a new page.

A new national layout, a different legend, or dropping the district lines would be a visible redesign. The alternative is the camera and the tab order above.

### 3. A guided “what / where” flow, and one brief-named variable

**Not guided. NASA data is already the core, and it is easy to miss which NASA products.**

The three controls are a location select, a variable select, and a year slider. Names are plain language plus the NASA code. There is no suggested path. The variables are the seven POWER parameters only. File headers cite MERRA-2, SRB, SYN1DEG, and POWER. MERRA-2 is one of the brief’s named models, via POWER, and the page does not say so. MODIS, OMI, AIRS, GPM, and GLDAS are not used. Agromonitoring’s NDVI helpers (Landsat and Sentinel) are not shown, which is appropriate: they are not the NASA core.

**Judging:** Relevance, and usability under Validity. **Effort:** small for the guide and the MERRA-2 attribution; large for a real GPM or GLDAS series.

**Approach, same look:** One sentence above the existing three controls: start with Accra solar, then open the Ghana map, then try temperature to see a different regional pattern. In the existing source caption, say that POWER’s meteorology is MERRA-2-based. A later, separate change can add one GPM precipitation or GLDAS evaporation series as another item in the same select, the same chart colours, and the same map. Do not add a wizard or a second app.

### 4. A worked example and plain-language definitions

**Definitions are in the product. A worked example is not.**

The sidebar, the metric caption, the variable select, and Data & methods all carry plain-language names. The time-series expander explains Sen’s slope, Mann-Kendall, and FDR in four bullets (screenshot `06`). There is no “follow this example” path that ties the Accra number, the map, and the word “significant” together. The slope line the example would point at is missing (UI-02).

**Judging:** Impact for a public reader, and usability. **Effort:** small, after UI-02.

**Approach, same look:** One more expander, same component as “What do the statistics mean?”, titled as a worked example for Accra solar. Point at the existing metric, the existing map, and the existing table. Do not add a coach-mark tour or a new type style.

### 5. Keep NASA first, and keep live weather secondary

**Largely done. The failure mode is a dead panel with the wrong setup hint.**

Weather now and Agromonitoring are the last tabs. Their copy says they are not the long-term NASA series. They are not merged into the trend math. Without keys they are empty, and the only instruction is “Streamlit Cloud Secrets” (screenshots `12` and `13`). A local `.env` already works and is not mentioned. `docs/CREDENTIALS.md` is missing from the tree. The Agromonitoring client is still marked unverified, and the current-temperature conversion assumes Kelvin.

**Judging:** Validity, so a judge does not think the live widgets are the analysis. **Effort:** small.

**Approach, same look:** Keep both tabs and their current cards. Fix the setup sentence so it matches `get_key` (environment, `.env`, or Streamlit secrets) and restore the credentials note as docs, not as a new banner. If a key is added later, check the current-weather units before subtracting 273.15. Do not restyle these tabs and do not promote them.

### 6. Keep the forecast secondary and the caveats visible

**Done on screen, with a labeling problem.**

The forecast tab opens with a yellow warning that these are statistical extrapolations, not climate projections. The chart, bands, backtest MAE, and the 10.6% skill sentence all rendered for Accra solar (screenshot `09`). `METHODS.md` and `LIMITATIONS_AND_FURTHER_VALIDATION.md` are not linked from the page. The selected method is labelled “Trend Seasonal”, which hides which model won.

**Judging:** Validity. This is a strength if the caveat stays and the label becomes English. **Effort:** small.

**Approach, same look:** Keep the warning colour and the chart. Map the three internal model names to the phrases the expander already uses. Add a text link to `METHODS.md` in the existing caption style. Do not give the forecast a new visual treatment, and do not put it above the historical evidence.

### 7. Optional independent check

**Absent, as the README says.** No station comparison is computed or shown.

**Judging:** Validity, extra credit rather than the core. **Effort:** large.

**Approach, same look:** If a published series for one city and one variable is available, show it as another dataframe in Data & methods, same table style. Do not add a second chart language. Skip this if it would delay the demo.

### Ranked again, for the hackathon

| Rank | Gap | Judging weight | Effort | Style-preserving approach |
| --- | --- | --- | --- | --- |
| 1 | Make the national grid the first answer to “where”, including different regions | Relevance, Impact, Presentation | Medium | Existing Folium map, existing evidence colours, existing national zoom. Reorder the Ghana-wide tab. No new basemap. |
| 2 | A demo that boots and paints | Presentation, usability | Medium | Fix the package shadow (UI-01). Do not call the full regional archive before first paint. Use the Load button that already exists. Same layout when it appears. |
| 3 | Say the Accra solar finding, for a solar planner, in the page | Impact, Presentation | Small | Existing `.note` and the existing metric. Do not rewrite the hero. |
| 4 | Draw the Sen line, and make the year slider honest | Validity | Small | Same `#D95745` dash. Label full-record metrics as full-record, in the current caption style. |
| 5 | One worked example, using the definitions already on the page | Impact, usability | Small | One existing expander. No tour UI. |
| 6 | Name MERRA-2 now; add GPM or GLDAS only if the demo is already solid | Relevance | Small, then large | One clause in the existing source caption. A new variable, if added, reuses the select, the chart colours, and the map. |
| 7 | Leave weather and the forecast secondary, and make their labels true | Validity | Small | Keep the tabs and the yellow warning. Fix copy and the “Trend Seasonal” label. |
| 8 | One external check for a single series | Validity | Large | A table in Data & methods, or skip. |
| 9 | Submission package: summary, public demo, 30-second video or 7 slides, sources, references, AI-use disclosure, an OSI license | Presentation | Medium, mostly outside the app | The README already says this is not done. There is no `LICENSE` file. No change to the product UI. |

## Remediation plan

Phases are ordered so a judge can open the app before the story is polished. Effort is the size of the change: small is a function or a block of copy, medium is one module plus tests, large is a new data source. Nothing here is a calendar estimate.

### Phase 1. Boot, and a stable first paint

| Item | Approach | Depends on | Visible change? | How to verify |
| --- | --- | --- | --- | --- |
| UI-01 | Make `earthscope` a real package again: either move modules back under `earthscope/` as in the November 10 zip, or drop the empty directory and add a package map. Point `pytest` and the scripts at that layout. Delete the `parents[2]` insert once imports work from a checkout of any name. | None | No | `python -c "from earthscope import config"` and `python -m pytest` from a fresh clone. CI forecasting and regional smoke workflows go green. |
| UI-14 | Expect 8 tabs. The unmerged branch already has this one-line change. Land it with UI-01 so collection works. | UI-01 | No | `test_synthetic_data_renders_with_warning` passes. |
| UI-05 | Do not call `fetch_parameter` for 1981–2025 during the first script run. Render the hero, metrics, and charts from the committed point data first. Keep the existing Load button for the regional grid. Turn the spinner back on for that button. | UI-01 | No, if the same sections appear. A spinner on the existing button is the current Streamlit control. | Cold load shows the hero and the Accra metrics without a regional request. The Ghana-wide button still loads 63 temperature cells. A forced NASA failure shows the existing warning and leaves the other tabs usable. |

### Phase 2. Broken behaviour

| Item | Approach | Depends on | Visible change? | How to verify |
| --- | --- | --- | --- | --- |
| UI-02 | Draw the Sen line with one vertex per displayed year, same `#D95745` dash, width 4. Keep the legend label. | Phase 1 | The missing line appears, in the colour the legend already shows. | Accra solar: a red dashed line from about 5.02 to 4.81 across 1984–2025, still visible after a period change. |
| UI-03 | In the metric caption, say the numbers are the full record and print that record’s years. Apply `display_years` to the anomaly chart only if the caption says so. Do not silently recompute FDR inside the slider callback in this phase. | None | Caption text only. | Move the slider to 2010–2025. The chart window changes. The metric still says −0.051 and is labelled 1984–2025. |
| UI-04, UI-07 | Overview uses the existing national zoom unless a later control asks for a city. Ghana-wide tab shows the grid first. Remove “select a cell” until clicks return a selection, or pass Folium clicks through and show the existing cell table row. Move the legend into a Leaflet control in the same white box, same 12px type, same swatches. | Phase 1 for the live fetch | The first map shows more of Ghana. The legend no longer covers the coast. Colours stay. | Screenshot the overview and the temperature trend map. Legend is inside the map frame. Clicking a cell either selects a row or the instruction is gone. |
| UI-06, UI-16 | Pad the hero below the Streamlit header using the existing padding rules. Give the time-series figure a larger left margin. On narrow screens, let the select show the plain-language name and put the NASA code in the caption that already exists. | None | Spacing and clipping only. | 390px: full eyebrow, full axis title, tabs scroll without cutting a word in half, hero is not under the toolbar. Desktop axis title is complete. |
| UI-10 | Give every point trace the same `cmin`/`cmax` (already intended) and a filled marker. Distinguish “not significant” with the existing grey, not a dark open ring. Use the display unit from `INFO`. Wire selection only if it sets `study_location`; otherwise remove “Select point” from the hover. | None | Marker shape for the not-significant class changes from an open ring to a grey filled circle already in the palette. | Takoradi solar is the lightest mark, Accra the darkest blue. Hover does not promise a click that does nothing. |
| UI-13 | Point the empty states at `.env` and Streamlit secrets. Put `docs/CREDENTIALS.md` back from the zip, without adding keys. Do not call Agromonitoring until a probe confirms Kelvin. | A key, only for the live check | Copy only. | Empty state mentions `.env`. With a real key, Accra’s current temperature is a plausible Celsius value, and the key is not in the error text. |
| UI-17 | Delete the unused map functions and the unread `overview_location_map` block, or call one of them. Deleting is the smaller repair. | None | No | Ruff F811 is gone. The page still shows the regional map. |

### Phase 3. The challenge story, still in this visual language

These are the competitive-gap ranks 1 and 3 through 7. They should land after Phase 2 so the story points at things that work.

| Item | Approach | Depends on | Visible change? | How to verify |
| --- | --- | --- | --- | --- |
| Accra note and four decimals | `.note` under the hero. Metric format `,.4f` for the slope. | UI-02, UI-08 | A new sentence in an existing panel. The hero art stays. | The note matches the table: −0.0514, interval −0.0716 to −0.0285, FDR p 0.0008, 1984–2025, named for solar planning in Ghana. |
| National first view | Phase 2 camera and tab order. Default the Ghana-wide control to Trend evidence. | UI-04 | What the map frames, not how it is drawn. | First screen shows more than the Accra district. Temperature trend map still shows green in the north and grey in the south. |
| Worked example | One expander that walks Accra solar in four steps, using the words already in the statistics expander. | The note and UI-02 | One more expander, same component. | A reader who opens only that expander can answer what, where, how much, and whether it is significant. |
| MERRA-2 attribution | One clause in the existing NASA caption. | None | Copy only. | The caption says POWER and MERRA-2. The trend math is unchanged. |
| Forecast label and methods link | A three-row label map. A caption link to `METHODS.md`. | None | Copy only. The yellow warning stays. | The metric says the seasonal-trend model in words. The warning is still the first thing in the tab. |
| Weather stays last | No move required. Phase 2 copy is enough. | UI-13 | No | The eight tabs are unchanged in order. |

Items that would visibly change the look, and the alternative:

| Tempting change | Why it is a look change | Do this instead |
| --- | --- | --- |
| Rewrite the hero into a case-study banner | New hierarchy, new headline, less of the current wordmark | Keep the hero. Use the existing `.note`. |
| Replace Folium with a new map library or a new tile style | New cartography | Keep Folium, OpenStreetMap, and the evidence colours. Change zoom and legend placement. |
| Recolour the whole product to a stricter palette | UI-12 is the one off-palette ramp | Recolour only that mean-value ramp with the existing teal `#176E73` and sand `#F2A65A`. Leave evidence greens and reds as they are. |
| A multi-step onboarding modal | New surface | One expander. |
| Drop the weather tabs or restyle them as a warning stripe | Removes part of the current page | Keep the tabs. Correct the sentences. |

### Phase 4. Polish, access, and optional science

| Item | Approach | Depends on | Visible change? | How to verify |
| --- | --- | --- | --- | --- |
| UI-09, UI-11, UI-15, UI-18, UI-22 | Plain-language evidence labels in the table, one evidence summary instead of two, “solar starts in 1984” beside the 93.3% cell, a matching OpenWeather window, and a README that admits the grid exists. | None | Copy and one less repeated paragraph. | Table says “Significant after FDR”. Solar completeness mentions 1984. README matches the app. |
| UI-12 | Swap only the mean-value colormap stops to teal and sand. | None | The mean layer changes colour. The alternative, if that is too visible, is to leave Viridis and accept the inconsistency. | Mean map legend uses the two existing colours. Trend-evidence colours are unchanged. |
| UI-19, UI-20 | Add `alt` on Plotly charts. Add the words already in the legend (“increasing”, “decreasing”) next to the swatches. Narrow the global `!important` colour rule so it does not cover the hero. Keep the hero’s own white-text rules. | None | None, if the hero still matches screenshot `01`. | Hero title is white on the gradient. A chart `alt` string is present. Legend text does not rely on colour alone. |
| UI-21 | No product change unless aborted tiles leave a blank map. They did not in this run. | None | No | A reload still shows tiles. |
| Gap 8 | One external series, one table. | A citable series | No new chart style. | The table states the source, the years, and the difference from POWER. |
| Gap 6, the large half | One GPM or GLDAS variable through the existing select and map. | A stable demo | No new page. | The new series is labelled as that mission, and it is not mixed into the POWER FDR family by accident. |
| Submission | Project page, short video or seven slides, sources, references, AI-use note, OSI license. | A demo that survives Phase 1 and 2 | Not an app visual. | The project page links to a public app that shows the Accra finding and the national temperature map. |

## Suggested first session after this audit

Phase 1, then UI-02 and UI-03. Those five items turn a red CI run and a blank first paint into a page whose main chart actually draws the trend it describes. The national map and the Accra note come next, because they are what the challenge asks to see, and the map code is already drawing them.

## Screenshot index

Files are in `docs/audit-screenshots/`.

| File | Screen |
| --- | --- |
| `00-blank-first-paint.png` | First paint, before the script finishes the regional fetch |
| `01-landing-desktop.png` | Landing, Accra solar metrics |
| `02-tamale-precipitation.png` | Location and variable change, Tamale precipitation |
| `03-overview-solar-trend-map.png` | Overview solar grid, trend evidence, southern zoom |
| `04-timeseries-slope-line-missing.png` | Time series with the Sen line in the legend only |
| `05-timeseries-page.png` | Time series tab in the page |
| `06-timeseries-expander.png` | Statistics expander |
| `07-compare-trends.png` | Comparison table |
| `08-five-point-map.png` | Five-city Plotly map |
| `09-forecast.png` | Forecast tab |
| `10-anomalies.png` | Anomaly chart |
| `11-data-methods.png` | Completeness table |
| `12-weather-missing-key.png` | OpenWeather without a key |
| `13-agromonitoring-missing-key.png` | Agromonitoring without a key |
| `14-ghana-tab-trend-controls.png` | Ghana-wide tab after loading temperature trend evidence |
| `15-ghana-temperature-trend-map.png` | National temperature trend grid |
| `16-ghana-mean-grid-loaded.png` | Temperature mean grid, cell count and spacing |
| `17-mobile-landing.png` | 390px landing |
| `18-mobile-timeseries.png` | 390px time series and tabs |
| `19-mobile-weather.png` | 390px weather tab |
| `20-mobile-360-weather.png` | 360px weather tab |
| `console-and-failed-requests.txt` | Console warnings and aborted tile requests |
