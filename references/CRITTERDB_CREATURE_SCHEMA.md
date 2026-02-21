# CritterDB Creature JSON Schema Reference

A community reference for the CritterDB creature/bestiary JSON export format, derived from the [CritterDB source code](https://github.com/haswellr/CritterDB) and verified against the Mongoose model definitions.

This document is maintained by [PromptFerret](https://github.com/promptferret) for use by tool authors who import CritterDB monster data. It is not affiliated with or endorsed by CritterDB.

## Source

- **Repository**: [haswellr/CritterDB](https://github.com/haswellr/CritterDB)
- **Creature model**: `server/models/creature.js`
- **Data adapter / export logic**: `server/public/js/services/dataAdapter.js`
- **Export controller**: `server/public/js/controllers/exportCreatureJson.js`
- **Foundry VTT importer (reference)**: [jwinget/fvtt-module-critterdb-import](https://github.com/jwinget/fvtt-module-critterdb-import)

## User Export Workflow

CritterDB supports clipboard and file export with **no network request required** to generate the JSON:

1. User opens their creature on CritterDB
2. User clicks export, selects format ("CritterDB JSON", "5E Tools", or "Improved Initiative")
3. User clicks **"Copy to Clipboard"** or **"Download File"**
4. JSON is generated client-side from the in-memory data - no server roundtrip
5. User pastes JSON into the importing tool

The "CritterDB JSON" format applies zero data mappers - it is a direct `JSON.stringify()` of the internal creature object. Clipboard copy uses the Clipboard.js library.

**Sharing via URL** (e.g., `https://critterdb.com/#/creature/view/{id}`) **does require fetching from the CritterDB API** and is not usable offline.

## Creature JSON Structure

A single creature export (native CritterDB JSON format):

```json
{
  "_id": "string",
  "name": "Creature Name",
  "bestiaryId": "string",
  "flavor": {
    "faction": "",
    "environment": "",
    "description": "",
    "nameIsProper": false,
    "imageUrl": ""
  },
  "stats": {
    "size": "Medium",
    "race": "",
    "alignment": "",
    "armorType": "",
    "armorClass": 10,
    "hitDieSize": 8,
    "numHitDie": 1,
    "speed": "",
    "abilityScores": {
      "strength": 10,
      "dexterity": 10,
      "constitution": 10,
      "intelligence": 10,
      "wisdom": 10,
      "charisma": 10
    },
    "proficiencyBonus": 0,
    "savingThrows": [
      { "ability": "strength", "value": null, "proficient": true }
    ],
    "skills": [
      { "name": "Perception", "value": null, "proficient": true }
    ],
    "damageVulnerabilities": [],
    "damageResistances": [],
    "damageImmunities": [],
    "conditionImmunities": [],
    "senses": [],
    "languages": [],
    "challengeRating": 0,
    "experiencePoints": 0,
    "additionalAbilities": [
      { "name": "Trait Name", "description": "Free-text description" }
    ],
    "actions": [
      { "name": "Action Name", "description": "Free-text description" }
    ],
    "reactions": [
      { "name": "Reaction Name", "description": "Free-text description" }
    ],
    "legendaryActionsPerRound": 3,
    "legendaryActionsDescription": "",
    "legendaryActions": [
      { "name": "LA Name", "description": "Free-text description" }
    ]
  }
}
```

## Bestiary Export

A bestiary export wraps creatures in a bestiary object:

```json
{
  "_id": "string",
  "name": "Bestiary Name",
  "description": "",
  "creatures": [ ... ]
}
```

The `creatures` array contains the same creature objects as single exports.

## Key Schema Notes

### Actions and Abilities are Free-Text

All actions, reactions, abilities, and legendary actions are stored as simple `{name, description}` pairs with **free-text descriptions**. Attack rolls, damage, and to-hit bonuses are embedded in the description text, not structured data.

Example action description:
```
"Melee Weapon Attack: +5 to hit, reach 5 ft., one target. Hit: 10 (2d6 + 3) slashing damage."
```

This means parsing structured attack data from CritterDB requires regex extraction from description text. The Foundry VTT importer notes that "automatically handling attacks and spells is next to impossible."

### Ability Scores vs Modifiers

CritterDB stores raw ability scores (10-30 range), not modifiers. Modifiers must be calculated: `Math.floor((score - 10) / 2)`.

### Saving Throws and Skills

- `savingThrows[].proficient` - boolean, whether proficient
- `savingThrows[].value` - override value (null = calculate from ability + proficiency)
- `skills[].proficient` - boolean
- `skills[].value` - override value (null = calculate)

### Challenge Rating

Stored as a number. Fractional CRs (1/8, 1/4, 1/2) are stored as `0.125`, `0.25`, `0.5`.

### Available Export Formats

CritterDB can export in three formats:
1. **CritterDB JSON** - native format (documented above)
2. **5E Tools** - mapped to 5etools schema via data adapter
3. **Improved Initiative** - creature-level only, not bestiary-level

## Import Considerations for EncounterManager

### Direct Mappings
| CritterDB Field | EncounterManager Field |
|---|---|
| `name` | `name` |
| `stats.size` | `size` |
| `stats.armorClass` | `ac` |
| `stats.hitDieSize` + `stats.numHitDie` | `hpFormula` |
| `stats.speed` | `speed` (string, may need parsing) |
| `stats.abilityScores.*` | `abilities.*` |
| `stats.savingThrows[]` | `savingThrows[]` |
| `stats.skills[]` | `skills[]` |
| `stats.senses[]` | `senses` |
| `stats.languages[]` | `languages` |
| `stats.challengeRating` | `cr` |
| `stats.additionalAbilities[]` | `features[]` |
| `stats.legendaryActionsPerRound` | `legendaryActionBudget` |
| `stats.legendaryActions[]` | `legendaryActions[]` |

### Requires Parsing
| CritterDB Field | Challenge |
|---|---|
| `stats.actions[]` | Free-text - need regex to extract to-hit, damage dice, damage types |
| `stats.reactions[]` | Free-text - map to features or separate field |
| `stats.armorType` | String - may need mapping to AC source |

### Not Available in CritterDB
- `critRange` (EncounterManager homebrew field)
- Structured attack data (must be parsed from action descriptions)
- Feature recharge/uses (embedded in description text, e.g., "Recharge 5-6")

## Research Date

February 2026
