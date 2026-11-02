# AI Hotel Pricing Automation

**Live competitor price intelligence and automatic repricing for an independent resort, built with AI-assisted development.**

A small resort sells the same rooms on Booking.com, Agoda and Goibibo and has to
stay competitive with the resorts next door. Doing that by hand means opening
three websites for every competitor, every room and every night, writing the
numbers down, working out a new rate and typing it into the hotel's Rate
Management System (RMS). Prices change several times a day, so by the time it
is done it is already out of date.

This project automates the whole loop:

```
 read competitor prices  →  compare like with like  →  decide a rate  →  write it to the RMS  →  tell the owner
 (Booking.com, Agoda,       (same website, same        (owner's rule,     (the RMS's own UI,       (WhatsApp, email,
  Goibibo, hotel sites)      room tier, same board)     guarded)           read back to verify)     reports, API)
```

It has run in production for a real resort since August 2026. In this public copy, **all
client and competitor names, contacts and data are anonymised**: "Our Resort",
"Competitor A", "Competitor B" and a fictional town called "Hilltown".

---

## What it does

| Capability | How |
|---|---|
| **Competitor price collection** | Adapters for Booking.com, Agoda and Goibibo, plus hotels' own booking engines (Hotelzify, iPMS247, Aiosell and others). Playwright drives a real browser where a site needs one; a JSON endpoint is used where a site offers one. |
| **Like-for-like comparison** | Every price is an *offer*: hotel, website, room, dates, guests, meal plan, refundability. Two prices are compared only when all of these match (`app/services/offer_key.py`). Booking.com is compared only with Booking.com, Agoda only with Agoda. |
| **Room pairing across websites** | The same room has different names on different sites ("Deluxe Double Room" on one, "Deluxe Room" on another, "Classic" in the RMS). Pairs are kept per website, and renamed rooms are detected (`app/services/room_matching.py`, `room_moves.py`). |
| **Rule-based repricing** | The owner's rule, e.g. "₹100 under Competitor A, with tax, on the breakfast plan", is applied per website, with floors, caps and GST slabs taken into account (`app/services/repricing.py`). |
| **Writing to the RMS** | The app logs into the hotel's RMS (RMS Cloud), opens the Room Rate Manager, overrides the rate and reads it back to confirm (`app/services/rate_app_rms.py`). It runs on Preview → Apply, or automatically. |
| **AI second opinion (optional)** | An LLM is asked where the rate should sit tonight, using a strict JSON schema at temperature 0. Its answer passes through the same guards as the rule and is logged in shadow mode first (`app/services/repricing_advisor.py`). |
| **Self-repairing scrapers** | When a website changes its layout, the selectors are re-derived from the live page automatically. A repair is written only if it is verified and actually different (`app/services/rediscovery.py`, `app/workers/tasks_repair.py`). |
| **Alerts** | WhatsApp (Meta Cloud API or a reseller), and email via SMTP or Resend. Includes quiet hours, digests, and a periodic market summary. A move must persist across checks before anyone is alerted. |
| **Reports and API** | Excel export of the comparison grid, a price-feed API that a WhatsApp sales assistant uses to quote tonight's prices, and an "Attention" page listing what needs a human. |
| **Polite by design** | Checks robots.txt before every fetch, rate-limits each site, and identifies itself honestly in the User-Agent. A CAPTCHA or bot wall stops the source, and there is no code path that tries to get around one. |

---

## Architecture

```
                 ┌──────────────┐     ┌────────────────────────┐
  Owner  ───────▶│   FastAPI    │◀───▶│ PostgreSQL             │ prices, history (monthly
  (dashboard)    │ API + Jinja  │     │ (partitioned history)  │ partitions), rules, mappings
                 └──────┬───────┘     └────────────────────────┘
                        │ enqueue                 ▲
                 ┌──────▼───────┐                 │ ingest
                 │    Redis     │                 │
                 └──────┬───────┘                 │
      ┌─────────────────┼──────────────────┬──────┴─────────────┐
┌─────▼─────┐   ┌───────▼───────┐   ┌──────▼──────┐   ┌─────────▼─────────┐
│ Celery    │   │ worker-browser│   │ worker-http │   │ worker-notify     │
│ beat      │   │ Playwright:   │   │ JSON APIs   │   │ WhatsApp / email  │
│ (schedule)│   │ OTAs, RMS     │   │             │   │ digests, summary  │
└───────────┘   └───────────────┘   └─────────────┘   └───────────────────┘
```

- **Python 3.12**, FastAPI, SQLAlchemy 2, Alembic (36 migrations), Celery, Redis, PostgreSQL, Playwright
- `app/adapters/` is the only code that knows about websites. Everything in `app/services/` is pure logic with no database, clock or network, which is why it carries most of the tests.
- **Security:** no secret has a default. Stored credentials, such as the RMS login, are envelope-encrypted, and logs are redacted automatically.

## Project structure

```
app/
  adapters/       one adapter per kind of website (Booking.com, Agoda, Goibibo, hotel engines)
  services/       pure logic: offer identity, comparison, repricing, room matching, rediscovery
  workers/        Celery tasks: fetch, repricing, repair, notify, maintenance, price feed
  notifications/  WhatsApp and email providers, message rendering, digests
  api/v1/         REST API (auth, hotels, sources, prices, repricing, price feed, ...)
  dashboard/      server-rendered pages (comparison, changes, history, repricing, attention)
  db/models/      SQLAlchemy models
alembic/          database migrations
scripts/          probe a site, health check, copy config between deployments, ...
tests/unit/       ~2,000 unit tests (pure logic + recorded page fixtures)
docs/             runbook, deployment guides, engineering notes
```

---

## Run it locally

Requirements: Docker Desktop, or Python 3.12 + PostgreSQL 16 + Redis.

```bash
python scripts/bootstrap_env.py      # writes .env with real random secrets
docker compose up --build
```

| Service | URL |
|---|---|
| Dashboard and API | http://localhost:8000 |
| API docs (OpenAPI) | http://localhost:8000/docs |
| Flower (task monitor) | http://localhost:5555 |
| Mailhog (catches dev email) | http://localhost:8025 |

Create the first account with `python scripts/create_account.py --username owner@123 --admin`,
then add hotels and their booking links on the **Hotels** page.

### Tests

```bash
python -m venv .venv && .venv/Scripts/pip install -r requirements.txt   # Linux/macOS: .venv/bin/pip
# the settings refuse placeholders, so give the test run throwaway values:
export POSTGRES_PASSWORD=test SECRET_KEY=$(python -c "import secrets;print(secrets.token_urlsafe(40))") \
       CREDENTIAL_KEK=$(python -c "from cryptography.fernet import Fernet;print(Fernet.generate_key().decode())")
python -m pytest tests/unit
```

Integration tests need a live database (`TEST_DATABASE_URL`) and are skipped otherwise.

Deployment guides: `docs/RUNBOOK.md`, `docs/DEPLOY-WINDOWS-NATIVE.md` and
`docker-compose.coolify.yml` (Linux VPS with Coolify).

---

## How it was built: AI-assisted development

The project was built by one developer working with **Claude Code** (Anthropic)
as a pair programmer, across many sessions from August to November 2026. The
developer brought the domain knowledge, such as how the resort prices its rooms,
which competitor matters and what the RMS screens look like, and made every
product decision. The AI wrote and refactored the code and tests under that
direction, investigated live pages, and diagnosed failures from logs and
screenshots.

Habits that made this work, and that show in the code:

- **Root causes, not one-off patches.** When one hotel's scraper broke, the fix went into the discovery heuristics so every hotel benefits, never into that hotel's configuration.
- **Every behaviour is pinned by a test named after the real incident**, e.g. `test_a_capped_follow_can_still_be_over_them.py`, `test_a_throttle_that_retried_itself_is_not_on_attention.py`, `test_goibibo_is_read_at_the_price_its_card_shows.py`.
- **Comments record why, with evidence.** Many modules open with the observed failure that motivated them (dates, prices, what went wrong), so the next change does not undo a lesson.
- **Verified against the real thing.** Prices were checked against the live websites and the RMS, not just against unit tests.

See the prompt documentation submitted with this project for how the prompts
were designed and refined.

---

## Screenshots

Screenshots of the dashboard, Repricing page, Attention page and a comparison
report go in `docs/screenshots/`.

---

## A note on the data

This is a sanitised copy of a private production repository. Hotel names,
locations, contact details, server addresses and credentials have been replaced
or removed. Recorded web pages of real hotels have been removed, apart from a
few public booking-engine pages kept as test fixtures. The code itself is
unchanged apart from these renames.

More depth on the design (offer identity, change detection, security) is in
[`docs/ENGINEERING_NOTES.md`](docs/ENGINEERING_NOTES.md).
