# Data Dictionary

Every field on a house object, its type, source, and constraints.

## House Object Fields

### Identity & Metadata

| Field | Type | Source | Description |
|-------|------|--------|-------------|
| `id` | string | Generated | Unique ID (`Date.now().toString(36)` + random). Preserved on updates. |
| `url` | string | User input | HouseSigma listing URL. Cleaned by `cleanURL()`. |
| `visited` | boolean | User action | Whether the user has marked this house as toured. Default `false`. |

### Address & Location

| Field | Type | Source | Description |
|-------|------|--------|-------------|
| `street` | string | Parsed | Street address (e.g. "54 Allworth Crescent"). Extracted from `{number} {street} - {community} - {municipality}` pattern. |
| `community` | string | Parsed | Neighbourhood name (e.g. "Bowmanville"). From `Community:` field. |
| `municipality` | string | Parsed | City/region (e.g. "Clarington"). From `Municipality:` field. Validated to exclude URL-slug patterns. |
| `frontingOn` | string | Parsed | Compass direction the property faces (e.g. "South", "East"). From `Fronting on:` field. |

**Natural key**: `fullAddress(h)` = `street + ' - ' + community + ' - ' + municipality`. Used for duplicate detection.

### Pricing

| Field | Type | Source | Description |
|-------|------|--------|-------------|
| `price` | number | Parsed | Listed/asking price in dollars. From `Listed for: $ X` or `Property listed for $X`. |
| `soldPrice` | number | Parsed | Sold price in dollars. From `Sold: $ X`. Null if not sold. Suppressed if `soldConditional` is true. |
| `soldConditional` | boolean | Parsed | Whether the listing shows "Sold Conditional" status. Detected before comparables are stripped. |

**Sold percentage**: Displayed as `(soldPrice - price) / price * 100` with +/- sign.

### Listing Details

| Field | Type | Source | Description |
|-------|------|--------|-------------|
| `listedDate` | string | Parsed | ISO date string (YYYY-MM-DD). From `Listed on:` field. |
| `tax` | number | Parsed | Annual property tax in dollars. From `Tax: $X`. |
| `propertyType` | string | Parsed | Property type (e.g. "Detached", "Link", "Semi-Detached"). First value before comma from `Property Type:`. |
| `style` | string | Parsed | Architectural style (e.g. "2-Storey", "Backsplit 4", "Bungalow"). From `Style:` field. Drives level merging and scoring config. |
| `buildingAge` | string | Parsed | Building age range (e.g. "16-30"). From `Building Age:`. |

### Size & Lot

| Field | Type | Source | Description |
|-------|------|--------|-------------|
| `sizeLo` | number | Parsed | Low end of size range in ft² (e.g. 1500). From `Size: 1500-2000 feet²`. |
| `sizeHi` | number | Parsed | High end of size range in ft². Equals `sizeLo` if single value. |
| `lotW` | number | Parsed/Corrected | Lot frontage width in feet. From `Lot Size: W x D feet`. May be auto-converted from metres. |
| `lotD` | number | Parsed/Corrected | Lot depth in feet. Same source and corrections as `lotW`. |
| `lotArea` | number | Computed | `lotW × lotD` in ft². Fallback to standalone `Lot Size: N sqft` if dimensions unavailable. |
| `lotCorrected` | boolean | Computed | `true` if lot dimensions were auto-converted from metres (original area < 500 ft²). |

### Parking & Basement

| Field | Type | Source | Description |
|-------|------|--------|-------------|
| `garage` | number | Parsed | Number of garage spaces, truncated to integer. From `Attached/Detached/Built-in/Carport N garages`. `Math.floor()` applied (1.5 → 1). |
| `parkingDesc` | string | Parsed | Raw parking description (e.g. "Attached 2 garages"). |
| `basement` | string | Parsed | Basement description (e.g. "Finished, Full", "Partially Finished", "Unfinished"). From `Basement:`. |

### Beds & Baths

| Field | Type | Source | Description |
|-------|------|--------|-------------|
| `beds` | number | Parsed | Total bedroom count. From `Bedrooms: N`. |
| `baths` | number | Parsed | Total bathroom count. From `Bathrooms: N`. |
| `hasFamilyRoom` | boolean | Parsed | Whether "Family Room" appears anywhere in the listing text. |

### Rooms

| Field | Type | Source | Description |
|-------|------|--------|-------------|
| `rooms` | Array | Parsed | Array of room objects. See Room Object below. |
| `roomsMissing` | boolean | Computed | `true` if no rooms were parsed (e.g. room data not present in listing). |

### Floor Totals

| Field | Type | Source | Description |
|-------|------|--------|-------------|
| `floorSqft` | object | Computed | Merged floor totals keyed by level (e.g. `{ main: 487, second: 440, basement: 300 }`). Null if `roomsMissing`. Computed by `calcFloorTotals(rooms, style)`. Excludes shared-area rooms and excluded room types. |

## Room Object Fields

Each entry in the `rooms` array:

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Room name (e.g. "Kitchen", "Primary Bedroom", "Recreation"). Cleaned of feature-word bleed. |
| `w` | number | Width in feet. May be auto-corrected (shifted decimal) if > 80. |
| `d` | number | Depth in feet. Same correction as `w`. |
| `level` | string | Raw floor level: `Main`, `Ground`, `Second`, `Upper`, `Third`, `Lower`, `Basement`. |
| `sqft` | number | Computed `w × d`, rounded to 1 decimal. |
| `features` | string | Room features text (e.g. "Hardwood Floor,Large Window,Open Concept"). |
| `corrected` | boolean | `true` if dimensions were auto-corrected from unrealistic values (> 80 ft). |
| `sharedArea` | boolean | `true` if this room shares its area with a higher-priority room on the same level. Excluded from floor totals. |

### Excluded Room Types

The following room names are excluded from floor total calculations but still displayed:

`Laundry`, `Powder Room`, `Bathroom`, `Sunroom`, `Pantry`, `Other`

Defined in the `EXCLUDED_ROOMS` constant.

### Shared-Area Priority

When two rooms on the same level have identical dimensions, the higher-priority room keeps its area. Priority order:

1. Kitchen
2. Living Room
3. Family Room
4. Dining Room
5. (all others — lower priority)

Bedrooms are **excluded** from shared-area detection entirely (same-sized bedrooms are always separate rooms).

## Computed/Derived Values

| Value | Function | Description |
|-------|----------|-------------|
| `fullAddress(h)` | utility | `street - community - municipality` |
| `midSize(h)` | utility | Average of `sizeLo` and `sizeHi`. Fallback to whichever is available. |
| `scoreHouse(h)` | scoring | Returns `{ total, breakdown }`. See [SCORING.md](SCORING.md). |
| `scoreClass(score)` | scoring | CSS class: `score-high` (≥4), `score-mid` (≥1), `score-low` (≤0). |
