# 5etools Creature JSON Schema Reference

A community reference for the 5etools creature/monster JSON format, derived from the [official JSON schema](https://github.com/TheGiddyLimit/5etools-utils/tree/master/schema-template/bestiary) (v1.21.60) and verified against real creature data.

This document is maintained by [PromptFerret](https://github.com/promptferret) for use by tool authors who import 5etools monster data. It is not affiliated with or endorsed by 5etools.

## Source Schema

- **Repository**: [TheGiddyLimit/5etools-utils](https://github.com/TheGiddyLimit/5etools-utils)
- **Bestiary schema template**: `schema-template/bestiary/bestiary.json`
- **Utility definitions**: `schema-template/util.json`
- **Entry definitions**: `schema-template/entry.json`
- **Schema variants**: `schema/site/`, `schema/brew/`, `schema/ua/` (generated from templates)
- **Homebrew wiki**: [wiki.tercept.net/en/Homebrew/Tools/Schema](https://wiki.tercept.net/en/Homebrew/Tools/Schema)
- **Homebrew examples**: [TheGiddyLimit/homebrew/creature/](https://github.com/TheGiddyLimit/homebrew/tree/master/creature)

## File Structure

A 5etools bestiary file:
```json
{
  "_meta": {
    "sources": [{ "json": "SourceCode", "abbreviation": "SC", "full": "Source Name", "authors": [], "version": "1.0" }],
    "dateAdded": 0,
    "dateLastModified": 0
  },
  "monster": [ /* array of creature objects */ ]
}
```

A single creature copied from the 5etools website is a top-level JSON object (no `monster` wrapper).

**Required fields** for a valid creature: `name`, `source`, `size`, `type`.

---

## Creature Properties

### Identity & Classification

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | **Required**. Creature name. |
| `source` | string | **Required**. Source book code (e.g. `"MM"`, `"XMM"`, `"PHB"`). |
| `size` | string[] | **Required**. Size codes array. Usually one element, but shapeshifters may have multiple (e.g. `["S", "M"]`). |
| `type` | string \| object | **Required**. See Type below. |
| `shortName` | string \| boolean | Shortened name for legendary action headers. `true` = use `name` as-is. |
| `alias` | string[] | Alternative names for search. |
| `group` | string[] | Creature group (e.g. `["Chromatic Dragon"]`, `["Devils"]`). |
| `level` | integer | Sidekick level. |
| `sizeNote` | string | Additional size info text. |
| `alignment` | array | Alignment codes. See Alignment below. |
| `alignmentPrefix` | string | Text before alignment. |
| `isNamedCreature` | true \| null | Is this a named NPC? |
| `isNpc` | true \| null | Is this an NPC (used for adventure NPCs)? |
| `familiar` | true \| null | Can be used as a familiar? |

#### Size Codes

| Code | Size |
|------|------|
| `F` | Fine (homebrew only) |
| `D` | Diminutive (homebrew only) |
| `T` | Tiny |
| `S` | Small |
| `M` | Medium |
| `L` | Large |
| `H` | Huge |
| `G` | Gargantuan |
| `C` | Colossal (homebrew only) |
| `V` | Varies |

#### Type

Either a simple string or an object:

```
// Simple
"undead"

// Object
{
  "type": "aberration",        // or { "choose": ["beast", "monstrosity"] }
  "tags": ["goblin", "Skirmisher"],  // string or { tag, prefix, prefixHidden }
  "swarmSize": "T",            // size of individual creatures in a swarm
  "sidekickType": "warrior",   // "expert" | "spellcaster" | "warrior"
  "note": "..."                // additional type note
}
```

**Creature types**: `aberration`, `beast`, `celestial`, `construct`, `dragon`, `elemental`, `fey`, `fiend`, `giant`, `humanoid`, `monstrosity`, `ooze`, `plant`, `undead`

#### Alignment Codes

`L` (Lawful), `N` (Neutral), `NX` (Neutral on law/chaos axis), `NY` (Neutral on good/evil axis), `C` (Chaotic), `G` (Good), `E` (Evil), `U` (Unaligned), `A` (Any alignment)

Can also be an object with `{ alignment, chance, note }` or `{ special }`.

---

### Combat Statistics

| Field | Type | Description |
|-------|------|-------------|
| `ac` | array | Armor Class. Array of `acItem`. See AC below. |
| `hp` | object | Hit Points. `{ average, formula }` or `{ special }`. |
| `speed` | object \| integer \| "Varies" | Movement speeds. See Speed below. |
| `initiative` | object \| number | Initiative modifier. See Initiative below. |
| `cr` | string \| object | Challenge Rating. See CR below. |
| `pbNote` | string | Proficiency bonus note. |

#### AC (Armor Class)

Each element in the `ac` array is one of:

```
// Plain number
23

// Object with details
{
  "ac": 15,
  "from": ["{@item leather armor|PHB}", "{@item shield|PHB}"],  // sources (may contain tags)
  "condition": "(in humanoid form)",    // when this AC applies
  "braces": true                        // display formatting hint
}

// Special text
{ "special": "varies based on form" }
```

#### HP (Hit Points)

```
// Standard
{ "average": 71, "formula": "11d8 + 22" }

// Special (scaling/custom)
{ "special": "equal to the summoner's level" }
```

#### Speed

```
// Object (most common)
{
  "walk": 30,              // can be integer or { number, condition }
  "burrow": 20,
  "climb": 30,
  "fly": 60,
  "swim": 40,
  "canHover": true,        // appended as "(hover)" to fly speed
  "alternate": {           // conditional alternate speeds
    "walk": [{ "number": 40, "condition": "(wolf form only)" }],
    "fly": [{ "number": 80, "condition": "(dragon form only)" }]
  },
  "choose": {              // choose from speed modes
    "from": ["walk", "burrow", "climb"],
    "amount": 30,
    "note": "one of the listed"
  }
}

// Simple integer (walk speed only)
30

// Special
"Varies"
```

Speed values can be:
- An integer (e.g. `30`)
- An object `{ "number": 40, "condition": "(wolf form only)" }`
- `true` (for "equal to walking speed" cases)

#### Initiative (2024 rules)

```
// Object
{
  "initiative": 5,        // flat bonus (if set directly)
  "proficiency": 1,       // 1 = proficiency, 2 = expertise
  "advantageMode": "adv"   // "adv" or "dis"
}

// Simple number
5
```

If absent, initiative is typically DEX modifier only (older/homebrew data).

#### Challenge Rating

```
// Simple string
"3"
"1/4"
"1/2"

// Object
{
  "cr": "13",
  "lair": "15",           // CR when in lair
  "coven": "15",          // CR when in coven
  "xp": 10000,            // custom XP override
  "xpLair": 11500         // XP when in lair
}
```

---

### Ability Scores

| Field | Type | Description |
|-------|------|-------------|
| `str` | integer \| null \| { special } | Strength |
| `dex` | integer \| null \| { special } | Dexterity |
| `con` | integer \| null \| { special } | Constitution |
| `int` | integer \| null \| { special } | Intelligence |
| `wis` | integer \| null \| { special } | Wisdom |
| `cha` | integer \| null \| { special } | Charisma |

Top-level fields (not nested). Can be null for creatures without standard scores. `{ "special": "text" }` for non-standard cases.

---

### Saving Throws & Skills

| Field | Type | Description |
|-------|------|-------------|
| `save` | object \| null | Saving throw bonuses. Keys: `str`, `dex`, `con`, `int`, `wis`, `cha`. Values: strings like `"+11"`. |
| `skill` | object \| null | Skill bonuses. Keys: standard D&D skill names (lowercase, e.g. `"perception"`, `"animal handling"`, `"sleight of hand"`). Values: strings like `"+7"`. May have `other` array for conditional skills. |
| `tool` | object \| null | Tool proficiency bonuses. Keys: tool names. Values: strings. |

**Standard skills**: `acrobatics`, `animal handling`, `arcana`, `athletics`, `deception`, `history`, `insight`, `intimidation`, `investigation`, `medicine`, `nature`, `perception`, `performance`, `persuasion`, `religion`, `sleight of hand`, `stealth`, `survival`

---

### Senses & Communication

| Field | Type | Description |
|-------|------|-------------|
| `senses` | string[] \| null | Sense descriptions (e.g. `["Darkvision 60 ft.", "Tremorsense 30 ft."]`). |
| `passive` | integer \| string \| null | Passive Perception score. |
| `languages` | string[] \| null | Languages known (e.g. `["Common", "Draconic"]`, `["Common (can't speak in wolf form)"]`). |

---

### Damage & Condition Immunities

All three damage arrays (`resist`, `immune`, `vulnerable`) share the same format. Each array element is one of:

```
// Simple damage type
"fire"

// Conditional
{
  "resist": ["bludgeoning", "piercing", "slashing"],  // or "immune" or "vulnerable"
  "note": "from nonmagical attacks",
  "preNote": "text before",    // optional
  "cond": true                 // marks as conditional
}

// Special text
{ "special": "damage from spells" }
```

| Field | Type | Description |
|-------|------|-------------|
| `resist` | array \| null | Damage resistances. |
| `immune` | array \| null | Damage immunities. |
| `vulnerable` | array \| null | Damage vulnerabilities. |
| `conditionImmune` | array \| null | Condition immunities. Same nested format (with `conditionImmune` key instead of damage key). |

**Damage types**: `acid`, `bludgeoning`, `cold`, `fire`, `force`, `lightning`, `necrotic`, `piercing`, `poison`, `psychic`, `radiant`, `slashing`, `thunder`

**Conditions**: `blinded`, `charmed`, `deafened`, `exhaustion`, `frightened`, `grappled`, `incapacitated`, `invisible`, `paralyzed`, `petrified`, `poisoned`, `prone`, `restrained`, `stunned`, `unconscious`, `disease`

---

### Abilities & Actions

All ability arrays (`trait`, `action`, `bonus`, `reaction`, `legendary`, `mythic`) use the same entry format:

```json
{
  "name": "Multiattack",
  "entries": ["The creature makes two attacks."]
}
```

For `legendary` and `mythic`, `name` is optional (unnamed entries are introductory/header text).

For `trait`, there are two additional optional fields:
- `type`: `"entries"` or `"inset"` (display hint)
- `sort`: integer (forces sort order)

| Field | Type | Description |
|-------|------|-------------|
| `trait` | array \| null | Passive traits/features. |
| `action` | array \| null | Actions (includes attacks and multiattack). |
| `actionNote` | string | Text displayed before actions section. |
| `actionHeader` | entry[] | Custom action section header. |
| `bonus` | array \| null | Bonus actions. |
| `bonusNote` | string | Text displayed before bonus actions. |
| `bonusHeader` | entry[] | Custom bonus action section header. |
| `reaction` | array \| null | Reactions. |
| `reactionNote` | string | Text displayed before reactions. |
| `reactionHeader` | entry[] | Custom reaction section header. |
| `legendary` | array \| null | Legendary actions. |
| `legendaryActions` | integer (min 1) | Legendary action budget (default 3). |
| `legendaryActionsLair` | integer (min 1) | Legendary action budget when in lair. |
| `legendaryHeader` | entry[] | Custom legendary action header text. |
| `legendaryGroup` | { name, source } | Reference to legendary group data (lair actions, regional effects). |
| `mythic` | array \| null | Mythic actions (e.g. Tiamat). |
| `mythicHeader` | entry[] | Custom mythic action header text. |

---

### Spellcasting

The `spellcasting` field is an array of spellcasting entries, each with a complex structure:

```json
{
  "name": "Charm {@recharge 5}",
  "type": "spellcasting",
  "headerEntries": ["The vampire casts {@spell Charm Person|XPHB}..."],
  "recharge": { "5": ["{@spell Charm Person|XPHB}"] },
  "ability": "cha",
  "displayAs": "bonus",
  "hidden": ["recharge"]
}
```

| Sub-field | Type | Description |
|-----------|------|-------------|
| `name` | string | Name (may contain `{@recharge}` tag). |
| `type` | string | Always `"spellcasting"`. |
| `headerEntries` | string[] | Description text entries. |
| `ability` | string | Spellcasting ability (`"int"`, `"wis"`, `"cha"`, etc.). |
| `displayAs` | string | Where it appears: `"action"`, `"bonus"`, `"legendary"`, `"trait"`. |
| `recharge` | object | Keyed by recharge number (e.g. `"5"` for recharge 5-6). |
| `spells` | object | Traditional spell slots (keyed by level `"0"` through `"9"`). |
| `will` | string[] | At-will spells. |
| `daily` | object | Daily-use spells (keyed by uses, e.g. `"1e"`, `"2e"`, `"3e"`). |
| `hidden` | string[] | UI display hints (e.g. `["recharge"]`). |

---

### Entries Format (Rich Text)

Entries in `trait`, `action`, etc. are arrays where each element is either a **string** (possibly with rich text tags) or a **structured object**:

#### Structured Entry Types

```
// List
{ "type": "list", "style": "list-hang-notitle", "items": [ ... ] }

// Named item (within a list)
{ "type": "item", "name": "Forbiddance", "entries": ["The vampire can't enter..."] }

// Inset/variant block
{ "type": "inset", "name": "Title", "entries": [...], "source": "...", "page": 0 }

// Table
{ "type": "table", "caption": "...", "colLabels": [...], "colStyles": [...], "rows": [[...]] }
```

#### Rich Text Tags

Entries use `{@tag content|source|...}` markup. The general rule is: take the first segment before `|`.

**Dice & Combat:**
| Tag | Output | Notes |
|-----|--------|-------|
| `{@damage 5d10 + 10}` | `5d10 + 10` | Damage dice |
| `{@dice 2d6}` | `2d6` | Generic dice |
| `{@hit 19}` | `+19` | Attack bonus |
| `{@dc 27}` | `DC 27` | Difficulty class |
| `{@h}` | `Hit: ` | Hit description marker |
| `{@recharge 5}` | `(Recharge 5-6)` | Recharge notation |
| `{@recharge}` | `(Recharge 6)` | Recharge 6 only |

**Attack Indicators (stripped to empty string, used for detection):**
| Tag | Format |
|-----|--------|
| `{@atkr m}` | 2024: melee attack |
| `{@atkr r}` | 2024: ranged attack |
| `{@atkr ms}` | 2024: melee spell attack |
| `{@atkr rs}` | 2024: ranged spell attack |
| `{@atk mw}` | Older: melee weapon attack |
| `{@atk rw}` | Older: ranged weapon attack |
| `{@atk ms}` | Older: melee spell attack |
| `{@atk rs}` | Older: ranged spell attack |

**Save/Effect Tags (2024):**
| Tag | Output |
|-----|--------|
| `{@actSave con}` | `Constitution saving throw` |
| `{@actSaveFail}` | `Failure:` |
| `{@actSaveSuccess}` | `Success:` |
| `{@actSaveSuccessOrFail}` | `Success or Failure:` |

**Reference Tags (take first segment before `|`):**
| Tag | Example | Output |
|-----|---------|--------|
| `{@condition X\|SRC}` | `{@condition Grappled\|XPHB}` | `Grappled` |
| `{@spell X\|SRC}` | `{@spell Wish\|XPHB}` | `Wish` |
| `{@creature X\|SRC}` | `{@creature Zombie\|MM}` | `Zombie` |
| `{@item X\|SRC}` | `{@item Shield\|XPHB}` | `Shield` |
| `{@skill X}` | `{@skill Perception}` | `Perception` |
| `{@action X}` | `{@action Disengage}` | `Disengage` |
| `{@variantrule X\|SRC}` | `{@variantrule Hit Points\|XPHB}` | `Hit Points` |
| `{@table X\|SRC}` | `{@table Mutations\|Source}` | `Mutations` |

---

### Equipment & Gear

| Field | Type | Description |
|-------|------|-------------|
| `gear` | array \| null | Equipment carried. UIDs like `"longbow\|xphb"` or objects `{ item, quantity, displayName }`. |
| `attachedItems` | string[] | Item UIDs attached to this creature. |

---

### Metadata Tags (for search/filtering)

These are used by 5etools for filtering and are **not needed for stat block rendering**. Tool authors can safely ignore these.

| Field | Description | Values |
|-------|-------------|--------|
| `traitTags` | Notable trait categories | `Aggressive`, `Ambusher`, `Amorphous`, `Amphibious`, `Antimagic Susceptibility`, `Beast of Burden`, `Brute`, `Camouflage`, `Charge`, `Damage Absorption`, `Death Burst`, `Devil's Sight`, `False Appearance`, `Fey Ancestry`, `Flyby`, `Hold Breath`, `Illumination`, `Immutable Form`, `Incorporeal Movement`, `Keen Senses`, `Legendary Resistances`, `Light Sensitivity`, `Magic Resistance`, `Magic Weapons`, `Mimicry`, `Pack Tactics`, `Pounce`, `Rampage`, `Reckless`, `Regeneration`, `Rejuvenation`, `Shapechanger`, `Siege Monster`, `Sneak Attack`, `Spell Immunity`, `Spider Climb`, `Sunlight Sensitivity`, `Tree Stride`, `Tunneler`, `Turn Immunity`, `Turn Resistance`, `Undead Fortitude`, `Unusual Nature`, `Water Breathing`, `Web Sense`, `Web Walker` |
| `actionTags` | Notable action categories | `Breath Weapon`, `Frightful Presence`, `Multiattack`, `Parry`, `Shapechanger`, `Swallow`, `Teleport`, `Tentacles` |
| `senseTags` | Sense type codes | `B` (Blindsight), `D` (Darkvision), `SD` (Superior Darkvision), `T` (Tremorsense), `U` (Truesight) |
| `languageTags` | Language codes | `AB` (Abyssal), `AQ` (Aquan), `AU` (Auran), `C` (Common), `CE` (Celestial), `CSL` (Common Sign Language), `D` (Dwarvish), `DR` (Draconic), `DS` (Deep Speech), `DU` (Druidic), `E` (Elvish), `G` (Gnomish), `GI` (Giant), `GO` (Goblin), `GTH` (Gith), `H` (Halfling), `I` (Infernal), `IG` (Ignan), `O` (Orc), `P` (Primordial), `S` (Sylvan), `T` (Terran), `TC` (Thieves' cant), `U` (Undercommon), `X` (Any), `XX` (All), `CS` (Can't Speak), `LF` (Languages Known in Life), `TP` (Telepathy), `OTH` (Other) |
| `damageTags` | Damage type codes | `A` (Acid), `B` (Bludgeoning), `C` (Cold), `F` (Fire), `O` (Force), `L` (Lightning), `N` (Necrotic), `P` (Piercing), `I` (Poison), `Y` (Psychic), `R` (Radiant), `S` (Slashing), `T` (Thunder) |
| `damageTagsSpell` | Damage from spells | Same codes as `damageTags` |
| `damageTagsLegendary` | Damage from LA | Same codes as `damageTags` |
| `spellcastingTags` | Spellcasting source | `P` (Psionics), `I` (Innate), `F` (Form Only), `S` (Shared), `O` (Other), `CA`-`CW` (Class-specific) |
| `miscTags` | Miscellaneous tags | `AOE` (Areas of Effect), `CUR` (Curse), `DIS` (Disease), `HPR` (HP Reduction), `MW` (Melee Weapon), `RW` (Ranged Weapon), `MA` (Melee Attack), `RA` (Ranged Attack), `RCH` (Reach), `MLW` (Melee Weapon+), `RNG` (Ranged), `THW` (Thrown Weapon) |
| `conditionInflict` | Conditions inflicted | Standard condition names |
| `conditionInflictSpell` | Conditions from spells | Standard condition names |
| `conditionInflictLegendary` | Conditions from LA | Standard condition names |
| `savingThrowForced` | Saves forced | Ability names |
| `savingThrowForcedSpell` | Saves from spells | Ability names |
| `savingThrowForcedLegendary` | Saves from LA | Ability names |

---

### Publishing & Cross-References

These fields are for 5etools internal use and can be ignored by importers.

| Field | Type | Description |
|-------|------|-------------|
| `page` | integer | Page number in source book. |
| `otherSources` | array | Additional source references. |
| `additionalSources` | array | Additional sources. |
| `referenceSources` | array | Books that reference this creature. |
| `reprintedAs` | array | Reprinted versions. |
| `isReprinted` | boolean | Has been reprinted? |
| `srd` | boolean | In the SRD (5.0)? |
| `srd52` | boolean | In the SRD (5.2)? |
| `basicRules` | boolean | In Basic Rules? |
| `basicRules2024` | boolean | In 2024 Basic Rules? |
| `legacy` | boolean | Legacy content? |
| `sourceSub` | string | Sub-source text. |

---

### Tokens & Art

| Field | Type | Description |
|-------|------|-------------|
| `hasToken` | true | Has a token image. |
| `tokenCredit` | string | Token credit text. |
| `token` | { name, source } | Reference to reuse another creature's token. |
| `tokenHref` | { type, url } | Direct URL to token image. |
| `tokenUrl` | string | Token URL (homebrew). |
| `altArt` | array | Alternative art references. |
| `soundClip` | { type, path } | Sound effect reference. |
| `hasFluff` | true | Has fluff/lore text. |
| `hasFluffImages` | true | Has fluff images. |

---

### Dragon-Specific

| Field | Type | Description |
|-------|------|-------------|
| `dragonCastingColor` | string | Dragon color for spellcasting variant. Values: `black`, `blue`, `green`, `red`, `white`, `brass`, `bronze`, `copper`, `gold`, `silver`, `deep`, `spirit`. |
| `dragonAge` | string | Dragon age. Values: `wyrmling`, `young`, `adult`, `ancient`, `greatwyrm`, `aspect`. |

---

### Summoning

| Field | Type | Description |
|-------|------|-------------|
| `summonedBySpell` | string | Spell that summons this creature. |
| `summonedBySpellLevel` | integer | Spell level required. |
| `summonedByClass` | string | Class that can summon this creature. |
| `summonedScaleByPlayerLevel` | true | Scales by summoner's level. |

---

### Environment & Treasure

| Field | Type | Description |
|-------|------|-------------|
| `environment` | string[] | Terrain/plane types. Values include: `any`, `underwater`, `coastal`, `mountain`, `grassland`, `hill`, `arctic`, `urban`, `forest`, `swamp`, `underdark`, `desert`, `badlands`, `farmland`, and 30+ planar types (e.g. `planar, nine hells`). |
| `treasure` | string[] | Treasure categories: `any`, `individual`, `arcana`, `armaments`, `implements`, `relics`. |

---

### Homebrew-Only Fields

| Field | Type | Description |
|-------|------|-------------|
| `resource` | array | Custom resources `[{ name, value, formula }]`. |
| `fluff` | object | Inline lore/art `{ entries[], images[] }`. |
| `externalSources` | array | External source references. |
| `footer` | entry[] | Footer entries. |
| `_isCopy` | boolean | Internal: this creature is a copy of another. |
| `_versions` | array | Version history for variants. |
| `variant` | array | Variant stat block entries. |

---

### Proficiency Bonus by CR

For computing initiative bonus when `initiative.proficiency` is present:

| CR | Prof. Bonus |
|----|-------------|
| 0 - 4 | +2 |
| 5 - 8 | +3 |
| 9 - 12 | +4 |
| 13 - 16 | +5 |
| 17 - 20 | +6 |
| 21 - 24 | +7 |
| 25 - 28 | +8 |
| 29+ | +9 |

Fractional CRs (1/8, 1/4, 1/2) use +2.

---

## Tools That Use This Format

- [5etools](https://5e.tools/) — the source of this data format
- [Foundry VTT Plutonium](https://5e.tools/plutonium.html) — imports 5etools JSON into Foundry VTT
- [ttrpg-convert-cli](https://github.com/ebullient/ttrpg-convert-cli) — converts 5etools JSON to Obsidian markdown
- [5etools-monster-grabber](https://github.com/jarrodwhitley/5etools-monster-grabber) — converts to Shieldmaiden format
- [Bestiary Builder](https://bestiarybuilder.com/) — creature builder with 5etools import
- [PromptFerret Encounter Manager](https://promptferret.github.io/tools/encounter/) — D&D encounter & initiative tracker with 5etools import

---

*Last updated: 2026-02-20. Schema version: 1.21.60.*
