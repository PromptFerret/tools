# Encounter Manager — Implementation Plan

## Overview

D&D encounter manager and initiative tracker for DMs running combat. Single self-contained HTML file at `encounter/index.html`. No server, no accounts, no dependencies — localStorage persistence only.

Built for heroic/homebrew D&D — CR100 monsters, level 60 players, non-standard rules are expected use cases.

## Completed Phases

### Phase 1 — Foundation (done)
- HTML structure: header, nav tabs (Monsters/Parties/Encounters/Combat), status bar, footer
- CSS: full PromptFerret dark theme (`--bg: #0f0f17`, `--surface: #1a1a2e`, `--accent: #7c5cbf`)
- View switching with dirty check warnings
- Monster template CRUD (list with search, form with all fields including abilities, attacks with multiple damage types, features, legendary actions/resistances)
- Encounter CRUD (name, location, campaign, notes, monster picker with qty)
- Party CRUD with shared player roster (toggle chips, array index handlers for safety)
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

## Phase 3 — Combat Features (next)

### Condition/Effect Tracking
- Condition chips on combatant rows (visible without expanding)
- Add condition: pick from standard list (Blinded, Charmed, Deafened, Frightened, Grappled, Incapacitated, Invisible, Paralyzed, Petrified, Poisoned, Prone, Restrained, Stunned, Unconscious) or custom text
- Each condition: `{ name, duration, durationType: 'rounds'|'minutes'|'indefinite', source }`
- Auto-decrement duration on turn start, notify when expired
- Store on combatant instance: `conditions[]`

### Concentration Tracking
- Toggle per combatant (players and monsters)
- Visual indicator on combatant row
- Warning when a concentrating combatant takes damage ("CON save DC X needed")
- Auto-clear previous concentration when new one applied

### Legendary Actions
- Budget display: "LA: 2/3" on monster row or detail panel
- Use button per legendary action in detail panel
- Auto-recharge at start of monster's turn (configurable — some DMs use recharge rolls for homebrew)
- Track `legendaryActionsRemaining` on combatant instance

### Legendary Resistances
- Budget display: "LR: 1/3"
- Use button with confirmation
- Track `legendaryResistancesRemaining` on combatant instance

### Feature Use Tracking & Recharge
- Features with `usesMax` show "Uses: 1/3" with use/restore buttons
- Features with `recharge` (e.g., "Recharge 5-6") prompt a recharge roll at start of monster's turn
- Recharge roll: 1d6, if >= recharge threshold, feature recharges — show result in roll log

### Crit Range on Template Schema
- Add `critRange: 20` (default) to monster template schema
- Template form field for crit range
- Attack rolling checks `d20 >= critRange` instead of `d20 === 20`

### Full Feature/Trait Display in Combat
- Show full feature description text in combat detail panel (currently only shows name/recharge/uses)
- Include legendary actions with their full text
- Detect dice notation patterns (NdN, NdN+M, NdN-M) in feature descriptions
- Make them clickable — clicking rolls the dice and shows result inline or in roll log
- Primary use case: healing features like "regains 1d6 hit points"

### Ability Check/Save Rolling for Monsters
- Quick-roll buttons in stats section of detail panel
- Roll 1d20 + ability modifier (or saving throw bonus if proficient)
- Results go to roll log

### Other Phase 3 Items
- Surprise round handling (surprised combatants skip first turn)
- Dynamic notes/riders with round expiry (e.g., "Booming Blade: 5d8 thunder if moves", expires in 1 round)
- Template override highlighting — UI marks overridden structural fields (AC, abilities, etc.) in distinct color
- Damage log (collapsible panel showing all damage dealt this combat)

### Phase 3 Data Changes
New fields on combatant instances (sparse, added on mutation only):
```json
{
  "conditions": [{ "name": "Frightened", "duration": 3, "durationType": "rounds", "source": "Dragon Fear" }],
  "concentration": false,
  "legendaryActionsRemaining": 3,
  "legendaryResistancesRemaining": 3,
  "featureUses": { "featureIndex": 2 },
  "surprised": false
}
```
New field on monster template:
```json
{
  "critRange": 20
}
```

## Phase 4 — Organization, Multi-Combat & Import/Export

### Multi-Combat Support
- `state.combat` (single object) becomes `state.combats` (array)
- `pf_enc_combat` localStorage key holds the array
- Combat tab landing page: list of active combats with name, round, party, combatant count
- Click to resume any combat
- "No active combats" empty state with link to Encounters
- Use case: split-party sessions, multiple groups with sessions between them
- Estimated ~30-50 lines of structural change + combat list UI
- All combat logic stays the same — operates on whichever combat is "selected"

### Monster Groups
- Groups CRUD: `{ id, name, templateIds[] }`
- Group filtering in template list (dropdown or chips)
- Assign templates to groups from template form
- `pf_enc_groups` localStorage key

### Campaign/Encounter Organization
- Encounter filtering by campaign field
- Campaign dropdown or filter chips in encounter list

### Import/Export
- Importers are pure functions: `importCritterDB(json) → template[]`, `import5etools(json) → template[]`
- Each maps external format to canonical template schema
- Import sources:
  - CritterDB JSON
  - Bestiary Builder JSON
  - 5etools JSON
- Export format: SquishText-encoded (deflate-raw → base64 → CRC32 → header)
  - Exported blob is a valid SquishText payload
  - Users can paste into SquishText tool to inspect raw JSON
- Import accepts both SquishText blobs and raw JSON (auto-detect)
- Full backup export/import (all templates, encounters, parties, groups)
- Compression functions (`compress`, `decompress`, `crc32`) embedded directly — same single-file approach

### Storage
- Storage usage indicator in footer or settings area
- localStorage stays plain JSON (no compression for local persistence)

### Portal Page
- Update root `tools/index.html` to include Encounter Manager in the tool list

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
`round: 0` = initiative setup phase. `round: 1+` = active combat. Combat view dispatches to different renderers based on this. The pendingCombatEncounterId handles the party selection step before combat state exists.
