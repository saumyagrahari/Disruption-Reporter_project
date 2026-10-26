# Live Disruption Reporter and Route Advisor

Crowd-sourced road closures, crowds and metro shutdowns on a live map, with route suggestions
that avoid confirmed disruption zones.

**Stack:** Python 3.10+, Flask, MySQL (PyMySQL), Leaflet.js + OpenStreetMap tiles.

## How it works

| Piece | What it does |
|---|---|
| **Reports** | Anyone can report *road closed*, *heavy crowd* or *metro closed* at a map point. Identity is an anonymous cookie. |
| **Merging** | A new report within 300 m of an active disruption of the same type joins it instead of creating a duplicate. The disruption centre moves to the running mean of its reports. |
| **Verification** | A disruption starts `pending`. It becomes `confirmed` when the **sum of trust of distinct reporters** reaches `CONFIRM_THRESHOLD` (default 3, so three new users). One person reporting repeatedly counts once. |
| **Trust** | Each reporter starts at 1.0 (range 0.1 to 2.0). Admin *reject* halves the trust of everyone who reported it; admin *confirm* adds 0.25. |
| **Expiry** | Every report refreshes the clock. Road closed/metro closed expire after 6 h, heavy crowd after 2 h. |
| **Routing** | Dijkstra over a road graph (`data/delhi_graph.json`). *Road closed* zones remove the edges they touch. *Heavy crowd* multiplies travel time by 2.5. *Metro closed* is shown as an advisory. Only confirmed disruptions affect routes unless "also avoid unconfirmed" is ticked. |
| **Admin** | `/admin` lists all disruptions and lets a moderator confirm, reject (spam) or resolve them. |

## Setup (MySQL)

```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt

mysql -u root -p < schema.sql                           # creates database disruption_db

export MYSQL_USER=root MYSQL_PASSWORD=yourpassword      # Windows: set instead of export
export ADMIN_KEY=choose-a-long-random-string
flask --app app run --debug
```

Open http://127.0.0.1:5000 for the map and http://127.0.0.1:5000/admin for moderation.

Optional environment variables: `MYSQL_HOST`, `MYSQL_PORT`, `MYSQL_DATABASE`, `CONFIRM_THRESHOLD`.

### No MySQL handy? SQLite demo mode

```bash
export DB_BACKEND=sqlite
flask --app app initdb
flask --app app run --debug
```

## Try it

1. Open the map in three different browsers (or private windows; each is a separate reporter).
2. In each, click the same spot, choose **Road closed**, submit. After the third, it turns solid red (confirmed).
3. Go to **Find route**, pick two places whose normal route crosses that spot, and see the detour (grey dashed = normal route).
4. Open `/admin`, enter your key, and reject or resolve it.

## Tests

```bash
python -m unittest discover -s tests -v
```

The 31 tests cover geometry, merging, trust-weighted confirmation, routing around closures
(including "no route" when cut off), the API flow, rate limiting, expiry and admin auth. They run
against SQLite; see the note below about MySQL.

## Project layout

```
app.py              Flask routes and report lifecycle
logic.py            Haversine, merging, confirmation, Dijkstra (no Flask/DB; unit-tested)
db.py               MySQL/SQLite wrapper
schema.sql          MySQL schema          schema_sqlite.sql   SQLite twin for tests/demo
data/delhi_graph.json   Demo road graph (14 landmarks, 22 links)
templates/, static/     Map UI, admin UI
tests/              Unit and API tests
```

## Known limitations (good talking points, and next steps)

- **The road graph is a demo.** Coordinates are approximate landmarks with simplified links, not real
  street geometry. For real routing, export a drive network from OpenStreetMap (e.g. with OSMnx), write it
  in the same `nodes`/`edges` JSON format, and point `GRAPH_PATH` at it. `logic.py` does not need changes.
- **Cookie identities can be cleared**, so one determined person could pose as several reporters.
  Mitigations to add: per-IP limits, sign-in, or requiring the reporter to be physically near the spot.
- **Concurrency:** two simultaneous first reports at the same spot can create two disruptions. Fix with a
  transaction plus `SELECT ... FOR UPDATE` on candidates, or a grid-cell key with a unique index.
- Report notes are stored but not displayed publicly (kept for moderators/future use).
- Travel times use a flat 25 km/h; there is no live traffic data.
