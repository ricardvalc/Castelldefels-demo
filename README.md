# FCF prototype — UE Castelldefels

Internal prototype. Searchable database built from Catalan federation match
reports (actes). **Not for distribution** — see "Legal" below.

## What works today

- `data/actes/3784273.json` — one fully parsed acta:
  Castelldefels 0–1 Júpiter, Lliga Elit Grup 1, jornada 30, 16/05/2026.
- `pipeline/load.py` — loads acta JSON into SQLite, runs reconciliation
  checks, and exports the web bundle.
- `web/index.html` — client-facing squad view (FIFA-style player cards).
- `web/search.html` — internal search across everything in the actes.

Both are **generated**. Edit `web/template-squad.html` and
`web/template-search.html`, then re-run the loader; it inlines the data so
each page is a single self-contained file that opens anywhere with no
server and no sibling files.

```
python3 pipeline/load.py
open web/index.html
```

## Findings on the FCF site

| Page | Server-rendered? | Notes |
|---|---|---|
| `/ca/competicio/acta/{id}` | **Yes** | Full acta in the HTML. Plain HTTP scrape works. |
| `/ca/clubs/{club}/categories/{team}` | No | "Carregant dades del equip…" — client-side XHR |
| `/ca/competicio?temporadaId=&grupId=` | No | Same |

Known IDs: club 1150 = Castelldefels, U.E.; team 33045; grup 54322937
(Lliga Elit Grup 1); temporadaId 22 = season 2025/26.

**Blocker:** we can fetch any acta by ID, but nothing yet enumerates the
acta IDs for a season. Next step is capturing the XHR the group page fires.

## What an acta gives us

Per match: competition, group, matchday, date, kickoff, venue, acta status,
three referees, goals (minute, scorer, dorsal, type: normal / penal / pròpia),
both XIs with dorsal and captain, goalkeepers flagged, bench with who came on
and when, every card with its minute, full technical staff by role, and staff
cards. Minutes played are derivable from the substitution minutes.

That is enough for minutes, availability, discipline and rotation analysis —
the core of the product.

## Data quirks found in the first acta

- A cards shown as "(Final)" carry no minute. Stored as `minute: null,
  note: "final"`, so they don't corrupt minute-based analysis.
- Júpiter list their delegat (Martinez Simon, Lorenzo) among the substitutes
  with dorsal 0. The validator flags dorsal 0 rather than silently creating
  a phantom player.
- The "C" badge means captain on a player row, but coordinator in the
  technical-staff legend. Same glyph, two meanings — worth confirming across
  more actes before trusting it.
- Names are upper-case and inconsistent on accents (GARCIA vs GARCÍA appear
  in the same document). `people` / `person_aliases` exist for this.

## Validation checks

`load.py` reconciles every acta and reports FAIL or WARN:
goal events vs scoreline; XI size = 11; every substitute who came on is on
the bench list; minutes played total 990 per team; dorsal 0 present.

## Legal

FCF's avís legal reserves reproduction and public communication of site
content for commercial purposes without their authorisation. This repo is
internal only: keep it private, don't publish the data, and open a licensing
conversation before anything is shown outside the company.

## Season / competition IDs (corrected)

    temporadaId=21     2025/26   <- the season these actes belong to
    temporadaId=22     2026/27   (current)
    disciplinaId=19308233
    competicioId=54322936        Lliga Elit
    grupId=54322937              Grup 1
    club 1150                    Castelldefels, U.E.

Acta IDs run near-sequentially within a season: jornada 1 begins at
3784039 and jornada 30 sits at 3784273, so 2025/26 Lliga Elit Grup 1 spans
roughly 3784039-3784278, eight per jornada plus a few gaps. Postponed or
replayed fixtures get out-of-band ids (3938288, 3938303, 4079169).

## Scraping

`pipeline/scrape.py` sweeps an id range, caches raw HTML, and filters to one
club so we never store more than we need.

    pip install httpx beautifulsoup4 lxml
    python3 pipeline/scrape.py --range 3784039 3784278 --club CASTELLDEFELS

Rate limited to one request every 2.5-4.5s, single threaded, resumable.
Set a real contact address in HEADERS before running.

Before trusting lineup data, calibrate the parser:

    python3 pipeline/scrape.py --ids 3784273 --inspect

Text extraction alone cannot tell a yellow-card minute from a substitution
minute - both render as "65'". Only the icon distinguishes them, so the
parser needs the real class names from --inspect output.
