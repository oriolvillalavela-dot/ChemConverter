# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Chem Converter** — A FastAPI service that resolves and converts between chemical identifiers: IUPAC names, SMILES, CAS Registry Numbers, and InChI strings with stereochemistry support. It aggregates data from the CAS API (OAuth2-authenticated) and CIRpy (Chemical Identifier Resolver).

## Commands

```bash
# Install dependencies
pip install -r requirements.txt

# Run the application (starts at http://0.0.0.0:8000)
python app.py

# Run rate limiter tests
python test_rate_limit.py
```

The API docs are available at `http://localhost:8000/docs`. The web UI is served from `static/` at `http://localhost:8000/`.

## Configuration

The app is configured via environment variables (`.env` file, not committed):

| Variable | Purpose |
|---|---|
| `CAS_SERVER` | Base URL for CAS API |
| `CAS_TOKEN_URL` | OAuth2 token endpoint |
| `CAS_CLIENT_ID` | OAuth2 client ID |
| `CAS_CLIENT_SECRET` | OAuth2 client secret |
| `CAS_SCOPE` | OAuth2 scope |
| `CAS_VERIFY_SSL` | SSL verification (use `certs/corp-root.pem` on corporate network) |
| `CAS_DEBUG` | Enables debug logging to `.cas_debug/` directory |
| `PORT` | Server port (default 8000) |

## Architecture

**Data flow:** `app.py:/resolve` → `converters.py` (IUPAC→SMILES) → `cas_client.py` (CAS API lookup/enrichment) → RDKit (fill missing fields) → JSON response

### Backend Modules

- **`app.py`** — FastAPI app with two endpoints: `/health` and `/resolve`. The `/resolve` endpoint accepts `inputType` (iupac/smiles/cas), `value`, and `fullConversion` (bool).

- **`cas_client.py`** — The most complex module (485 lines). Contains:
  - `CASAuth` — OAuth2 token caching with expiry checking
  - `_RateLimiter` — Thread-safe rate limiter using `threading.Condition`, enforces minimum interval between requests
  - `RequestHandler` — HTTP request handling with token refresh and debug logging (masks Bearer tokens)
  - Lookup methods: `lookup_by_smiles()` uses InChI-first matching for stereochemical accuracy; `lookup_by_cas()` and `lookup_by_name()` for other input types

- **`converters.py`** — `iupac_to_kekule_smiles()`: CIRpy → StdInChI → RDKit isomeric SMILES pipeline. Preserves stereochemistry (@/@@ notation) by going through InChI rather than direct SMILES conversion. Falls back to CIRpy SMILES if InChI is unavailable.

### Frontend

Vanilla JS/HTML/CSS in `static/` — no build step. The UI handles batch processing with concurrent API calls, real-time progress tracking, and per-row status badges. Uses relative paths for Posit Connect compatibility.

## Deployment

Configured for **Posit Connect** (formerly RStudio Connect) via `manifest.json`. Requires Python 3.10. Dev environment uses a Roche `ona-base:2.0` Docker image (see `.devcontainer/`).
