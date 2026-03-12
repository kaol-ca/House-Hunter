# Scoring System

## Overview

Houses are scored with an additive point system. Each factor contributes +N, 0, or -N to a total. The total determines the score badge color:

| Total | Class | Color |
|-------|-------|-------|
| ≥ 4 | `score-high` | Green |
| 1–3 | `score-mid` | Gold |
| ≤ 0 | `score-low` | Red |

`scoreHouse(h)` returns `{ total: Number, breakdown: Array<{ factor, pts, reason }> }`.

## Fixed-Threshold Factors

These use hard-coded cutoffs. A factor only appears in the breakdown if it contributes non-zero points.

| Factor | -1 Condition | 0 (neutral) | +1 Condition |
|--------|-------------|-------------|-------------|
| **Price** | > $825,000 | $798,000–$825,000 | < $798,000 |
| **Tax** | > $6,000/yr | $5,500–$6,000 | < $5,500/yr |
| **Type** | Townhouse | Other | Detached |
| **Style** | 3-Storey | Other | 2-Storey |
| **Parking** | No garage (0) | — | +N per garage car |
| **Basement** | Unfinished | — | — |
| **Walkout** | — | — | +1 if walk-out basement |
| **Family Room** | — | — | +1 if present |
| **Municipality** | Oshawa (−3), Ajax (−2), Pickering (−2) | Clarington | Whitby (+1) |

**Parking** is unique: it awards the garage count directly as points (e.g. 2-car garage = +2).

## Fixed-Threshold Floor Factors

These apply when room measurements are available (`floorSqft` present, `roomsMissing` false). Style-aware: Second Floor uses `upper` for Backsplit/Sidesplit styles.

| Factor | -1 Condition | +1 Condition |
|--------|-------------|-------------|
| **Main Floor** | < 520 ft² | ≥ 520 ft² |
| **Second Floor** | < 430 ft² | ≥ 430 ft² |

Second Floor treats `upper` (Backsplit/Sidesplit) and `second` (2-Storey/Bungalow) as synonyms — whichever key is present is used, scored against the same threshold.

## Style Configuration

The style system determines which floor levels contribute to scoring, how levels merge for display, and what counts as basement.

### Style Config Table

| Style | mergeMap | scoreLevels | basementLevels | displayOrder |
|-------|----------|-------------|----------------|--------------|
| **2-Storey** (default) | Ground→Main, Upper→Second, Lower→Basement | main, second | basement, lower | Main, Second, Third, Basement |
| **Backsplit 3** | Lower→Main, Ground→Main | main, upper | basement | Main, Upper, Basement |
| **Backsplit 4** | Lower→Upper, Ground→Main | main, upper | basement | Main, Upper, Basement |
| **Sidesplit 4** | Lower→Main, Ground→Main | main, upper | basement | Main, Upper, Basement |
| **Sidesplit 5** | Ground→Main | main, upper, lower | basement | Main, Upper, Lower, Basement |
| **Bungalow** | Ground→Main | main, second | basement, lower | Main, Basement |

### How mergeMap Works

The `mergeMap` determines how raw room levels are grouped for both display and calculation:

- A room on level "Ground" in a 2-Storey home → treated as "Main" floor
- A room on level "Lower" in a Backsplit 3 home → treated as "Main" floor
- A room on level "Lower" in a Backsplit 4 home → treated as "Upper" floor

### How scoreLevels Works

Only levels listed in `scoreLevels` are summed for the "Living Area" rank factor:

- 2-Storey: Main + Second = Living Area
- Backsplit 3: Main (includes merged Lower) + Upper = Living Area
- Sidesplit 5: Main + Upper + Lower = Living Area (Lower is key living space, not basement)

### How basementLevels Works

Levels in `basementLevels` are:
- Excluded from scoring
- Still displayed in the rooms table

### Second Floor Factor Logic

The "Second Floor" rank factor is style-aware:
- If `scoreLevels` includes `'upper'` → uses `floorSqft.upper`
- Otherwise → uses `floorSqft.second`

This ensures Backsplit/Sidesplit homes that use "Upper" are compared fairly against 2-Storey homes that use "Second".

## Excluded Rooms

The following room types are excluded from **all** floor sqft calculations (not just scoring). They still display in the rooms table with an "(excluded)" label:

- Laundry
- Powder Room
- Bathroom
- Sunroom
- Pantry
- Other

## Shared-Area Rooms

Rooms that share physical space are marked `sharedArea: true` and excluded from floor totals. Two detection rules run on parse and on every load (migration):

**Rule 1 — Normalized dimension match**: Rooms on the same level whose dimensions match when normalized (i.e. `min(w,d) × max(w,d)`) are treated as shared. This catches swapped listings (e.g. `18.4 × 11` and `11 × 18.4` are the same space).

**Rule 2 — "Combined w/X" feature**: If a room's features contain `Combined w/Living`, `Combined w/Kitchen`, `Combined w/Dining`, `Combined w/Family`, etc., it is resolved as sharing space with that named room on the same level.

When two rooms are matched, the lower-priority room is marked shared. If priorities are equal, the room carrying the "Combined w/" feature is treated as secondary.

**Priority order**: Kitchen > Living Room > Family Room > Dining Room > everything else.

**Exception**: Bedrooms are never flagged as shared-area by either rule.

## Score Display Locations

| Location | What's Shown |
|----------|-------------|
| Card view | Score badge (colored, +/- prefix) |
| List view | Score badge |
| Modal | Score badge + full breakdown table (factor, points, reason) |
| Parse preview | Score badge + inline factor chips |
| Filter bar | Dropdown to filter by specific factor +/- |
