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
2. **Format detection**: `doImportMonster()` currently only handles `.squishtext` files. For Phase 5+, update it to also accept `.json` files. Detection order: try SquishText decompress first → if that fails, try JSON.parse → if valid JSON, detect which format (our native format has `version` + `templates`, 5etools has `monster` array or top-level 5etools fields like `str`/`dex`/`ac` as array).
3. **Smart merge**: Same ID-based dedup as existing import. Imported templates get new `uuid()` IDs (since external formats don't use our IDs).
4. **Status reporting**: Show count of imported vs skipped (by name dedup for external formats since they have no IDs — dedup by name to avoid importing the same monster twice).

#### 5etools JSON Format (from user-exported data)

A 5etools export is either:
- A wrapper object: `{ "monster": [ ...entries... ] }`
- A single monster object (top-level, has `name`, `str`, `ac` as array, etc.)

**Detection**: If parsed JSON has a `monster` array → 5etools multi. If it has `str` (number) + `ac` (array) → 5etools single.

#### 5etools Monster Entry — Key Fields

```
name            string          "The World Ender"
source          string          "MonstersOfDrakkenheim" (ignore)
size            string[]        ["G"] — size codes: F/D/T/S/M/L/H/G
type            string|object   "aberration" or { type: "aberration", tags: ["Epic Boss"] }
ac              array           [23] or [{ ac: 15, from: ["natural armor"] }]
hp              object          { average: 60, formula: "8d8+24" } or { special: "..." }
speed           object          { walk: 60, fly: 80, swim: 30, burrow: 20, climb: 20 }
str/dex/con/int/wis/cha  number Top-level ability scores (not nested)
save            object|null     { dex: "+11", int: "+10" } — string values with +/-
skill           object|null     { athletics: "+19", perception: "+19" }
senses          string[]        ["Truesight 120 ft."]
passive         number          29 (passive perception)
resist          array|null      ["acid", "cold"] or [{ resist: [...], note: "..." }]
immune          array|null      damage immunities (same format as resist)
vulnerable      array|null      damage vulnerabilities (same format)
conditionImmune array|null      ["charmed", "frightened"]
languages       array|null      ["Common", "Draconic"]
cr              string|object   "30" or "1/2" or { cr: "10", lair: "11" }
trait           array|null      [{ name, entries[] }] — passive features
action          array|null      [{ name, entries[] }] — actions (attacks mixed in)
legendary       array|null      [{ name, entries[] }] — legendary actions
legendaryActions number|null    LA budget (default 3 if legendary array exists)
legendaryHeader  string[]|null  custom legendary action intro text
```

#### 5etools Rich Text Tags (must strip/convert)

Entries use `{@tag content}` markup. Key tags to handle:

```
{@damage 5d10 + 10}              → "5d10 + 10"
{@dice 2d6}                      → "2d6"
{@hit 19}                        → "+19"
{@dc 27}                         → "DC 27"
{@h}                             → "Hit: " (marks hit description)
{@recharge 5}                    → "(Recharge 5-6)" — extract for recharge field
{@recharge}                      → "(Recharge 6)"
{@condition Grappled|XPHB}       → "Grappled" (take first part before |)
{@spell Wish|XPHB}               → "Wish"
{@creature Zombie|MM}            → "Zombie"
{@item Shield|XPHB}              → "Shield"
{@skill Perception}              → "Perception"
{@atkr m}                        → "" (melee attack indicator — use for detection)
{@atkr r}                        → "" (ranged attack indicator)
{@atkr ms}                       → "" (melee spell attack)
{@atkr rs}                       → "" (ranged spell attack)
{@actSave con}                   → "Constitution saving throw"
{@actSaveFail}                   → "Failure:"
{@actSaveSuccess}                → "Success:"
{@actSaveSuccessOrFail}          → "Success or Failure:"
{@variantrule Hit Points|XPHB}   → "Hit Points"
{@table Mutations|Source}         → "Mutations"
```

**General rule**: `{@tag content|source|...}` → take `content` (first segment before `|`). Exception: `{@h}` → "Hit: ", `{@recharge N}` → extract for recharge field.

Implement as `strip5eToolsTags(text)` — regex-based, handles nested tags.

#### Field Mapping: 5etools → Our Template

| 5etools | Our Template | Transform |
|---------|-------------|-----------|
| `name` | `name` | Direct |
| `size[0]` | `size` | Map code: F→Fine, D→Diminutive, T→Tiny, S→Small, M→Medium, L→Large, H→Huge, G→Gargantuan |
| `type` or `type.type` | `type` | Extract string, capitalize |
| `type.tags` | (discard) | Tags like "Epic Boss" — no field for this |
| `ac[0]` or `ac[0].ac` | `ac` | Number extraction |
| `ac[0].from` | `acNote` | Join array: `"natural armor"` → `"Natural Armor"` |
| `hp.average` | `hpMax` | Direct number |
| `hp.formula` | `hpFormula` | Direct string (already in NdN+M format) |
| `hp.special` | `hpFormula` (as note) | Store in hpFormula, set hpMax to 0 or parse if possible |
| `speed` | `speed` | Format: `"60 ft., fly 80 ft., swim 30 ft."` |
| `str/dex/con/int/wis/cha` | `abilities` | Nest into `{ str, dex, con, int, wis, cha }` |
| `save` | `savingThrows` | Parse `{ dex: "+11" }` → `[{ ability: "dex", bonus: 11 }]` |
| `skill` | `skills` | Parse to skill array format |
| `senses` | `senses` | Join array to string |
| `passive` | `passivePerception` | Direct number |
| `resist` | `damageResistances` | Join/format to string, capitalize |
| `immune` | `damageImmunities` | Join/format to string, capitalize |
| `vulnerable` | `damageVulnerabilities` | Join/format to string, capitalize |
| `conditionImmune` | `conditionImmunities` | Join, capitalize: `"Charmed, Frightened"` |
| `languages` | `languages` | Join array or `"None"` |
| `cr` or `cr.cr` | `cr` | Direct string |
| `dex` mod | `initBonus` | `Math.floor((dex - 10) / 2)` |
| — | `initAdvantage` | `false` (default) |
| `trait[]` | `features[]` | Map `{name, entries[]}` → `{name, desc, recharge, uses, usesMax}`. Desc = join entries + strip tags. Extract recharge from name if present. |
| `action[]` | `attacks[]` + `features[]` + `multiattack` | **Complex** — see below |
| `legendary[]` | `legendaryActions[]` | Map entries, extract cost from name pattern `"(Costs N Actions)"` |
| `legendaryActions` | `legendaryActionBudget` | Direct number, default 3 |
| — | `legendaryResistances` | Extract from traits if "Legendary Resistance" trait exists, parse `"(N/Day)"` |
| — | `tactics` | `""` (empty — not in 5etools data) |
| — | `groups` | `[]` |
| — | `critRange` | `20` (default) |

#### Action Parsing (the complex part)

The `action[]` array mixes attacks, multiattack, and non-attack abilities. Detection:

1. **Multiattack**: Action named "Multiattack" → extract `entries[0]` text (stripped) → `template.multiattack`
2. **Melee/Ranged attack**: Entry text contains `{@atkr m}` or `{@atkr r}` or `{@hit N}` → parse as attack:
   - Attack bonus: extract from `{@hit N}` → number
   - Reach/range: extract from text like `"reach 15 ft."` or `"range 30/120 ft."` → `note`
   - Damage: extract from `{@damage NdN + N}` → `damages[]` entries with type from surrounding text
   - Multiple damage riders: some attacks have multiple `{@damage}` tags (e.g., slashing + fire)
3. **Non-attack actions**: Everything else → append to `features[]` with `recharge` extracted from name if present (e.g., `"Bellowing Roar (Recharges after a Short or Long Rest)"` → recharge as note, or `{@recharge 5}` → recharge `"5-6"`)

#### Implementation Steps

1. **`strip5eToolsTags(text)`** — regex to convert all `{@tag ...}` to plain text. Handle nested tags.
2. **`parse5eToolsSize(sizeArr)`** — map size codes to full names
3. **`parse5eToolsAc(acArr)`** — extract AC number and note
4. **`parse5eToolsSpeed(speedObj)`** — format speed object to string
5. **`parse5eToolsSaves(saveObj)`** — parse save bonuses to our format
6. **`parse5eToolsResist(arr)`** — handle simple strings and conditional objects
7. **`parse5eToolsAction(actionArr)`** — split into multiattack / attacks / features
8. **`parse5eToolsLegendary(legArr)`** — extract costs, map entries
9. **`import5etools(json)`** — orchestrator: detect single vs array, map each entry using above helpers
10. **Update `doImportMonster()`** — accept `.json` files, detect format, route to converter
11. **Update Import Monster modal** — add `.json` to file accept attribute, update help text

#### Edge Cases
- `hp.special` (scaling HP like "X plus Y per player") — store formula text, set hpMax to parsed base or 0
- Conditional resistances: `[{ resist: ["bludgeoning", "piercing"], note: "from nonmagical attacks" }]` — flatten with note
- CR as object: `{ cr: "10", lair: "11" }` — use `cr` field
- Missing `legendary` array but has legendary resistance trait — still parse LR count from trait text
- Actions with `"(1/Round)"` or `"(Recharges after a Short or Long Rest)"` in name — extract to recharge/uses
- Entries that are arrays of strings (join with newline before stripping tags)

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
