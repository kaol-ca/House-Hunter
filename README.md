# House Hunter v2

A single-file HTML real estate listing tracker with a paste-to-parse workflow for [HouseSigma](https://housesigma.com) listings. Designed for comparing homes in the Durham Region (Ontario, Canada) housing market.

## What It Does

- **Paste & Parse**: Copy text from a HouseSigma listing page, paste it in, and the app extracts 20+ structured fields automatically
- **Score & Compare**: An additive scoring system rates each house on 12 factors (price, tax, location, lot size, living area, etc.)
- **Browse & Filter**: Card grid and list views with search, dropdown filters, and sort options
- **Track**: Mark homes as visited, add notes, track sold vs. for-sale status
- **Export**: Download the full app with embedded data as a portable HTML file

## Quick Start

1. Open `house-hunter-v2.html` in any modern browser
2. Click **+ Add** in the header
3. Paste the full text from a HouseSigma listing page (Ctrl+A → Ctrl+C on the listing)
4. Optionally paste the listing URL
5. Click **Parse** to preview extracted data and score
6. Click **Confirm & Save** to add to your pool

No server, no install, no dependencies. Everything runs client-side with localStorage.

## Project Structure

```
house-hunter-v2.html    — The entire application (HTML + CSS + JS)
README.md               — This file
ARCHITECTURE.md         — Technical design and component relationships
DATA_DICTIONARY.md      — Every field: type, source, constraints
SCORING.md              — Scoring system rules and factor definitions
PARSER.md               — Parser regex patterns, OCR corrections, edge cases
CHANGELOG.md            — Chronological history of all changes
```

## Key Concepts

- **Pool**: The set of all saved houses. Pool-ranked scoring factors compare a house against the pool.
- **Style Config**: Houses of different architectural styles (2-Storey, Backsplit, Sidesplit, Bungalow) have different floor-level merging and scoring rules. See [SCORING.md](SCORING.md).
- **Seed Data**: Houses are embedded as `SEED_HOUSES` in the HTML file. On export, current data replaces the seed. On first load without localStorage, seed data is used.
- **Duplicate Detection**: Uses `fullAddress()` (street + community + municipality) as a natural key to prevent duplicate entries.

## Tech Stack

- Vanilla HTML/CSS/JS — no frameworks, no build step
- Google Fonts (DM Sans + Fraunces)
- localStorage for persistence
- Self-contained export via Blob/download

## Browser Support

Any modern browser (Chrome, Firefox, Safari, Edge). No IE support.
