# Changelog

All notable changes to House Hunter v2, in reverse chronological order.

## 2026-03-04 (Session 4) — Scoring Overhaul, Filters, Documentation

### Style-Aware Scoring System
- Introduced `getStyleConfig(style)` architecture with per-style `mergeMap`, `scoreLevels`, and `basementLevels`
- Added style configs for: 2-Storey (default), Backsplit 3, Backsplit 4, Sidesplit 4, Sidesplit 5, Bungalow
- `calcFloorTotals(rooms, style)` now style-aware — merges levels per config before summing
- All display locations (cards, modal, preview) use `getDisplayOrder(style)` for dynamic level rendering
- Modal metric box for Second Floor adapts label/value based on style (Second vs Upper)

### Room Calculation Refinements
- Added `EXCLUDED_ROOMS` constant: Laundry, Powder Room, Bathroom, Sunroom, Pantry, Other excluded from all sqft calculations
- Excluded rooms displayed dimmed with "(excluded)" label in modal and preview
- Shared-area detection expanded to all levels (was Main/Ground only)
- Bedrooms explicitly excluded from shared-area detection (same-sized bedrooms are separate rooms)
- Shared-area detection runs on init migration for existing saved data

### Scoring Factor Changes
- Renamed "Total Floors" rank factor to "Living Area"
- Removed "Basement Space" rank factor
- Added "Municipality" fixed-threshold factor: -1 Oshawa/Ajax, 0 Clarington, +1 Whitby
- "Living Area" and "Second Floor" factors now use style-aware level selection
- "Basement Space" scoring now uses style-aware `basementLevels` (removed in later commit)

### Filter System
- Added filter bar with 7 dropdowns + free-text search
- Dropdown filters: Property Type, Style, Community, Municipality, Size range, Sold/For Sale, Scoring Factor
- Scoring Factor filter: lists every factor with +/- direction, runs `scoreHouse()` per house
- Free-text search matches against street, community, municipality, type, style, basement
- Property Type, Style, Community, Municipality dropdowns dynamically populated from data
- Listing count shows "X of Y listings" when filters active
- "Clear filters" button appears when any filter is set
- Empty state adapts message for filtered vs unfiltered views

### Display Changes
- Reverted "2nd" labels back to "Second" (Second Floor throughout)
- Undid "3rd" alias — Third Level now displays as "Third"
- Added sold-over-asking percentage to all sold price displays: cards, modal, preview (e.g. "+7%")
- Floor totals added to modal rooms section (below table) and card view (replaced Living Space metric)
- Card first metric tile now shows per-floor breakdown (Main/Second/Lower/Bsmt) instead of total size
- Score preview section added to parse preview panel (badge + factor chips)
- Fixed score preview referencing `b.label` instead of `b.factor` — was showing "undefined"

### Duplicate Detection
- `confirmSave()` checks `fullAddress()` against existing entries before adding
- On collision: confirm dialog asks to update existing entry
- Preserves existing house's `id` and `visited` status on update

### Documentation
- Created README.md, ARCHITECTURE.md, DATA_DICTIONARY.md, SCORING.md, PARSER.md, CHANGELOG.md

---

## 2026-03-03 (Session 3) — OCR Corrections, Level Support, Parser Fixes

### OCR Data Correction
- Room dimensions > 80 ft auto-corrected by shifting decimal (e.g. 1197.2 → 11.97)
- Lot dimensions auto-converted from metres to feet when area < 500 ft² (e.g. 12×30 → 39×98)
- Corrected values highlighted with ⚠ amber indicator and `.data-corrected` CSS class
- Missing room data flagged with `roomsMissing` flag; floor metrics show "N/A"

### Multi-Level Room Support
- Added `Ground` to room regex (was missing — affected listings like 27 Majestic Street)
- `Ground` level merged with Main in `calcFloorTotals` and display
- Lower and Basement rooms now displayed in both preview and modal (were previously hidden)
- Display order: Main → Second → Third → Lower → Basement
- Floor totals dynamically show only levels with data

### Parser Fixes
- **Style regex**: Changed `Style[:\s]*` to `Style\s*:\s*` — required colon prevents matching "style" in description text
- **Sold price regex**: Added `\s*` after `$` — HouseSigma formats as `$ 902,527` with space
- **Listed price regex**: Same `$ space` fix applied to both primary and fallback patterns
- **Tax regex**: Same `$ space` fix
- **Garage regex**: Changed `\d+` to `[\d.]+` and `garage` to `garages?` — handles "1.5 garages" format; `Math.floor()` truncates decimal
- **Garage types**: Added `Carport` to regex (was only Attached/Detached/Built-in)
- **Sold Conditional false positive**: Added comparables truncation — strips text after "Sold Comparables", "Similar for Sale", etc. before field extraction
- **Room name bleed**: Changed room name regex from `[A-Za-z\s/]` to `[A-Za-z /]` — `\s` was matching newlines, causing features from previous room to bleed into next room's name
- **Room name cleanup**: Post-extraction strip of known feature keywords (Vinyl Floor, Ceramic Floor, etc.)
- **Init migration**: Cleans corrupted room names in existing saved data on every load

### Shared-Area Detection (initial implementation)
- Rooms on same level with identical dimensions flagged as `sharedArea`
- Priority: Kitchen > Living Room > Family Room > Dining Room
- Shared rooms excluded from floor totals, displayed dimmed with "(shared)" label

---

## 2026-03-03 (Session 2) — New Scoring System, Card Redesign, Modal Metrics

### Scoring System (complete replacement)
- Replaced percentage-based relative scoring (0-100) with additive +/- point system
- 12 scoring factors: 8 fixed-threshold + 4 pool-ranked
- `scoreHouse()` returns `{ total, breakdown }` instead of primitive number
- Score badge thresholds: ≥4 green, ≥1 gold, ≤0 red
- All scores show signed values (+5, -2, 0) with +/- prefix
- Modal shows full "Score Breakdown" table (factor, points, reason)

### Card Layout Redesign
- Listed date moved above price (below address, tight under community/municipality)
- "Main floor ft²" metric replaced with monthly tax (`$N tax/mo`)
- "Lot Area" metric replaced with dims + sqft combo (e.g. "29×99" / "2,871 ft²")

### Modal Metrics Reorder
- Tax shows annual value only (removed monthly breakdown)
- Added 2nd Floor metric (was missing)
- Tile order: Tax, Lot Area, Living Space, Main Floor, 2nd Floor, Bed, Bath, Gar
- Fixed `.upper` vs `.second` key mismatch in floor data

---

## 2026-03-03 (Session 1) — Initial Parser Build

### Parser
- Built HouseSigma paste-to-parse workflow
- Field extraction for: street, community, municipality, price, sold price, date, tax, property type, style, size, lot, parking, basement, beds, baths, fronting, family room
- Room parsing with dimensions and levels (Main, Second, Upper)
- Card and list views with sort options
- localStorage persistence with export-as-HTML
- Relative scoring system (later replaced)
