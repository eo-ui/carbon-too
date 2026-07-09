# Carbon Tracker

A guided carbon accounting tool built with **Django + htmx + Bootstrap**. It walks users
through everyday questions ("how many monitors does the team use?", "how do people travel
into work?") and maps the answers to a **GHG Protocol** breakdown across **Scopes 1, 2 and 3** —
no carbon-accounting knowledge needed from the person filling it in.

## Features

- **Guided wizard** — five short steps of plain-English questions, swapped in and out with
  htmx (no full page reloads), with a progress bar, back/next, and save-and-resume.
- **Three report templates**, chosen up front when the user sets up a report:
  - **Client report** — polished external summary with scope breakdown and methodology note
  - **Bid summary** — compact one-pager ready to drop into a tender response
  - **Internal review** — every activity line, per-person figures, reduction priorities and
    data-quality notes
  You can also switch between layouts on the report page (`?as=client|bid|internal`) and
  print to PDF via the browser.
- **Transparent calculations** — every emission factor and assumption lives in one file,
  `emissions/factors.py`, mapped to GHG Protocol scopes and categories.
- **Commuting survey import** — upload the annual commuting survey spreadsheet on the
  "Getting to work" step and the answers fill in automatically.
- **Annual factor updates** — import the gov.uk conversion factors flat file each June
  with one command; reports state which factor set they used.
- **Year-on-year comparison** — a Compare page charting every completed report by scope,
  with per-person figures and % change.

## Quick start — Windows (VS Code PowerShell)

Run these **one line at a time** in the VS Code terminal, from the project folder:

```powershell
# 0. Check Python is installed (3.10+). If this fails or opens the Microsoft
#    Store, install it from https://www.python.org/downloads/ and TICK
#    "Add python.exe to PATH" in the installer, then reopen VS Code.
py --version

# 1. Create and activate a virtual environment
py -m venv venv
venv\Scripts\Activate.ps1

# If activation is blocked with "running scripts is disabled on this system":
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
# ...then run venv\Scripts\Activate.ps1 again. You should see (venv) in the prompt.

# 2. Install, set up the database, run
pip install -r requirements.txt
python manage.py migrate
python manage.py runserver
```

Open http://127.0.0.1:8000 and click **New report**.

## Quick start — macOS / Linux

```bash
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
python manage.py migrate
python manage.py runserver
```

## Project layout

```
emissions/
  steps.py      # the wizard: questions, wording, help text — edit freely
  factors.py    # emission factors + the calculation engine
  models.py     # Assessment model (answers stored as JSON)
  views.py      # wizard flow (htmx partials) + report rendering
  templates/emissions/
    _step.html            # htmx step partial
    reports/client.html   # the three report layouts
    reports/bid.html
    reports/internal.html
```

## Importing the annual commuting survey

Each assessment has an import page (linked from the "Getting to work" step). Upload the
survey as `.xlsx` — one row per person, with a column for travel mode and one for one-way
distance (miles or km both work; a "days in office" column is used too if present).
Column headings are matched by keyword, so the survey doesn't have to follow an exact
template — but `commuting-survey-template.xlsx` in this folder is a ready-made one you
can hand to whoever runs the survey.

After import you'll see a summary of what was recognised, and the values pre-fill the
wizard where they can still be adjusted. Unrecognised travel modes are counted as car
(the cautious option) and flagged.

## Automatic conversion factor updates (GOV.UK checker)

The UK Government publishes new conversion factors every June, including a **flat file**
Excel version designed for automated processing. The checker finds and imports it in one
step:

```powershell
python manage.py check_defra_factors --year 2026
```

What it does:
1. Asks the GOV.UK Content API
   (`gov.uk/api/content/government/publications/greenhouse-gas-reporting-conversion-factors-<year>`)
   for the publication's attachments and picks the `.xlsx` whose title contains "flat".
   Falls back to scanning the HTML page for an `assets.publishing.service.gov.uk` link.
2. Downloads it and compares its SHA-256 hash against the last import — if nothing
   changed, it stops.
3. Parses the flat file and **stages** the factors (never auto-published), running
   change detection against the factors currently in use: changes above **5 % for
   Scope 1 & 2** or **10 % for Scope 3** are flagged ⚠ for review.
4. You review the printed change report (or the Factor sets screen in Django admin at
   `/admin/`), then publish:

```powershell
python manage.py publish_factors 2026
```

Run the checker without `--year` and it tries the current year, then the previous one —
handy for a scheduled job (Windows Task Scheduler, cron, or an Azure Function timer)
through June/July. `--publish` stages and publishes in one go if you'd rather.

**Manual fallback** (checker can't reach gov.uk, or you already have the file):

```powershell
python manage.py update_factors --file "C:\Downloads\flat-file.xlsx" --year 2026
```

Same staging and change detection; publish the same way.

Two honest caveats:
- The wording in the flat file drifts slightly between years. If a factor shows ✗,
  adjust its matchers in `emissions/flat_file.py` (they're simple substring rules).
- Flight factors are published per passenger-km, but the wizard asks in return trips, so
  they're converted using representative distances in `ASSUMED_FLIGHT_KM`
  (`emissions/factors.py`) — review those if your travel profile is unusual.

Always spot-check a few imported values against the workbook before formal use.

## Year-on-year comparison

Click **Compare years** on the home page. Every completed report is recalculated with
the *current* active factor set so the trend is like-for-like, then shown as a stacked
scope chart and a table with per-person figures and % change against the previous report.

## Database (local vs Azure)

Django always uses a database — out of the box this is **SQLite**, a single `db.sqlite3`
file created by `migrate`, which is fine while the tool runs on one machine. Once
several people need to use it, move to **Azure Database for PostgreSQL (Flexible
Server)**:

1. Create the server in the Azure portal and note the host, admin user and password.
2. Add its firewall rule / VNet access for wherever the app runs (e.g. Azure App Service).
3. `pip install "psycopg[binary]"` and set these environment variables (App Service →
   Configuration → Application settings):
   `DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASSWORD` (and optionally `DB_PORT`).
4. Run `python manage.py migrate` once against the new database.

`settings.py` switches to PostgreSQL automatically when `DB_HOST` is set (with SSL,
which Azure requires) and falls back to SQLite when it isn't — no code changes needed.

## Customising

- **Change or add questions**: edit `emissions/steps.py`. Each question is a small dict
  (key, label, help, unit, type). New numeric answers become available in
  `factors.calculate()` via `answers["your_key"]`.
- **Update emission factors**: everything is in `emissions/factors.py` with comments.

## ⚠️ Before formal use

The conversion factors shipped here are **indicative UK values** for demonstration.
Before using outputs in client deliverables or bids, replace them with your
organisation's approved conversion factors for the correct reporting year (e.g. the
current DEFRA/DESNZ set), and have the methodology reviewed by whoever owns
sustainability reporting internally. The tool is designed for estimation and
transparency, not assured reporting.

## Production notes

- Set `DEBUG = False`, a real `SECRET_KEY`, and proper `ALLOWED_HOSTS` in
  `carbontool/settings.py`.
- Add authentication (e.g. `django.contrib.auth` login-required decorators, or your SSO)
  before deploying anywhere shared — the current build is single-user/local.
- SQLite is fine for a small internal tool; swap `DATABASES` for Postgres if it grows.
