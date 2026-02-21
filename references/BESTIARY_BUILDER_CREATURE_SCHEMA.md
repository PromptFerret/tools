# Bestiary Builder Creature JSON Schema Reference

A community reference for the Bestiary Builder creature/statblock JSON export format, derived from the [Bestiary Builder source code](https://github.com/Bestiary-Builder/Bestiary-Builder) and verified against the TypeScript type definitions.

This document is maintained by [PromptFerret](https://github.com/promptferret) for use by tool authors who import Bestiary Builder monster data. It is not affiliated with or endorsed by Bestiary Builder.

## Source

- **Repository**: [Bestiary-Builder/Bestiary-Builder](https://github.com/Bestiary-Builder/Bestiary-Builder)
- **Statblock type definition**: `shared/src/build-types.ts`
- **Default statblock / Creature class**: `shared/src/types.ts`
- **Single creature export**: `frontend/src/views/StatblockEditorView.vue`
- **Bestiary export (clipboard + file)**: `frontend/src/views/BestiaryViewer.vue`
- **Avrae integration API**: `backend/logic/avrae.ts`

## User Export Workflow

Bestiary Builder supports clipboard and file export with **no network request required** to generate the JSON:

1. User opens their creature in the statblock editor
2. User clicks the export dropdown
3. Options:
   - **JSON to clipboard** - copies creature JSON via `navigator.clipboard.writeText()`
   - **JSON as file download** - downloads a `.txt` file via browser File API
   - **Homebrewery markdown** - requires API call (not usable offline)
4. User pastes JSON into the importing tool

Single creature exports include an `isBB: true` flag for format detection.

Bestiary export copies/downloads an **array** of statblock objects (without the `isBB` flag).

**Avrae integration** uses server-side API endpoints (`/api/export/bestiary/:id`) and requires network access.

## Creature JSON Structure (Single Export)

```json
{
  "description": {
    "name": "Creature Name",
    "isProperNoun": false,
    "description": "",
    "image": "",
    "faction": "",
    "environment": "",
    "alignment": "Unaligned",
    "cr": 0,
    "xp": 0
  },
  "core": {
    "proficiencyBonus": 2,
    "race": "Humanoid",
    "size": "Medium",
    "speed": [
      { "name": "Walk", "value": 30, "unit": "ft", "comment": "" }
    ],
    "senses": [],
    "languages": []
  },
  "abilities": {
    "stats": {
      "str": 10,
      "dex": 10,
      "con": 10,
      "wis": 10,
      "int": 10,
      "cha": 10
    },
    "saves": {
      "str": { "isProficient": false, "override": null },
      "dex": { "isProficient": false, "override": null },
      "con": { "isProficient": false, "override": null },
      "wis": { "isProficient": false, "override": null },
      "int": { "isProficient": false, "override": null },
      "cha": { "isProficient": false, "override": null }
    },
    "skills": []
  },
  "defenses": {
    "hp": {
      "numOfHitDie": 1,
      "sizeOfHitDie": 6,
      "override": null
    },
    "ac": {
      "ac": 10,
      "acSource": "natural armor"
    },
    "vulnerabilities": [],
    "resistances": [],
    "immunities": [],
    "conditionImmunities": []
  },
  "features": {
    "features": [
      { "name": "Trait Name", "description": "Description text" }
    ],
    "actions": [
      { "name": "Action Name", "description": "Description text" }
    ],
    "bonus": [
      { "name": "Bonus Action Name", "description": "Description text" }
    ],
    "reactions": [
      { "name": "Reaction Name", "description": "Description text" }
    ],
    "legendary": [
      { "name": "LA Name", "description": "Description text" }
    ],
    "lair": [],
    "mythic": [],
    "regional": []
  },
  "spellcasting": {
    "innateSpells": {},
    "casterSpells": {}
  },
  "misc": {
    "legActionsPerRound": 3,
    "telepathy": 0,
    "passivePerceptionOverride": null,
    "featureHeaderTexts": {}
  },
  "isBB": true
}
```

## Bestiary Export

A bestiary export is a **JSON array** of statblock objects (without `isBB` flag):

```json
[
  { "description": { ... }, "core": { ... }, ... },
  { "description": { ... }, "core": { ... }, ... }
]
```

## Key Schema Notes

### Format Detection

- Single creature exports include `"isBB": true` at the top level
- Bestiary exports are a plain JSON array with no wrapper object
- Can detect Bestiary Builder format by checking for `isBB` flag or the `description.cr` + `core.speed[]` structure

### Structured Speed

Unlike CritterDB (which stores speed as a single string), Bestiary Builder uses structured speed objects:

```json
"speed": [
  { "name": "Walk", "value": 30, "unit": "ft", "comment": "" },
  { "name": "Fly", "value": 60, "unit": "ft", "comment": "hover" }
]
```

### Ability Scores

Uses abbreviated keys (`str`, `dex`, `con`, `wis`, `int`, `cha`) instead of full names.

### Saving Throws

Each save has `isProficient` (boolean) and `override` (number or null). When `override` is null, calculate from ability modifier + proficiency bonus (if proficient).

### HP Calculation

- `defenses.hp.numOfHitDie` - number of hit dice
- `defenses.hp.sizeOfHitDie` - die size (6, 8, 10, 12, etc.)
- `defenses.hp.override` - if set, use this value instead of calculating
- Formula: `numOfHitDie * (sizeOfHitDie/2 + 0.5) + numOfHitDie * CON_modifier`

### Actions are Free-Text

Like CritterDB, actions and features are `{name, description}` pairs with free-text descriptions. Attack data (to-hit, damage, range) must be parsed from description text.

### Feature Categories

Bestiary Builder separates features into more categories than most tools:
- `features` - passive traits
- `actions` - standard actions
- `bonus` - bonus actions
- `reactions` - reactions
- `legendary` - legendary actions
- `lair` - lair actions
- `mythic` - mythic actions
- `regional` - regional effects

### Challenge Rating

Stored as a number in `description.cr`. Fractional CRs: `0.125` (1/8), `0.25` (1/4), `0.5` (1/2).

## Import Considerations for EncounterManager

### Direct Mappings
| Bestiary Builder Field | EncounterManager Field |
|---|---|
| `description.name` | `name` |
| `description.cr` | `cr` |
| `core.size` | `size` |
| `defenses.ac.ac` | `ac` |
| `defenses.hp.numOfHitDie` + `defenses.hp.sizeOfHitDie` | `hpFormula` |
| `core.speed[]` | `speed` (join into string) |
| `abilities.stats.*` | `abilities.*` (map abbreviated keys) |
| `abilities.saves.*` | `savingThrows[]` |
| `abilities.skills[]` | `skills[]` |
| `core.senses[]` | `senses` |
| `core.languages[]` | `languages` |
| `features.features[]` | `features[]` |
| `misc.legActionsPerRound` | `legendaryActionBudget` |
| `features.legendary[]` | `legendaryActions[]` |

### Requires Parsing
| Bestiary Builder Field | Challenge |
|---|---|
| `features.actions[]` | Free-text - need regex to extract to-hit, damage dice, damage types |
| `features.bonus[]` | No direct equivalent in EncounterManager - map to features |
| `features.reactions[]` | No direct equivalent - map to features |
| `defenses.ac.acSource` | String - may need mapping |
| `spellcasting` | Complex nested structure - may skip or simplify |

### Advantages Over CritterDB
- Structured speed data (no string parsing needed)
- Abbreviated ability keys (cleaner mapping)
- Separate feature categories (bonus actions, lair actions, mythic actions)
- `isBB` flag for easy format detection
- TypeScript source means schema is well-defined and consistent

### Not Available in Bestiary Builder
- `critRange` (EncounterManager homebrew field)
- Structured attack data (must be parsed from action descriptions)
- Feature recharge/uses (embedded in description text)

## Research Date

February 2026
