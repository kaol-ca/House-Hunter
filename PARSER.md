# Parser

The parser extracts structured data from raw text copied from HouseSigma listing pages. Entry point: `parseHouseSigma(text, urlRaw)`.

## Pipeline

### 1. Pre-processing

**Noise removal** — Strip lines matching:
- `*| Image N*` — image captions
- `*-real-estate*` — URL slug lines
- `Image N` — standalone image labels

**Comparables truncation** — Cut `cleaned` text at the first match of:
- `Sold Comparables`
- `Similar for Sale`
- `Rent Comparables`
- `Similar listings`
- `Popularity : N/100`

This prevents data from other listings (in the "Similar Listings" section) from contaminating field extraction. The original text `t` is preserved for room parsing.

### 2. Field Extraction

All regex patterns run against `cleaned` (post-noise, post-truncation).

| Field | Pattern | Notes |
|-------|---------|-------|
| **URL** | User-provided input | Cleaned via `cleanURL()` |
| **Street** | `{N} {Street Name} - {Community}` | Falls back to simpler `{N} {Street}` pattern |
| **Listed Price** | `(?:Listed\|Property listed for)[:\s]*\$\s*([\d,]{6,})` | Allows space after `$` (HouseSigma formats as `$ 859,000`) |
| **Listed Price fallback** | Same keywords with `[^\n$]*\$\s*` lookahead | Catches price further from keyword |
| **Sold Conditional** | `Sold\s+Conditional` | Tested before comparables truncation |
| **Sold Price** | `Sold[:\s]*\$\s*([\d,]{6,})` | Suppressed if `soldConditional` is true |
| **Listed Date** | `Listed\s+on[:\s]*(date pattern)` | Supports `YYYY-MM-DD`, `Month DD, YYYY`, etc. |
| **Tax** | `Tax[:\s]*\$\s*([\d,]{3,})` | Annual amount |
| **Property Type** | `Property\s+Type[:\s]*([^\n]+)` | First value before comma (e.g. "Detached" from "Detached, 2-Storey") |
| **Style** | `Style\s*:\s*([^\n]+)` | Requires colon (prevents matching "style" in descriptions) |
| **Building Age** | `Building\s+Age[:\s]*([^\n]+)` | Raw text (e.g. "16-30") |
| **Size** | `Size[:\s]*(\d+)\s*[-–]\s*(\d+)\s*(feet²\|ft²\|...)` | Range or single value |
| **Lot Size** | `Lot\s+Size[:\s]*([\d.]+)\s*[x×]\s*([\d.]+)` | Width × Depth in feet |
| **Lot fallback** | `Lot\s+Size[:\s]*([\d,]+)\s*(sqft\|...)` | Standalone area value |
| **Garage** | `(?:Attached\|Detached\|Built-in\|Carport)\s+([\d.]+)\s+garages?` | Decimal truncated via `Math.floor()` |
| **Basement** | `Basement[:\s]*([A-Za-z][^\n]{0,40})` | Description text |
| **Fronting On** | `Fronting\s+on[:\s]*([A-Za-z][^\n]{0,40})` | Compass direction |
| **Community** | `Community[:\s]*(value)` | With newline-bleed prevention |
| **Municipality** | `Municipality\s*:\s*(value)` | Requires colon; validated against URL-slug patterns |
| **Bedrooms** | `Bedrooms?\s*:\s*(\d+)` or `(\d+)\s+Bedrooms?` | Colon-form preferred |
| **Bathrooms** | `Bathrooms?\s*:\s*(\d+)` or `(\d+)\s+Bathrooms?` | Same priority |
| **Family Room** | `family\s*room` anywhere in text | Boolean test |

### 3. Room Parsing

`parseRooms(text)` runs against the **original** text (not cleaned), since room data follows a structured format that doesn't need noise removal.

**Room regex**: 
```
([A-Za-z /]+\d?)[\s(（]+([\d.]+)\s*[x×]\s*([\d.]+)\s*ft[）)]\s*Level:\s*(Main|Second|Upper|Lower|Basement|Third|Ground)[^\n]*
```

**Supported levels**: Main, Second, Upper, Lower, Basement, Third, Ground

**Name cleaning**: After extraction, room names are cleaned of feature-word bleed:
- Split on `\n`, take last segment
- Strip leading feature words: `Vinyl Floor`, `Ceramic Floor`, `Hardwood Floor`, `Laminate`, `Broadloom`, `Carpet`, `Window`, `Open Concept`, `Combined w/...`, `In Between`, `Sunken Room`, `Pot Lights`, `Large Window`, `Picture Window`
- Skip rooms with empty names after cleaning

### 4. Post-Processing (OCR Corrections)

#### Room Dimension Correction
Any room dimension > 80 ft is iteratively divided by 10 until realistic:
- `1197.2` → `119.72` → `11.97`
- Flagged as `corrected: true`, displayed with ⚠ indicator

#### Lot Size Metre-to-Feet Conversion
If `lotW × lotD < 500 ft²` and both dimensions < 80:
- Values are treated as metres and multiplied by 3.28084
- e.g. `12 × 30` → `39 × 98` (3,822 ft²)
- Flagged as `lotCorrected: true`, displayed with ⚠ indicator

#### Garage Decimal Truncation
HouseSigma reports "1.5 garages" — `Math.floor()` truncates to integer (1.5 → 1).

### 5. Shared-Area Detection

After room parsing, rooms on the same level with identical `w × d` are grouped. The highest-priority room keeps its area; others are flagged `sharedArea: true`.

**Priority**: Kitchen > Living Room > Family Room > Dining Room > others

**Exclusion**: Rooms matching `/bedroom|primary bedroom/i` are never considered for shared-area detection.

**Rationale**: HouseSigma often lists Kitchen and Breakfast with the same dimensions because they share the same physical space (open-concept). Same for Living Room/Dining Room and Living Room/Family Room combos.

## Known Limitations

1. **No image parsing** — Only text data is extracted
2. **Single listing per paste** — Cannot batch-parse multiple listings
3. **HouseSigma-specific** — Regex patterns are tuned for HouseSigma's text format; other MLS sites would need different patterns
4. **Room data optional** — Some listings don't include room dimensions; these get `roomsMissing: true`
5. **Rooms without dimensions** — Rooms listed without `(W x D ft)` format are skipped (e.g. `LaundryLevel: Second` with no dimensions)
6. **Price format dependency** — Requires `$ N,NNN` format with space between `$` and digits

## Room Inventory

As of the current pool (65 houses, 495 rooms), the most common room types per level:

**Main floor**: Kitchen (53), Living Room (48), Dining Room (42), Breakfast (25), Family Room (23)
**Second floor**: Bedroom 2 (47), Bedroom 3 (47), Primary Bedroom (45), Bedroom 4 (9)
**Basement**: Recreation (20), Office (6), Bedroom 4 (6)

See `room-inventory.csv` for the complete breakdown by style, level, and room name.
