# Scraper Manual Tool

A Python + Selenium scraper that collects business details from **Yelp** and exports them to CSV. 
Includes both a **command-line tool** and an optional **graphical interface (GUI)** for ease of use.

## What it extracts

Per business, into these CSV columns:
**Business Niche, Company Name, Location: USA, Phone Number, Yelp URL, Website URL**.

- **Business Niche** — the search term used (e.g. "restaurants", "plumbers").
- **Location: USA** — the business's full address (street, city, state, zip).
- **Website URL** — the external business site Yelp links out to (via its
  `/biz_redir?url=...` anchor).

## How it works

1. Opens a Yelp search (`/search?find_desc=<term>&find_loc=<location>`) and
   collects `/biz/<slug>` listing URLs, paginating via `&start=` in steps of 10
   until `max_results` is reached (or `--max-all` to scrape every available result).
2. Visits each business page and reads the embedded **JSON-LD**
   (`<script type="application/ld+json">`, schema.org `LocalBusiness`) for the
   name, telephone, and address. Falls back to label-based XPath/regex for the
   phone if JSON-LD is missing, and to a `/biz_redir` anchor scan for the website.
3. Simulates human behavior with scrolling, random delays, and realistic browser properties
   to avoid detection by Yelp's anti-bot protection (DataDome).
4. Writes all rows to the output CSV (`utf-8-sig` so Excel opens it cleanly).

## De-duplication / append mode

With `append: true` (the default) the scraper **merges** into the existing
output CSV instead of overwriting it, keyed on the `Yelp URL` column:

- URLs already present in the CSV are **skipped before scraping** (fewer
  requests = less chance of being blocked).
- Re-scraped businesses **refresh** their existing row rather than adding a
  duplicate; the newest scrape wins.
- This lets you run several searches (different terms/cities) into one growing,
  duplicate-free file.

Set `append: false` in `config.json` (or pass `--no-append`) to overwrite the
file each run instead.

---

## Setup

### Requirements

- **Python 3.10+**
- **Google Chrome** installed

Selenium 4.6+ downloads the matching chromedriver automatically (Selenium Manager)
— no manual driver setup needed.

### Python dependencies

Install from `requirements.txt`:
```bash
pip install -r requirements.txt
```

This includes: **Selenium 4.6+**, **undetected-chromedriver** (for stealth), and dependencies.

---

## Run

### Command line

```powershell
# Uses config.json defaults
python scraper.py

# Override on the command line
python scraper.py --term "plumbers" --location "Austin, TX" --max 30
python scraper.py --term "restaurants" --location "Boston, MA" --max-all
python scraper.py --headless --out results.csv
python scraper.py --no-append            # overwrite output.csv instead of merging
```

CLI flags override `config.json`. Available flags:
- `--term TEXT` — search term (e.g., `"plumbers"`)
- `--location TEXT` — location (e.g., `"Austin, TX"`)
- `--max N` — max businesses to collect
- `--max-all` — collect every result Yelp returns (ignores `--max`)
- `--out FILE` — output CSV path
- `--headless` — run without a visible window (cannot solve CAPTCHAs)
- `--no-append` — overwrite CSV instead of merging

### GUI (graphical interface)

Alternatively, run the Tkinter-based GUI:

```powershell
python gui.py
```

**Features:**
- Simple form to enter search term, location, and max results
- **Run scraper** button — collect up to your specified max
- **Get MAX leads** button — collect every result Yelp returns (ignores the max field)
- **Stop** button — gracefully save results and stop the scraper mid-run
- Headless mode checkbox (enabled = no visible window, *cannot solve CAPTCHAs*)
- Append/merge checkbox (enabled = merge into existing CSV, skip duplicates by Yelp URL)
- Live log pane showing CAPTCHA prompts, progress, and errors in real-time

The GUI runs `scraper.py` as a subprocess with the flags you enter — all scraping logic is identical to the CLI.

---

## Configure

Edit `config.json` to set defaults:

```json
{
  "search_term": "restaurants",
  "location": "San Francisco, CA",
  "max_results": 10,
  "max_all": false,
  "headless": false,
  "output_csv": "output.csv",
  "append": true,
  "page_load_timeout": 30,
  "delay_min": 12.0,
  "delay_max": 15.0
}
```

| Setting             | Meaning                                                        |
|---------------------|----------------------------------------------------------------|
| `search_term`       | What to search for (e.g. `"plumbers"`). Becomes "Business Niche". |
| `location`          | Where to search (e.g. `"Austin, TX"`).                        |
| `max_results`       | How many businesses to collect (if `max_all` is `false`).     |
| `max_all`           | `true` = collect every result Yelp returns (ignores `max_results`). |
| `headless`          | `true` = no visible window. **Keep `false`** so you can solve CAPTCHA challenges. |
| `output_csv`        | Output file name.                                              |
| `append`            | `true` = merge into existing CSV and skip duplicates by Yelp URL. |
| `page_load_timeout` | Per-page load timeout in seconds.                              |
| `delay_min`/`max`   | Random pause (seconds) between requests. Raise if you get blocked. |

---

## Important: Yelp anti-bot protection (DataDome)

Yelp serves a "verify you are a human" CAPTCHA (a `geo.captcha-delivery.com` iframe) 
on search and business pages, protected by DataDome. The scraper uses multiple stealth 
techniques to minimize detection:

- **undetected-chromedriver** — patches Selenium fingerprints automatically
- **Realistic browser profile** — window size, language, user-agent, and hardware properties
- **Human-like scrolling** — simulates reading behavior with pauses and drift-back
- **Randomized delays** — between requests (configurable via `delay_min`/`delay_max`)
- **Persistent Chrome profile** (`.chrome-profile/`) — reuses CAPTCHA clearance cookies across runs

### Solving CAPTCHAs

**On startup**, the scraper opens google.com then yelp.com as a warm-up. **If the CAPTCHA 
appears, solve it once in the visible Chrome window.** DataDome then sets a clearance 
cookie that persists in the Chrome profile, so subsequent pages load cleanly for the rest 
of the run.

**This only works non-headless** (`headless: false`, the default). Headless mode cannot solve CAPTCHAs.

### Block recovery

The scraper can automatically recover from some blocks by clearing browsing data (history, 
cookies, cache, site data) and refreshing the page — this often clears DataDome blocks outright 
before you ever need to solve a CAPTCHA.

### If you get hard-blocked

A "You have been blocked" page (not a solvable CAPTCHA) means DataDome has blacklisted your 
IP/network. No browser tweak gets past this. Options:

1. **Switch networks** — phone hotspot, VPN, or residential proxy (most effective)
2. **Wait** — IP blocks typically expire after a few hours
3. **Clean slate** — delete `.chrome-profile/` and retry

Then re-run. Alternatively, slow down by raising `delay_min`/`delay_max` in `config.json` 
or lowering `max_results` to reduce request volume.

---

## Output

The output CSV has these columns:

| Business Niche | Company Name | Location: USA | Phone Number | Yelp URL | Website URL |
|----------------|--------------|---------------|--------------|----------|-------------|

**Website URL** is the external business site Yelp links to. Email is not 
collected (Yelp does not publish it).

With `append: true`, running again merges new businesses into the same file and 
refreshes existing rows (matched by Yelp URL), never creating duplicates — so you 
can run several searches (different terms/cities) into one growing file.

---

## Project layout

- `scraper.py` — main scraper (Yelp search → per-business extraction → CSV merge/write)
- `gui.py` — optional Tkinter-based graphical interface with live log pane
- `config.json` — default settings (term, location, limits, delays, output path)
- `requirements.txt` — Python dependencies
- `output.csv` — generated results (created on first run)
- `.chrome-profile/` — persistent Chrome profile with CAPTCHA clearance cookies

---

## Tuning

Edit `config.json`:

- `headless` — `true` runs without a visible window (more likely to be flagged).
- `delay_min` / `delay_max` — seconds of random pause between requests; raise
  these if you get blocked.
- `page_load_timeout` — per-page load timeout in seconds.

---

## Important notes

- **Email is not collected.** Yelp does not publish business email addresses;
  the `Website URL` column is the practical contact link.
- **Terms of Service.** Yelp's ToS prohibit automated scraping. Keep volumes
  low and use only for authorized/personal purposes.
