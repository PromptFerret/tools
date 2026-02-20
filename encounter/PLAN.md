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
