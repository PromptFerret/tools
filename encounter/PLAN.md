# Encounter Manager — Implementation Plan

## Overview

D&D encounter manager and initiative tracker for DMs running combat. Single self-contained HTML file at `encounter/index.html`. No server, no accounts, no dependencies — IndexedDB persistence (with localStorage fallback).

Built for heroic/homebrew D&D — CR100 monsters, level 60 players, non-standard rules are expected use cases.

## Completed Phases

### Phase 1 — Foundation (done)
- HTML structure: header, nav tabs (Monsters/Parties/Encounters/Combat), status bar, footer
- CSS: full PromptFerret dark theme (`--bg: #0f0f17`, `--surface: #1a1a2e`, `--accent: #7c5cbf`)
- View switching with dirty check warnings
- Monster template CRUD (list with search, form with all fields including abilities, attacks with multiple damage types, features, legendary actions/resistances)
- Encounter CRUD (name, location, campaign, notes, monster picker with qty)
- Party CRUD with per-party player list (type name + Add, remove with × button)
- Dice engine: `rollDice(notation, opts)` — NdN+M, advantage/disadvantage
- Passive Perception auto-calc
- localStorage persistence with `pf_enc_` namespace
- Save/Close/Cancel form pattern with dirty tracking

### Phase 2 — Combat Core (done)
- Combat state machine: empty → party select → init setup (round 0) → active combat (round 1+)
- `launchCombat(partyId)`: creates numbered monster instances, rolls monster init with breakdown text stored as `initRollText`
- Initiative setup: monsters pre-rolled with roll text shown left of input, players blank (they roll in Discord/Avrae)
- Active combat: column headers (Init/Name/AC/HP/R), expandable combatant rows
- Combatant row: init | name (color-coded by type) | AC | HP bar (color-coded) | reaction indicator
- Detail panel (monsters): HP controls, multiattack box, attack cards, roll log, stats, features, tactics
- Detail panel (players): AC input, notes
- Detail panel (all): init edit, notes, kill/revive/remove
- HP tracking: single input + Dmg(red)/Heal(green)/THP(yellow) buttons. THP takes higher (doesn't stack per D&D rules)
- Attack rolling: Atk/Adv(green)/Dis(red) buttons per attack. Crit detection on chosen d20
- Damage rolling: Dmg/Crit(green) buttons. Crit doubles dice count not modifiers (NdM → 2NdM)
- Crit highlighting: nat 20 green/bold "NAT 20!", nat 1 red/bold "NAT 1"
- Roll log: stacking chronological log per combatant, persisted to localStorage, survives refreshes, clears on that combatant's next turn start
- Multiattack: prominent yellow-bordered box with "Multiattack:" prefix
- Turn management: next/prev with round tracking, reaction auto-reset on turn start, roll log clear on turn start
- Init drop: editing current combatant's init ends their turn and advances to next
- Ad-hoc combatants: sorted into position when added during active combat
- Player AC: editable in detail panel, shown in row
- Encounter delete clears active combat if linked
- Auto-save on every mutation, resume on page load

## Phase 3 — Combat Features (done)

- Crit range on template schema (`critRange`, default 20, labeled "Crits On" in form, customizable for homebrew 18-20 etc.)
- Full feature/trait text display in combat detail panel with inline clickable dice rolling (`renderDiceText()`)
- Inline dice roll log entries include source feature/LA name (e.g., "Fire Breath: 2d6+3")
- Feature use tracking: `featureUses` sparse map on combatant, Use/Restore buttons (accent/success colored), depleted state display
- DM-initiated recharge: notification in roll log at turn start, Roll Recharge button (warning colored) in detail panel
- Legendary actions: LA budget display on row (badge) and detail panel, per-action Use buttons (cost-aware), Restore LA button for DM take-backs, auto-recharge on turn start
- Legendary resistances: LR counter on row (badge) and detail panel, Use LR + Restore LR buttons
- LA/LR use and restore actions logged to combatant rollLog
- Ability check/save rolling: Chk/Save buttons per ability score in monster detail panel stats section
- Condition/effect tracking: 14 standard D&D conditions + custom text, three duration types (rounds/save-based/indefinite), condition chips on ALL combatant rows, add/remove UI in all detail panels
- Concentration tracking: text input per combatant, chip on row, damage triggers CON save DC warning (`max(10, damage/2)`)
- Consolidated turn-start hook: `applyTurnStartEffects(idx)` handles reaction reset, rollLog clear, recharge prompts, LA recharge, condition decrement/save reminders
- Turn-start notification badges on combatant rows (pulse animation) when panel is collapsed
- Button class system: `.btn-accent`, `.btn-success`, `.btn-warning`, `.btn-danger`, `.btn-remove` — never use inline color styles on buttons (breaks hover contrast)
- Features and legendary actions styled as cards (background + border-radius) matching attack cards
- Roster chips sized for comfortable × button interaction

### Phase 3 Data Changes
New fields on combatant instances (sparse, added on mutation only):
```json
{
  "conditions": [
    { "name": "Frightened", "durationType": "rounds", "duration": 3, "source": "Dragon Fear" },
    { "name": "Stunned", "durationType": "save", "saveDC": 15, "saveAbility": "con", "source": "Power Word Stun" },
    { "name": "Prone", "durationType": "indefinite", "source": "" }
  ],
  "concentration": "Wall of Fire",
  "legendaryActionsRemaining": 3,
  "legendaryResistancesRemaining": 3,
  "featureUses": { "0": 2, "3": 0 }
}
```
New field on monster template:
```json
{
  "critRange": 20
}
```

### Deferred to Phase 4+
- Surprise round handling
- Dynamic notes/riders with round expiry
- Template override highlighting
- Damage log (collapsible panel)

## Phase 3.5 — IndexedDB Migration + No-Cache (done)

- Migrated from localStorage (~5-10MB) to IndexedDB (~50MB+)
- Auto-migration: on first load, reads localStorage data, writes to IndexedDB, clears localStorage
- Fallback: if IndexedDB unavailable (Safari `file://`), silently falls back to localStorage via `_useLocalStorage` flag
- `save(key)` is fire-and-forget async — callers unchanged, in-memory `state` is source of truth
- `load()` is async — init block changed to `load().then(render)`
- Database: `pf_encounter`, version 1, single object store `state`
- No-cache meta tags added (`Cache-Control`, `Pragma`, `Expires`) so browsers always fetch latest version
- Init value 0 fix: initiative of 0 now displays correctly (not as empty). Null/undefined init is distinct from 0 — empty means "not set", 0 means "rolled zero". All sorting uses `?? -Infinity` so null-init combatants sort to bottom. `beginCombat()` validation checks for null/undefined, not falsy.

## Phase 4 — Multi-Combat, SquishText Export/Import & Polish (done)

### Multi-Combat Support
- `state.combat` (single object) → `state.combats` (array) + `activeCombatId` (transient)
- `pf_enc_combats` IndexedDB key (auto-migrates from old `pf_enc_combat` single-object key)
- `getActiveCombat()` helper replaces all direct `state.combat` references
- Combat list view: cards for each combat showing name, round, turn, alive count, player names
- Click card to resume, "Back to List" button in combat bar when multiple combats exist
- Each combat gets `id` (UUID) and `name` (auto-generated from encounter + party names)
- `endCombat()` removes from array instead of setting null
- `startCombat()` clears `activeCombatId` before switching to combat view (ensures party select shows)
- `switchView('combat')` clears `activeCombatId` when no pending encounter (ensures combat list shows)

### SquishText Import/Export
- Toolbar buttons: "Save Backup" / "Load Backup"
- Save Backup downloads `.squishtext` file (SquishText-encoded: deflate-raw → base64 → CRC32 → header)
- Load Backup uses direct file picker (no modal) — creates hidden `<input type="file">`, processes on change
- SquishText format only (no JSON fallback)
- Smart merge on import: skips duplicates by ID, keeps existing data, adds new items only
- Compression functions (`compress`, `decompress`, `crc32`) embedded directly
- Export payload: `{ version, exported, templates, encounters, parties, combats }`

### Per-Monster Import/Export
- Export button on each monster card — downloads single template as `.squishtext`
- "Import Monster" button in toolbar (visible on Monsters tab only)
- Import Monster modal with multiselect file input — processes multiple `.squishtext` files at once
- Smart merge: dedupes by ID across all selected files, skips existing templates
- Works with both full backup files and single-monster exports (reads `templates` array from payload)

### Searchable Monster Picker
- Replaced `<select>` in encounter form with text input + filtered dropdown (`.search-select`)
- `filterMonsterPicker()` filters by name (case-insensitive), `selectMonster()` sets `data-template-id`
- Dropdown closes on click outside, shows CR tag per monster

### Storage Indicator
- Footer shows "Storage: X.X KB / 2.0 GB" via `navigator.storage.estimate()`
- Auto-scales units: B → KB → MB → GB for both usage and quota
- Updated after load and debounced after saves (2s timeout)

## Phase 4.1 — Per-Party Player Lists (done)

- Removed shared player roster (`state.players` array, `pf_enc_players` storage key)
- Each party now owns its own `players[]` array — self-contained, no shared state
- Party form simplified: text input + Add button, player chips with × remove button
- Removed `togglePlayerInParty()`, `removeFromRoster()` functions
- `addPlayerToParty()` simplified — just pushes to `formData.players`, no shared roster
- New `removePlayerFromParty(idx)` — splices from `formData.players`
- CSS: `.roster-chip`/`.roster-grid` renamed to `.player-chip`/`.player-list`, removed toggle/selected styling
- Import migration: old backups with `players` array merge roster names into all parties
- Storage cleanup: `pf_enc_players` key deleted from IndexedDB on load

## Future Phases

### Phase 5 — 5etools Importer

#### Importer Architecture Pattern (applies to all future importers)

All importers follow the same pattern:
1. **Pure converter function**: `importXxx(json) → template[]` — takes parsed JSON, returns array of our template objects. No side effects, no DOM, no state mutation.
2. **Format detection**: `doImportMonster()` currently only handles `.squishtext` files. For Phase 5+, update it to also accept `.json` files AND clipboard paste. Detection order: try SquishText decompress first → if that fails, try JSON.parse → if valid JSON, detect which format (our native format has `version` + `templates`, 5etools has `monster` array or top-level 5etools fields like `str`/`dex`/`ac` as array).
3. **Input methods**: File upload (`.squishtext`, `.json`) AND paste textarea. Users copy JSON from 5etools via clipboard — paste is the primary UX for external imports.
4. **Smart merge**: Same ID-based dedup as existing import. Imported templates get new `uuid()` IDs (since external formats don't use our IDs).
5. **Status reporting**: Show count of imported vs skipped (by name dedup for external formats since they have no IDs — dedup by name to avoid importing the same monster twice).

#### 5etools Data Source

Users copy JSON from the 5etools website via clipboard (not file download). The site shows monster JSON and users copy it. This means:
- A single monster = a top-level JSON object (most common use case)
- A multi-monster export = `{ "monster": [ ...entries... ] }` wrapper
- Import UX should support both `.json` file upload AND clipboard paste

**Detection**: If parsed JSON has a `monster` array → 5etools multi. If it has `str` (number) + `ac` (array) → 5etools single.

#### 5etools Monster Entry — Key Fields

Verified against official JSON schema (v1.21.60 from `TheGiddyLimit/5etools-utils`) and 6 real samples: The World Ender (CR 30 aberration, homebrew), Goblin Warrior (CR 1/4, MCDM), Goblin Archer (CR 1/4, LotR homebrew), Werewolf (CR 3, XMM 2024), Vampire (CR 13, XMM 2024), Pit Fiend (CR 20, XMM 2024).

Full schema reference: `tools/5ETOOLS_CREATURE_SCHEMA.md`

Fields we care about (fields we ignore are documented in the schema reference):

```
name            string          "The World Ender"
size            string[]        ["G"] or ["S", "M"]. Codes: F/D/T/S/M/L/H/G/C/V (C=Colossal, V=Varies — homebrew only)
type            string|object   "undead" or { type: "aberration", tags: ["Epic Boss"] }
ac              array           [23] or [{ ac: 15, from: [...], condition: "..." }] or [{ special: "..." }]
hp              object          { average: 60, formula: "8d8+24" } or { special: "..." }
speed           object|int      { walk: 60, fly: 80, canHover: true, swim: 30, burrow: 20, climb: 20 } or 30 or "Varies"
speed.alternate object|null     { walk: [{ number: 40, condition: "(wolf form only)" }] } — conditional speeds
initiative      object|number   { initiative: 5, proficiency: 1, advantageMode: "adv" } or 5. Absent in older/homebrew.
str/dex/con/int/wis/cha  int|null|{special}  Top-level ability scores (not nested). Can be null or { special: "..." }.
save            object|null     { dex: "+11", int: "+10" } — string values with +/-
skill           object|null     { athletics: "+19", perception: "+19" }. May have `other` array.
senses          string[]|null   ["Truesight 120 ft."]
passive         int|string|null 29 (passive perception). Can be a string in rare cases.
resist          array|null      ["acid", "cold"] or [{ resist: [...], note: "...", preNote: "...", cond: true }] or [{ special: "..." }]
immune          array|null      damage immunities (same format as resist)
vulnerable      array|null      damage vulnerabilities (same format as resist)
conditionImmune array|null      ["charmed", "frightened"] or [{ conditionImmune: [...], note: "..." }] — same nested pattern
languages       array|null      ["Common", "Draconic"] or ["Common (can't speak in wolf form)"]
cr              string|object   "30" or "1/4" or { cr: "13", lair: "15", coven: "15", xp: 10000, xpLair: 11500 }
trait           array|null      [{ name, entries[], type?, sort? }] — passive features
action          array|null      [{ name, entries[] }] — actions (attacks mixed in)
bonus           array|null      [{ name, entries[] }] — bonus actions (separate from action[])
reaction        array|null      [{ name, entries[] }] — reactions (separate from action[])
spellcasting    array|null      [{ name, headerEntries[], recharge, displayAs, ability }] — see below
legendary       array|null      [{ name?, entries[] }] — legendary actions (name optional for header text)
legendaryActions number|null    LA budget (default 3 if legendary array exists)
legendaryActionsLair number|null  LA budget when in lair (Vampire: 4)
legendaryHeader  entry[]|null   custom legendary action header text
mythic          array|null      [{ name?, entries[] }] — mythic actions (e.g. Tiamat)
mythicHeader    entry[]|null    custom mythic action header text
```

#### Spellcasting Array (2024 format)

New in 2024 monsters. Each entry:
```
{
  name: "Charm {@recharge 5}",           // may contain recharge tag
  type: "spellcasting",
  headerEntries: ["The vampire casts..."],  // description text
  recharge: { "5": ["{@spell Charm Person|XPHB}"] },  // keyed by recharge number
  ability: "cha",                          // spellcasting ability
  displayAs: "action"|"bonus"|"legendary", // where it appears in statblock
  hidden: ["recharge"]                     // UI hints, ignore
}
```
**Handling**: Convert to features. Extract recharge from name tag. Use `headerEntries` joined as description. Place in features array regardless of `displayAs` — the DM can use them from the feature panel.

#### Structured Entries (recursive)

Entries are usually strings but can be objects:
```
{ type: "list", style: "list-hang-notitle", items: [
  { type: "item", name: "Forbiddance", entries: ["The vampire can't enter..."] },
  { type: "item", name: "Running Water", entries: ["The vampire takes 20 Acid damage..."] }
]}
```
**Handling**: Implement `flattenEntries(entries)` that recursively processes:
- `string` → use as-is
- `{ type: "item", name, entries }` → `"Name. " + entries joined`
- `{ type: "list", items }` → process each item recursively
- Any other object with `entries` → process entries recursively
- Unknown → `JSON.stringify()` fallback

#### 5etools Rich Text Tags (must strip/convert)

Entries use `{@tag content}` markup. Two attack tag formats exist across different sources:

```
--- Dice/combat tags ---
{@damage 5d10 + 10}              → "5d10 + 10"
{@dice 2d6}                      → "2d6"
{@hit 19}                        → "+19"
{@dc 27}                         → "DC 27"
{@h}                             → "Hit: " (marks hit description)
{@recharge 5}                    → "(Recharge 5-6)" — extract for recharge field
{@recharge}                      → "(Recharge 6)"

--- Attack indicators (TWO formats) ---
{@atkr m}                        → "" (2024 format: melee attack)
{@atkr r}                        → "" (2024 format: ranged attack)
{@atkr ms}                       → "" (2024 format: melee spell attack)
{@atkr rs}                       → "" (2024 format: ranged spell attack)
{@atk mw}                        → "" (older format: melee weapon attack)
{@atk rw}                        → "" (older format: ranged weapon attack)
{@atk ms}                        → "" (older format: melee spell attack)
{@atk rs}                        → "" (older format: ranged spell attack)

--- Save/effect tags (2024) ---
{@actSave con}                   → "Constitution saving throw"
{@actSaveFail}                   → "Failure:"
{@actSaveSuccess}                → "Success:"
{@actSaveSuccessOrFail}          → "Success or Failure:"

--- Reference tags (take first segment before |) ---
{@condition Grappled|XPHB}       → "Grappled"
{@spell Wish|XPHB}               → "Wish"
{@creature Zombie|MM}            → "Zombie"
{@item Shield|XPHB}              → "Shield"
{@skill Perception}              → "Perception"
{@action Disengage}              → "Disengage"
{@variantrule Hit Points|XPHB}   → "Hit Points"
{@variantrule Speed|XPHB}        → "Speed"
{@variantrule Fly Speed|XPHB}    → "Fly Speed"
{@variantrule Advantage|XPHB}    → "Advantage"
{@variantrule Disadvantage|XPHB} → "Disadvantage"
{@variantrule Resistance|XPHB}   → "Resistance"
{@variantrule Emanation [Area of Effect]|XPHB} → "Emanation"
{@table Mutations|Source}         → "Mutations"
```

**General rule**: `{@tag content|source|...}` → take `content` (first segment before `|`). Exceptions:
- `{@h}` → `"Hit: "`
- `{@recharge N}` → `"(Recharge N-6)"` — also extract for recharge field
- `{@atkr X}` / `{@atk X}` → `""` (empty string, used only for attack detection)
- `{@actSave ABILITY}` → expand ability abbreviation to full name + "saving throw"
- `{@actSaveFail}` / `{@actSaveSuccess}` / `{@actSaveSuccessOrFail}` → label text

**Attack detection**: An action entry is an attack if it contains `{@atkr` OR `{@atk` OR `{@hit`. Must check both tag formats.

Implement as `strip5eToolsTags(text)` — regex-based, handles nested tags. Run in a loop until no more `{@` found (for nested cases).

#### Field Mapping: 5etools → Our Template

| 5etools | Our Template | Transform |
|---------|-------------|-----------|
| `name` | `name` | Direct |
| `size` | `size` | Map codes and join: `["S","M"]` → `"Small/Medium"`. Codes: F→Fine, D→Diminutive, T→Tiny, S→Small, M→Medium, L→Large, H→Huge, G→Gargantuan, C→Colossal, V→Varies |
| `type` or `type.type` | `type` | Extract string, capitalize |
| `type.tags` | (discard) | Tags like "Epic Boss" — no field for this |
| `ac[0]` or `ac[0].ac` | `ac` | Number extraction |
| `ac[0].from` | `acNote` | Join array, strip tags: `["{@item leather armor\|PHB}", "{@item shield\|PHB}"]` → `"Leather Armor, Shield"` |
| `hp.average` | `hpMax` | Direct number |
| `hp.formula` | `hpFormula` | Direct string (already in NdN+M format) |
| `hp.special` | `hpFormula` (as note) | Store in hpFormula, set hpMax to 0 or parse if possible |
| `speed` | `speed` | Format: `"30 ft., fly 60 ft. (hover), climb 20 ft."`. Append `(hover)` if `canHover: true`. Include `alternate` conditions. Handle integer/`"Varies"` fallbacks. |
| `initiative` | `initBonus` | If number: use directly. If object with `initiative`: use that. If object with `proficiency`: `dexMod + proficiency * profBonusFromCR`. Absent: `dexMod` only. `advantageMode: "adv"` → set `initAdvantage: true`. |
| `str/dex/con/int/wis/cha` | `abilities` | Nest into `{ str, dex, con, int, wis, cha }` |
| `save` | `savingThrows` | Parse `{ dex: "+11" }` → `[{ ability: "dex", bonus: 11 }]` |
| `skill` | `skills` | Parse to skill array format |
| `senses` | `senses` | Join array to string |
| `passive` | `passivePerception` | Direct number |
| `resist` | `damageResistances` | Join/format to string, capitalize. Handle conditional objects. |
| `immune` | `damageImmunities` | Same format as resist |
| `vulnerable` | `damageVulnerabilities` | Same format as resist |
| `conditionImmune` | `conditionImmunities` | Join, capitalize: `"Charmed, Frightened"` |
| `languages` | `languages` | Join array or `"None"` |
| `cr` or `cr.cr` | `cr` | Direct string. `{ cr: "13", xpLair: 11500 }` → `"13"` |
| `initiative.advantageMode` | `initAdvantage` | `"adv"` → `true`, otherwise `false` |
| `trait[]` | `features[]` | Map `{name, entries[]}` → `{name, desc, recharge, uses, usesMax}`. Flatten entries recursively + strip tags. Extract recharge/uses from name. |
| `action[]` | `attacks[]` + `features[]` + `multiattack` | **Complex** — see below |
| `bonus[]` | `features[]` (appended) | Bonus actions → features with `"(Bonus Action)"` appended to name |
| `reaction[]` | `features[]` (appended) | Reactions → features with `"(Reaction)"` appended to name |
| `spellcasting[]` | `features[]` (appended) | Convert each: name (strip recharge tag) → feature name, headerEntries → desc, extract recharge from name tag |
| `legendary[]` | `legendaryActions[]` | Map entries, extract cost from name pattern `"(Costs N Actions)"` |
| `legendaryActions` | `legendaryActionBudget` | Direct number, default 3 |
| `mythic[]` | `features[]` (appended) | Mythic actions → features with `"(Mythic)"` appended to name |
| — | `legendaryResistances` | Extract from traits if "Legendary Resistance" trait exists, parse `"(N/Day)"`. Ignore lair variant. |
| — | `tactics` | `""` (empty — not in 5etools data) |
| — | `groups` | `[]` |
| — | `critRange` | `20` (default) |

#### Proficiency Bonus from CR (for initiative calculation)

```
CR 0-4:   +2     CR 13-16: +5
CR 5-8:   +3     CR 17-20: +6
CR 9-12:  +4     CR 21-24: +7
                  CR 25-28: +8
                  CR 29+:   +9
```
Used only when `initiative.proficiency` is present. Fractional CRs (1/8, 1/4, 1/2) → proficiency +2.

#### Action Parsing (the complex part)

The `action[]` array mixes attacks, multiattack, and non-attack abilities. Two attack tag formats exist:

**2024 format** (XMM): `{@atkr m} {@hit 5}, reach 5 ft. {@h}12 ({@damage 2d8 + 3}) Piercing damage.`
**Older format** (homebrew/3pp): `{@atk mw} {@hit 4} to hit, reach 5 ft., one target. {@h}5 ({@damage 1d6 + 2}) piercing damage.`

Detection and parsing:

1. **Multiattack**: Name starts with "Multiattack" (may have suffix like "(Vampire Form Only)") → extract `entries[0]` text (stripped) → `template.multiattack`
2. **Attack** (either format): Entry text contains `{@atkr` OR `{@atk ` OR `{@hit` → parse as attack:
   - Attack bonus: extract from `{@hit N}` → number
   - Reach/range: extract from text like `"reach 10 ft."` or `"range 80/320 ft."` → `note`
   - Damage: extract from `{@damage NdN + N}` → `damages[]` entries with type from surrounding text (word after closing paren, e.g., "Piercing", "Necrotic")
   - Multiple damage riders: Pit Fiend Fiery Mace has `{@damage 4d6 + 8}) Force damage plus 21 ({@damage 6d6}) Fire damage`
   - Parenthetical in name: `"Bite (Wolf or Hybrid Form Only)"` → strip parens, put in attack `note`
3. **Save-based actions** (2024): Entry starts with `{@actSave con} {@dc 17}` (no `{@hit}`) → these are save-based "attacks". Vampire Bite uses this. Parse as attack: no attack bonus (save-based), extract DC and damage.
4. **Non-attack actions**: Everything else → append to `features[]` with `recharge` extracted from name if present

#### Implementation Steps

1. **`strip5eToolsTags(text)`** — regex to convert all `{@tag ...}` to plain text. Loop until no `{@` remains (nested tags). Handle special cases: `{@h}` → "Hit: ", `{@actSave X}` → "X saving throw", `{@atkr X}` / `{@atk X}` → "".
2. **`flattenEntries(entries)`** — recursively process entry arrays. Strings pass through. Objects with `type: "list"` / `type: "item"` get flattened. Returns single joined string.
3. **`parse5eToolsSize(sizeArr)`** — map size codes to full names, join multiples with "/"
4. **`parse5eToolsAc(acArr)`** — extract AC number and note (strip tags from `from[]`)
5. **`parse5eToolsSpeed(speedObj)`** — format speed object to string, include `alternate` conditions
6. **`parse5eToolsSaves(saveObj)`** — parse save bonuses to our format
7. **`parse5eToolsResist(arr)`** — handle simple strings and conditional objects
8. **`parse5eToolsCr(cr)`** — handle string, fraction, or object with `cr` field
9. **`profBonusFromCr(crStr)`** — compute proficiency bonus from CR string
10. **`parse5eToolsAction(actionArr)`** — split into multiattack / attacks / features. Handle both tag formats.
11. **`parse5eToolsLegendary(legArr)`** — extract costs, map entries
12. **`parse5eToolsSpellcasting(spellArr)`** — convert to features with recharge
13. **`import5etools(json)`** — orchestrator: detect single vs array, map each entry using above helpers. Also processes `bonus[]`, `reaction[]`, `spellcasting[]`.
14. **Update `doImportMonster()`** — accept `.json` files, detect format, route to converter
15. **Update Import Monster modal** — add `.json` to file accept attribute, add paste textarea, update help text

#### Edge Cases
- `hp.special` (scaling HP like "X plus Y per player") — store formula text, set hpMax to parsed base or 0
- Conditional resistances: `[{ resist: ["bludgeoning", "piercing"], note: "from nonmagical attacks" }]` — flatten with note (may be rare in 2024 data)
- CR as object: `{ cr: "13", xpLair: 11500 }` → use `cr` field only
- CR as fraction: `"1/4"`, `"1/2"`, `"1/8"` — pass through as string, use for proficiency lookup
- Missing `legendary` array but has legendary resistance trait — still parse LR count from trait text
- `"Legendary Resistance (3/Day, or 4/Day in Lair)"` — parse first number only, ignore lair variant
- Actions with `"(1/Round)"` or `"(Recharges after a Short or Long Rest)"` in name — extract to recharge/uses
- Action names with form restrictions: `"Bite (Wolf or Hybrid Form Only)"` — preserve in note, strip from attack name
- Structured entries (nested lists/items) — recursively flatten before stripping tags
- `ac[0].from` may contain 5etools tags: `"{@item leather armor|PHB}"` → strip to `"Leather Armor"`
- Multiple sizes for shapeshifters: `["S", "M"]` → `"Small/Medium"`
- `initiative.proficiency` absent in older/homebrew data — fall back to dex mod only
- `spellcasting` array entries with `displayAs: "legendary"` — still place in features (not LA), user decides where to use them
- `bonus[]` and `reaction[]` arrays — append to features with type annotation in name
- `mythic[]` — mythic boss actions (rare, e.g. Tiamat). Append to features with "(Mythic)" annotation.
- `speed` as plain integer (walk-only shorthand) or `"Varies"` — handle all three speed formats
- `speed.canHover` — append "(hover)" to fly speed string
- `ac` with `condition` field — different AC in different forms, take first/highest
- `ac` with `{ special }` — text-only AC, store as note
- Ability scores as `null` or `{ special }` — set to 10 (default) or 0, note the special text
- `passive` as string — rare edge case, parse to number or store as note
- `conditionImmune` with nested conditional format — same pattern as resist/immune
- `resist`/`immune` with `{ special }` entries — plain text, append to string
- `resist`/`immune` with `preNote` — text before the resistance list
- `initiative.advantageMode: "adv"` → set `initAdvantage: true`
- `initiative` as plain number → use as flat init bonus directly

### Phase 6 — CritterDB Importer
- Discovery: investigate CritterDB JSON export format
- Data mapping: map CritterDB fields to canonical template schema
- Pure function: `importCritterDB(json) → template[]`
- UI: integrate into Import Monster modal

### Phase 7 — Bestiary Builder Importer
- Discovery: investigate Bestiary Builder JSON export format
- Data mapping: map fields to canonical template schema
- Pure function: `importBestiaryBuilder(json) → template[]`

### Backlog (unscheduled)
- Surprise round handling
- Dynamic notes/riders with round expiry
- Template override highlighting in combat
- Damage log (collapsible panel)
- Campaign/encounter filtering (dropdown or chips in encounter list)

## Key Architecture Decisions

### Sparse Delta Pattern
Combat instances only store what differs from the template. Value resolution: `instance.overrides?.[field] ?? template[field]`. This keeps combat state small and allows template edits to propagate to future combats.

### Single-File, No Dependencies
Same as all PromptFerret tools. All CSS and JS inline in `index.html`. Works from `file://` and GitHub Pages. No build step.

### Event Handler Safety
Never interpolate user strings into inline `onclick`/`onchange` handlers. Use UUIDs, numeric indices, or hardcoded strings only. User input reads from DOM via `this.value`. See CLAUDE.md for full details.

### Roll Log Persistence
Roll logs are stored on combatant instances and saved to localStorage. They survive page refreshes, panel toggles, and re-renders. Cleared only when that combatant's turn starts again. This is intentional — DMs need to reference past rolls when players contest results or use reactions retroactively (Silvery Barbs, Shield, etc.).

### Combat State Machine
`state.combats` is an array of combat objects. `activeCombatId` (transient, not persisted) selects which combat is displayed. `getActiveCombat()` returns the selected combat or null. `round: 0` = initiative setup phase. `round: 1+` = active combat. Combat view dispatches: empty → combat list → party select → init setup → active combat.
