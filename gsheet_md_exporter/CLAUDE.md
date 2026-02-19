# GSheet D&D Character Exporter

## What This Is
A single self-contained HTML file (`index.html`) that exports D&D 5e character sheet data from a public Google Sheet into Markdown or JSON, suitable for pasting into LLMs.

## How It Works

### Data Fetching
- Uses the **Google Sheets direct CSV export** endpoint via `fetch()`
- URL format: `https://docs.google.com/spreadsheets/d/{SHEET_ID}/export?format=csv&gid={GID}`
- The sheet MUST be public (Share -> Anyone with the link -> Viewer)
- **Requires HTTP origin** — must be served from a web server, not opened as `file://` (CORS restriction)

### Multi-Tab Architecture
The tool fetches data from up to **4 tabs** per spreadsheet:

| Tab | Purpose | Discovery |
|-----|---------|-----------|
| **First tab** (main) | Main character sheet — stats, abilities, skills, spells, etc. | Always uses the first tab |
| **Additional** | Extended attacks, extra cantrips/spells, multiclass info, resistances/immunities/vulnerabilities | Found by name in discovered tabs |
| **Inventory** | Currency, wealth/weight, full item list | Found by name in discovered tabs |
| **Spell Preparation** | Source of truth for spells — replaces main/Additional spell data when present | Found by name in discovered tabs |

**Tabs NOT processed** (backend/reference data used by sheet formulas):
- `?`, `Attack Info`, `Gear Info`, `Race Info`, `Class Info`

**Tab discovery** uses the `/htmlview` endpoint which contains a JS `items.push({name, gid})` array. This is ~36KB but is the lightest endpoint that returns tab metadata. The tool always uses the first discovered tab as main; if tab discovery fails, it falls back to the gid from the user's URL.

Additional, Inventory, Spell Preparation, and any unknown/custom tabs are all fetched **in parallel** after the main sheet.

**Custom tab auto-detection**: Unknown tabs (not matching any known name or ignored list) are fetched and inspected via `detectTabType()`. Row 1 headers are checked:
- `#` at col 8, `ITEM` at col 9, `COST` at col 17, `WEIGHT` at col 20 → detected as **inventory** type
- `NAME` at col 1, `ATK BONUS` at col 8, `DAMAGE/TYPE` at col 12 → detected as **additional** type

Auto-detected inventory tabs are extracted as `extraInventories` and rendered as additional inventory sections in the markdown output. Auto-detected additional tabs are processed for attacks, spells, etc.

### Why CSV Export (not gviz)
The tool originally used the Google Visualization API (gviz) with JSONP, but we switched to direct CSV export because:
- **gviz has a critical bug**: an empty cell gap in col 2 combined with data in col 28 corrupts the entire response
- **gviz can't read merged cells**: Speed, Inspiration, Weight, Hair, odd-level spell slots, and skill proficiency markers (◉) all returned null
- **CSV export reads everything**: all merged cells, all proficiency markers, all spell slots — no limitations
- **Trade-off**: CSV export requires HTTP origin (can't run from `file://`), which is acceptable since the user has a server

### Data Format
- CSV is parsed with a custom RFC 4180 parser (`parseCSV()`) that handles quoted fields, commas within quotes, and escaped double-quotes
- The parser produces a simple 2D string array: `rows[row][col]`
- Row 0 = CSV row 1, Col 0 = column A
- **CSV row indices differ from gviz indices** — varying offsets (+1 to +16 depending on section)

### Parsing Strategy
- **Position-map based**: each template version has a hardcoded map of [row, col] positions for every field
- **Template detection**: check cell `[1,19]` for template identifier (e.g. "Gsheet 4.0")
- Falls back to v4.0 layout if no template detected — may produce wrong data for incompatible sheets
- No searching or scanning — every field is read directly from its known position
- To add a new template, create a new position map object

### Output Formats
- **Preview** (default): renders markdown as styled HTML for visual review
- **Markdown**: raw markdown code for pasting into LLMs
- **JSON**: structured object with all the same fields
- Toggle between Preview/MD/JSON with toolbar buttons
- "Copy" button always copies the raw markdown (or JSON when in JSON mode), even from Preview
- **Download buttons** ("⬇ MD" / "⬇ JSON"): save output as local files, disabled until a sheet is processed
  - Filenames derived from character name, sanitized (special chars stripped, spaces → underscores)
  - Uses Blob + object URL for client-side download (no server needed)
- Enter key in the URL input triggers Fetch

### Sheet Tabs UI
After tab discovery, a pill-style bar appears below the status bar showing all discovered tabs:
- **Active** tabs (highlighted with accent border): tabs that were fetched and used in the output
- **Ignored** tabs (dimmed): known backend/reference tabs (`?`, `Attack Info`, etc.)
- **Other** tabs: discovered but not matching any known or auto-detected structure
- Tooltips describe each tab's role (e.g., "Extended attacks, spells, multiclass, defenses")
- Auto-detected custom tabs show their detected type in the tooltip (e.g., "Inventory container (auto-detected)")

## v4.0 Layout — Complete Position Map (CSV)

All positions verified against marker sheet CSV export AND real test sheet CSV export.
Row/Col are 0-indexed from the raw CSV output.

### Basic Info
| Field | Row | Col | Notes |
|-------|-----|-----|-------|
| Template Marker | 1 | 19 | Contains "Gsheet 4.0" |
| Character Name | 5 | 2 | |
| Class/Level | 4 | 19 | |
| Race | 6 | 19 | |
| Player Name | 4 | 30 | |
| Background | 10 | 35 | |
| Alignment | 27 | 35 | |

### Combat Stats
| Field | Row | Col | Notes |
|-------|-----|-----|-------|
| AC | 11 | 17 | |
| Initiative | 11 | 21 | |
| Speed | 11 | 25 | |
| Inspiration | 10 | 7 | |
| HP Max | 15 | 20 | |
| Current HP | 16 | 17 | |
| Temp HP | 20 | 17 | |
| Condition | 19 | 20 | |
| Hit Dice | 24 | 17 | |
| Proficiency Bonus | 13 | 7 | |
| Passive Perception | 44 | 2 | |

### Ability Scores
Each ability: score at col 2 (scoreRow), modifier at col 2 (modRow).

| Ability | Score Row | Modifier Row |
|---------|-----------|-------------|
| STR | 14 | 12 |
| DEX | 19 | 17 |
| CON | 24 | 22 |
| INT | 29 | 27 |
| WIS | 34 | 32 |
| CHA | 39 | 37 |

### Saving Throws (rows 16-21)
| Col | Content |
|-----|---------|
| 7 | Proficiency marker: ◉ = proficient, 〇 = not (readable in CSV) |
| 8 | Save modifier (number) |
| 9 | Ability name (Strength, Dexterity, etc.) |

### Skills (rows 24-41)
| Col | Content |
|-----|---------|
| 7 | Proficiency marker: ◉ = proficient/expert, 〇 = not (readable in CSV) |
| 8 | Skill modifier (number) |
| 9 | Skill name |

Skill order (rows 24-41): Acrobatics, Animal Handling, Arcana, Athletics, Deception, History, Insight, Intimidation, Investigation, Medicine, Nature, Perception, Performance, Persuasion, Religion, Sleight of Hand, Stealth, Survival

Proficiency/expertise detection (◉ marker + math for expertise vs proficient):
- Marker is ◉ AND `skillMod >= abilityMod + profBonus * 2` -> Expertise
- Marker is ◉ -> Proficient
- Marker is 〇 -> Not proficient

### Attacks
| Rows | Col 17 | Col 24 | Col 28 |
|------|--------|--------|--------|
| 31-35 | Attack Name | Attack Bonus | Damage/Type |
| 36-41 | Additional Attacks (text only) | — | — |

### Languages, Equipment, Proficiencies
| Section | Col | Rows | Notes |
|---------|-----|------|-------|
| Languages | 17 | 44-55 | Filter out label "LANGUAGES" |
| Equipment | 28 | 44-55 | Filter out label "EQUIPPED ITEMS"; `*` prefix = attunement |
| Proficiencies | label col 2, value col 8 | 48-54 | Filter out label "PROFICIENCIES" |

### Personality (col 30)
| Section | Value Rows | Label Row |
|---------|-----------|-----------|
| Personality Traits | 11, 12, 13 | 14 |
| Ideals | 15, 16, 17 | 18 |
| Bonds | 19, 20, 21 | 22 |
| Flaws | 23, 24, 25 | 26 |

Up to 3 values per section, comma-separated.

### Features & Traits
- Cols: 2, 15, 28
- Rows: 58-83 (26 rows x 3 cols = 78 slots)
- Read column by column (left, middle, right) to preserve grouping
- Filter out label "FEATURES & TRAITS"

### Spellcasting Header
| Field | Row | Col |
|-------|-----|-----|
| Spellcasting Class | 90 | 2 |
| Spellcasting Ability | 90 | 20 |
| Spell Save DC | 90 | 27 |
| Spell Attack Bonus | 90 | 34 |

### Cantrips
- Rows: 95-97
- Columns: name at 13/23/33
- 3 rows x 3 cols = 9 cantrip slots

### Leveled Spells
Odd levels use cols [2,3], [12,13], [22,23] (marker, name pairs).
Even levels use cols [12,13], [22,23], [32,33] (marker, name pairs).
Marker col has ◉ (prepared) or 〇 (known but not prepared).
All spells with names are included regardless of marker. Marker determines prepared status.
Output format is a two-column table per level: `| Prepared | Spell |` with ◉/〇 markers.
This distinguishes e.g. Warlock spells (◉) from item-granted spells like Ring of Shooting Stars (〇).

| Level | Start Row | Rows | Max Slots Position | Current Slots Position |
|-------|-----------|------|--------------------|-----------------------|
| 1 | 99 | 5 | [100, 36] | [100, 32] |
| 2 | 105 | 5 | [106, 4] | [106, 8] |
| 3 | 111 | 5 | [112, 36] | [112, 32] |
| 4 | 117 | 4 | [118, 4] | [118, 8] |
| 5 | 122 | 4 | [123, 36] | [123, 32] |
| 6 | 127 | 4 | [128, 4] | [128, 8] |
| 7 | 132 | 3 | [133, 36] | [133, 32] |
| 8 | 136 | 3 | [137, 4] | [137, 8] |
| 9 | 140 | 3 | [141, 36] | [141, 32] |

### Physical Description
| Field | Row | Col |
|-------|-----|-----|
| Age | 147 | 2 |
| Height | 147 | 5 |
| Weight | 147 | 8 |
| Size | 147 | 11 |
| Gender | 149 | 2 |
| Eyes | 149 | 5 |
| Hair | 149 | 8 |
| Skin | 149 | 11 |

### Other Sections
| Field | Row(s) | Col |
|-------|--------|-----|
| Allies & Organizations | 146-161 | 17 |
| Character Backstory | 164-176 | 17 |
| Character Image | 154 | 2 |
| Appearance Image URL | 175 | 2 |
| Symbol Image URL | 175 | 8 |

## v1.4 Layout — Position Map Differences

The v1.4 template (public-domain "D&D 5e Gsheet v1.4") shares most positions with v4.0. Only the differences are listed here — all other fields use the same positions as v4.0.

### Template Detection
| Field | Row | Col | Notes |
|-------|-----|-----|-------|
| Template Marker | 3 | 42 | Contains "1.4" |

### Fields That Differ from v4.0
| Field | v4.0 Position | v1.4 Position |
|-------|---------------|---------------|
| Background | [10, 35] | [4, 25] |
| Alignment | [27, 35] | [6, 25] |
| Hit Dice | [24, 17] | [25, 17] |
| Total Level | Additional tab only | [5, 37] (main sheet) |

### Equipment (v1.4)
- Col: 34, Rows: 31-41
- Label to exclude: `WEAPONS & ARMOR` (v4.0 uses `EQUIPPED ITEMS` at col 28, rows 44-55)

### Features & Traits (v1.4)
- Cols: 25, 33
- Rows: 44-55 (12 rows x 2 cols = 24 slots)
- v4.0 has 78 slots (rows 58-83, cols 2/15/28)

### Proficiencies (v1.4)
- Single column at col 2, rows 48-54
- Inline text format: e.g. `"Proficient in Armor: Light, Medium"`
- Parsed by splitting on first colon → label/value
- v4.0 uses separate label (col 2) and value (col 8) columns

### Main Sheet Inventory (v1.4 only)
v1.4 has inventory on the main sheet instead of a separate Inventory tab.

#### Currency
| Currency | Row | Col |
|----------|-----|-----|
| Copper | 59 | 3 |
| Silver | 62 | 3 |
| Electrum | 65 | 3 |
| Gold | 68 | 3 |
| Platinum | 71 | 3 |

#### Inventory Summary
| Field | Row | Col |
|-------|-----|-----|
| Coin Value | 75 | 4 |
| Wealth | 76 | 3 |
| Max Weight | 80 | 4 |
| Current Weight | 81 | 3 |

#### Items
- Rows: 59-83 (two side-by-side item lists, same column structure as Inventory tab)
- Left: qty=8, name=9, cost=17, weight=20
- Right: qty=25, name=26, cost=34, weight=37

### Additional Tab (v1.4)
The v1.4 Additional tab is nearly identical to v4.0's. Differences:
- **Immunities** at col 30 (v4.0: col 27)
- **No vulnerabilities** column (v4.0: col 34)
- All other positions (attacks, cantrips, spells, classes, resistances) are the same

### Tab Expectations (v1.4)
- Has an **Additional** tab (with layout differences noted above)
- **No separate Inventory tab** — inventory is on the main sheet (set to null in template)
- If custom tabs matching Inventory structure are found, auto-detection still handles them

## Spell Preparation Tab

The Spell Preparation tab is the **source of truth** for spells when present. It contains richer metadata (spell source attribution, class assignment, "Always prepared" flags) that the main sheet flattens away. Spell slot info (maxSlots, curSlots) always comes from the main sheet.

### Detection & Fallback Logic
1. Spell Prep tab exists + has spell data → **use it**, replace main/Additional spell data
2. Spell Prep tab exists but empty + main/Additional have spells → **fallback** to main/Additional
3. No Spell Prep tab → use main/Additional as-is

### Two Layouts

#### 5-col layout (customized for large spell lists)
- **Detect**: Row 1, col 9 = `"Spell Preparation"` (CSV 0-indexed)
- **Level start columns**: 1 (cantrips), 6, 11, 16, 21, 26, 31, 36, 41, 46 (stride +5)
- **Scan rows**: 14+ until no more data
- **Per row**: `col+0`=TRUE/FALSE, `col+1`=name, `col+2`=source, `col+3`=preparedAs
- Skip rows where name=`"Extra"` (section dividers) or name is empty
- **Cantrips**: only include rows where `col+0` = TRUE (selected)
- **PreparedAs**: class name, `"Always prepared"`, or empty

#### 14-col layout (default template, simpler characters)
- **Detect**: Row 2, col 16 = `"Spell Preparation"` (CSV 0-indexed)
- **Level start columns**: 2 (cantrips), 16, 30, 44, 58, 72, 86, 100, 114, 128 (stride +14)
- **Scan rows**: 18 to 149
- **Per row**: `col+0`=TRUE/FALSE (only process rows with these values), `col+1`=name
- **Source**: try `col+10` first (Extra section), then `col+7` (Natural section)
- **Cantrips**: only include rows where `col+0` = TRUE (selected)
- No always-prepared tracking in this layout

## Output Format Spec

### Markdown Structure
```
# Character Name

| **Race** | value |
|---|---|
| **Class** | value |
| **Background** | value |
| **Alignment** | value |
| **Player** | value |

| **Personality Traits** | value |
|---|---|
| **Ideals** | value |
| **Bonds** | value |
| **Flaws** | value |
| **Age** | value |
... (physical description fields merged into this table)

## Combat Stats
| AC | HP | Initiative | Speed | Prof Bonus | Hit Dice |
|:--:|:--:|:----------:|:-----:|:----------:|:--------:|
| values... |

**Current HP:** ... | **Temp HP:** ... | **Condition:** ...
**Passive Perception:** ... | **Inspiration:** ...

## Ability Scores (table)
## Saving Throws (table with Proficient column)
## Skills (table with Proficient / Expert columns)
## Proficiencies (table with Type|Details header)
## Attacks (table)
## Features & Traits (bullet list)
## Equipment (bullet list, attunement items marked)
## Languages (comma-separated)
## Spellcasting
When spell prep data is used (d.spellSource === 'spell_prep'):
  - Cantrips: `| Spell | Source |` two-column table
  - Leveled spells: `| Prepared | Spell | Source |` three-column table
  - Prepared column: `◉` (prepared), `Always` (always prepared), `〇` (not prepared)
  - Sort: Always first, then ◉, then 〇
When no spell prep (main/Additional fallback):
  - Cantrips: single-column `| Spell |` table
  - Leveled spells: `| Prepared | Spell |` two-column table
## Allies & Organizations (bullet list)
## Backstory (text)
```

### Markdown Table Rules
- Tables with column headers (Combat Stats, Skills, Attacks, Spells, Proficiencies) have a proper header row
- Tables used as key-value pairs (Basic Info, Personality/Physical) use `**bold**` labels in the first column — no column header row
- The separator row (`|---|---|`) goes after the first row
- Empty headers (`| | |`) break markdown rendering — never use them

### Preview Rendering
- Built-in markdown-to-HTML renderer (`mdToHTML()`) handles headings, tables, lists, bold/italic
- Header detection heuristic: first table row is `<th>` only if cells don't contain `**bold**` markup
- This correctly renders key-value tables (bold labels = data) vs column-header tables (plain labels = headers)

## Development Philosophy

### Prefer Local Execution Over Agent Processing
When data needs to be fetched, parsed, transformed, or verified, **write and run a local script** (Python or Node) on the user's machine rather than doing it manually in conversation. A local script executes instantly at zero token cost; the same work done step-by-step by the agent consumes thousands of tokens for an identical result.

Examples of tasks that should always run locally:
- Downloading and inspecting CSVs or API responses
- Testing Google Sheets endpoints (gviz queries, CSV export URLs, etc.)
- Parsing spreadsheet data to discover structure or verify cell positions
- Comparing outputs or diffing results

**Never use WebFetch or agent-side processing to test an API endpoint.** Write a local script, run it, and read the output.

## Development Notes

### DO NOT
- Filter out MARKER text from output — markers are temporary test data the user needs to see for verification
- Use landmark/search-based field finding — use the position map directly
- Spawn expensive subagents for debugging — use simple direct approaches
- Assume cells are empty because they're merged — CSV export reads all merged cells

### DO
- ASK the user questions instead of burning tokens guessing
- Use the marked reference sheet to verify cell positions
- Test with the user's actual sheet URLs
- Keep everything in a single `index.html` file (no external dependencies)
- Use the position map for all field extraction
- Remember: CSV positions differ from gviz positions (see history below)

### Potential Base Ability Scores
The row below each ability score (e.g., STR score at [14,2], potential base at [15,2]) may contain the
base/unmodified score. On Nyx's sheet, CON score is 19 (Amulet of Health) but the row below shows 11.
All other abilities match their current scores (no items modifying them). Not yet extracted — needs
further investigation.

### Character Name
CSV export reads the character name directly from the sheet (no longer needs a manual input field).
The name input workaround was removed after switching from gviz to CSV.

### Historical Note: gviz vs CSV Position Offsets
When we switched from gviz to CSV export, all row indices changed. The offsets are NOT uniform — they vary by section (+1 to +16). The CSV position map was built by downloading the marker sheet as CSV and parsing it with Python. Never mix gviz positions with CSV positions.
