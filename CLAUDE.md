# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A single-file static web app (`index.html`) that displays a live airport departure/arrivals board for Boston Logan (KBOS). It is designed for a specific apartment viewpoint and only shows flights visible from that location.

## Architecture

Everything lives in one `index.html` file — no build step, no dependencies, no bundler. Open the file directly in a browser or serve it as a static file.

The JavaScript is organized into these logical sections (all global scope, inline `<script>`):

- **VIEWER config** — hardcoded apartment coordinates (`42.3662°N, 71.0621°W`) and visible arc (`VIS_MIN=10°` to `VIS_MAX=160°`). The `bearingToFlight()` and `isVisible()` functions filter flights to only those in the NE-facing window.
- **Lookup tables** — `CITY` (IATA→city name), `AIRLINE` (ICAO callsign prefix→name/country), `FLEET` (airline→weighted aircraft pool). Aircraft type is deterministically assigned via `acft()` using the hex ICAO code as a seed.
- **Data pipeline** — `fetchFlights()` calls `https://api.adsb.lol/v2/lat/{lat}/lon/{lon}/dist/50` every 30 seconds. On failure, falls back to `demoData()`. Raw ADS-B states are parsed by `parseState()`.
- **Classification** — `classify()` puts each flight in one of: `ground`, `approach`, `climb`, `cruise`, `enroute`. `isDep()` / `isArr()` determine which board a flight appears on. `isImmDep()` / `isImmArr()` detect flights within ~3 km of the airport at low altitude.
- **Route cache** — `routeCache` maps callsign → `{destCity, destIata, origCity, origIata}`. Populated by the ADSB.LOL `/route` endpoint (called per-flight) and by `demoData()`.
- **Render** — `renderBoards()` filters, sorts by distance from airport, and calls `renderList()` which generates HTML strings injected via `innerHTML` (up to 20 rows per board).
- **Alert queue** — `checkAlerts()` builds an ordered queue of imminent flights (arrivals first, then departures, sorted by proximity). Each alert shows for 10s then auto-dismisses. The `dismissed` Set prevents re-showing the same callsign in a session.
- **Loading animation** — Canvas-based plane trail animation runs during the 2.2s startup delay, then stops via a MutationObserver.

## Key Constants

| Constant | Value | Purpose |
|---|---|---|
| `VIEWER` | `42.3662°N, 71.0621°W` | Apartment location for bearing calculations |
| `VIS_MIN / VIS_MAX` | `10° / 160°` | Visible arc from apartment window |
| `OVERHEAD_BEARING` | `85°` | Airport bearing from apartment (flights near this show ⚠) |
| `RANGE` | `0.8°` | Declared radar range (display only) |
| `REFRESH` | `30` (seconds) | API polling interval |
| `NM` | `50` (nautical miles) | ADSB.LOL search radius |

## Data Source

`https://api.adsb.lol/v2/` — free, no API key required. The app uses:
- `/lat/{lat}/lon/{lon}/dist/{nm}` — nearby aircraft
- `/callsign/{cs}` — route lookup per flight (cached in `routeCache`)
