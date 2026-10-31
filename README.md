# CWQT Data Portal

The **Community Water Quality Testing Data Portal**: an interactive map and charts of Enterococcus
results from the CWQT program, based on the CWQT master Google Sheet.

**Data Portal:** <https://SWIMCoalition.github.io/CWQT-Data-Portal/>

**More about CWQT:** <https://www.swimmablenyc.org/cwqt>

Results entered in the master sheet automatically appear on the map the next morning.

---

## What's in this repository

| File | Purpose |
|---|---|
| `cwqt_visualize.py` | Reads the workbook, calculates the statistics and writes every output. All thresholds and definitions are in its **CONFIGURATION** section near the top. |
| `templates/` | The web pages' design: `map.html` (map page), `charts.html` (charts page), `cwqt.css` (shared colors, fonts, banner). Edit these to change how the portal looks; see [Changing the look](#changing-the-look-of-the-portal). |
| `requirements.txt` | Python packages the script needs. |
| `.github/workflows/update-map.yml` | The daily job: download the sheet → run the script → publish the site. |
| `status.json` | Written by the daily job: latest sample date, sample count and site count. A new commit means new data arrived. |
| `.gitignore` | Keeps the workbook and generated files out of the repository. |

## How the numbers are calculated

Each run, the script recalculates the **30 Day GeoMean Primary Contact** tab and prints whether 
every site matches.

- **Censored results:** `<10` counts as 10, `>24196` as 24196 and `>6000` as 6000.
- **Geometric mean** = exp(mean(ln(MPN))).
- **Categories** (NYC DOH primary contact, same as the sheet's green/yellow/red):
  - Good: geomean ≤ 35
  - Advisory: 35–104
  - Poor: > 104
- **Single-sample limit** for "% of samples over": 104.
- **Fewer than 5 samples:** a geomean from fewer than 5 samples is shown with a dashed / hollow
  marker, because standards require at least 5.
- **NYS DEC criteria** shown in map popups:
  - Class I: geomean 72, threshold 266
  - Classes SB and SA: geomean 35, threshold 130
- **Regular season** = May 1 – Oct 5. Seasonal statistics use only these samples. Winter and
  post-season samples appear in the "Latest 30 days" layer but not in seasonal geomeans.
- **Wet weather** = at least 0.25" of rain at Central Park on the sampling day plus the 2 days
  before. Precipitation text is cleaned up first: "Trace" counts as 0, and "0.45 (3.2 snow)" counts
  as 0.45.
- **Lab/field splits** recorded in Notes are ignored as of now, so the numbers match the GeoMean
  tabs. Run with `--average-splits` to average them as the FAQ describes.

To change any of these, edit the CONFIGURATION section of `cwqt_visualize.py`
(`GOOD_MAX`, `ADVISORY_MAX`, `SEASON_START`, `WET_WEATHER_INCHES`, `RAIN_DAYS`, `DEC_CRITERIA`, …).
The next daily run uses the new values.

### What the script expects in the sheet
Columns are found **by their header names**, so moving or inserting columns is fine.
- **Required.** Renaming any of these stops the daily run:
  - Data tab: `Site`, `Full Date` and
    `Most Probable Number (MPN) of Enterococcus colonies per 100 ml`.
  - Information tab: a header row containing `Table of Contents - Site (Listed Alphabetically)`
    followed by `Site ID`, plus `Currently Testing`, `Waterbody`, `Borough/State`,
    `Neighborhood/Town`, `Latitude`, `Longitude` and `NYS DEC Water Body Classification`.
- **Used if present.** Renaming one of these quietly turns off the feature that depends on it:
  - `Site ID` (otherwise sites are matched by name);
  - `Notes` (lab splits and QC notes);
  - the seven precipitation columns, `Day of Collection Precipitation (Thursday)`,
    `Previous Day Precipitation (Wednesday)`, `Previous Tuesday Precipitation`, … (wet/dry weather).

New sites only need a row on the Information tab (with coordinates) and a matching Site ID in Data.

---

## The daily update

`.github/workflows/update-map.yml` runs every day at 11:17 UTC (7:17 am EDT / 6:17 am EST) on
Ubuntu 24.04:

1. Downloads the sheet as .xlsx, using the `CWQT_SHEET_ID` secret (plus
   `GOOGLE_SERVICE_ACCOUNT_JSON` if the sheet is private).
2. Runs `python cwqt_visualize.py --out build`.
3. Copies the public files to `site/` and deploys them to GitHub Pages.
4. Commits `status.json` if the data changed. If 45 days pass with no commit (e.g. off-season), it
   makes an empty "keep-alive" commit instead, because GitHub pauses scheduled jobs in repositories
   with no commits for 60 days.

**Update now:** Actions → *Update CWQT map* → **Run workflow**.

**Check it's working:**
- The map title says "Data through <date>".
- The Actions tab shows a green check for each morning.
- The commit history shows "Data updated: …" after new results are entered.

### If a run fails
GitHub emails the person who set up the workflow (or who last changed its schedule line). Open the failed run → `update` job to see the
error, then check:

| Message | Likely cause / fix |
|---|---|
| "Could not download the Google Sheet" / "Google returned a web page" | The sheet's sharing changed, the sheet ID secret is wrong, or the service-account key was deleted. |
| "Data tab is missing expected column(s)" / "Could not find a header row" | A column or tab in the sheet was renamed. Rename it back, or update the names in the CONFIGURATION section. |
| Fails in "deploy" | Settings → Pages → Source must be **GitHub Actions**. |
| No runs at all for weeks | The schedule was paused. Open Actions → *Update CWQT map* → **Enable workflow**. |

GitHub occasionally retires old versions of the actions used in the workflow (`actions/checkout`,
`setup-python`, …). Warnings about this appear as "annotations" on a run. Updating the version
numbers in the `uses:` lines fixes them.

---

## Changing the look of the portal

The design lives in `templates/` and is separate from the calculations. Edit those files on GitHub
(pencil icon → commit), then run the workflow (or wait for the next morning) to publish.

| To change… | Edit |
|---|---|
| Colors, fonts, button style, banner height | the **SWIM BRAND** block at the top of `templates/cwqt.css` |
| Banner title, button labels, popup wording, legend text | `templates/map.html` |
| Chart descriptions and page text on the charts page | `templates/charts.html` |
| The logo | It's loaded from SWIM's Wix site (`LOGO_URL` in `cwqt_visualize.py`). To host it here instead, save it as `templates/assets/swim-logo.png`. |
| Basemap choices or the streams overlay | the `BASEMAPS` and `WATERWAYS` lists near the top of the script in `templates/map.html`. `WATERWAYS.minZoom` sets how far in you must zoom before streams appear. |
| Thresholds, categories, season dates | the CONFIGURATION section of `cwqt_visualize.py` (used by the charts *and* the map) |

Words in double curly braces in the templates (e.g. `{{DATA}}`, `{{N_SITES}}`) are filled in by the
script. Keep them where you want those values to appear.

**Adding map layers later** (e.g. CSO outfalls, green infrastructure): put the data file (GeoJSON) in
`templates/assets/`, then add an entry to the `EXTRA_LAYERS` list in `templates/map.html`. There's a
commented example there. Each entry appears as its own group with a checkbox in the Layers menu.

Shapefiles and KML files need converting to GeoJSON first. Do it once per dataset, e.g. at
<https://mapshaper.org>: drop in the file (for a shapefile, all of its parts: `.shp`, `.shx`, `.dbf`
and `.prj`), then Export → GeoJSON. While there:
- Make sure it's in latitude/longitude. NYC data often comes in State Plane feet; in mapshaper, type
  `-proj wgs84` in the console.
- Simplify large drainage-area polygons (Simplify, ~10–20%) so the file stays small and fast.

**Current design** (October 2026), matched to swimmablenyc.org:
- Poppins font, SWIM navy `#092e6e` with square-cornered buttons, SWIM blue `#0d4ab2` links, cream
  `#f7f7f3` page background.
- Light gray basemap with the USGS streams and rivers overlay on top, so narrow waterways like the
  Bronx River stand out. The overlay appears from the ~1-mile zoom level, because USGS labels every
  lake and reservoir at the citywide view. "Streets" and "Satellite" basemaps are also available.
- On phones the legend starts collapsed, button labels shorten, and tap targets are larger.

---

## Running it on your own computer

```bash
pip install -r requirements.txt

# from a downloaded copy (Google Sheets → File → Download → Microsoft Excel)
python cwqt_visualize.py CWQT_Master_Data_Sheet.xlsx

# or straight from Google Sheets
python cwqt_visualize.py --sheet-id <long code from the sheet's URL>
```

Options:

| Option | Effect |
|---|---|
| `--out FOLDER` | Where outputs go (default `cwqt_outputs/`). |
| `--window-end YYYY-MM-DD` | End date of the 30-day window (default: latest sample). |
| `--average-splits` | Average lab/field split results recorded in Notes. |
| `--site-pngs` | Also save one PNG per site. |
| `--no-map` | Skip the web pages (map and charts page). |

Outputs go to `cwqt_outputs/` (or the `--out` folder). Open `index.html` there in a browser to
preview the portal before publishing.

---

## Known data issues to fix in the sheet
Run the script locally and open `data_quality_report.txt` for the current list. Found in October 2026:
- 8 duplicate site+date rows.
- 5 rows with no MPN result.
- 4 site names in Data that differ from Information.

Fixed in October 2026: site 160's (Hackensack) positive longitude, and the 2026 rainfall, which is now
entered, so 2026 has a wet/dry split. The script still checks for both, in case they recur.
