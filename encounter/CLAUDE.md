# Encounter Manager

## What This Is
A single self-contained HTML file (`index.html`) for D&D encounter management and initiative tracking. DMs build monster templates, organize parties, construct encounters, and run combat — all in the browser with IndexedDB persistence (auto-migrates from localStorage, falls back to localStorage if IndexedDB unavailable). No server, no accounts, no dependencies.

## Code Map

The file is ~3200+ lines. Rough section layout:

| Section | Contents |
|---------|----------|
| CSS | Full theme, button classes, searchable select, modal overlay, combat list cards, condition/concentration chips, LA/LR badges, turn-notify pulse, responsive breakpoints |
| HTML | Header, toolbar (nav tabs + Import/Export buttons), status bar, 4 view containers, footer with storage indicator |
| Constants & State | `ABILITIES`, `CONDITIONS`, `SKILLS`, `SIZES`, `STORAGE_KEYS`, `state`, `activeCombatId`, `getActiveCombat()`, `load()`/`save()`, `uuid()`, `esc()`, `modStr()` |
| Dice Engine | `rollDice(notation, opts)`, `renderDiceText()`, `rollInlineDice()` |
| Compression | CRC32, `compress()`, `decompress()` — SquishText-compatible, embedded for import/export |
| Combat Helpers | `saveCombat()`, `getTemplate()`, `resolveField()` |
| View Management | `switchView()`, `handleNew()`, `render()` dispatcher |
| Monster Templates | List, form, CRUD, passive perception, crits-on field |
| Encounters | List, form, CRUD, searchable monster picker, delete (clears linked combats) |
| Parties | List, form, CRUD, player roster chips |
| Combat Entry | `startCombat()`, party select cards, `launchCombat()` |
| Combat List | `renderCombatList()`, `resumeCombat()` — multi-combat selector |
| Initiative Setup | `renderInitiativeSetup()`, `beginCombat()`, `addAdhocCombatant()` |
| Active Combat | `renderActiveCombat()`, `renderCombatantDetail()` (with conditions, concentration, legendary sections) |
| Condition Tracking | `updateConditionFields()`, `addCondition()`, `removeCondition()` |
| Concentration | `setConcentration()`, `dropConcentration()` |
| HP Tracking | `applyDamage()` (with concentration DC warning), `applyHeal()`, `applyTempHp()` |
| Feature Use Tracking | `useFeature()`, `restoreFeature()`, `rollRecharge()` |
| Legendary Actions | `useLegendaryAction()`, `useLegendaryResistance()`, `restoreLegendaryResistance()`, `restoreLegendaryAction()` |
| Ability Check/Save | `rollAbilityCheck()`, `rollSavingThrow()` |
| Attack Rolling | `rollAttack()`, `rollDamageForAttack()`, `renderRollLog()` |
| Turn Management | `applyTurnStartEffects()`, `nextTurn()`, `prevTurn()`, `toggleReaction()` |
| Mid-Combat Editing | `editInit()`, `killCombatant()`, `reviveCombatant()`, `removeCombatant()`, `endCombat()` |
| Import/Export | `exportData()`, `showImportDialog()`, `doImport()` |
| Storage Indicator | `updateStorageInfo()` |
| Init | `load().then(() => { render(); updateStorageInfo(); })` |

*Line numbers are approximate and shift as code is added.*

## Architecture

### Views
Four top-level views, switched via nav tabs with show/hide divs:

| View | Purpose |
|------|---------|
| **Monsters** | CRUD for reusable monster templates |
| **Parties** | Manage player rosters (reusable across encounters) |
| **Encounters** | Build encounters from templates (monsters only, no players) |
| **Combat** | Initiative tracking, rolling, HP/condition management |

### Render Dispatcher
The main `render()` function checks which view is active and delegates:
- Templates/Encounters/Parties: calls `renderTemplateList()` or `renderTemplateForm()` (etc.) based on whether `editingId` is set
- Combat: calls `renderCombatView()`, which further dispatches based on combat sub-state (empty → party select → init setup → active combat)

### Form Editing Pattern
Forms use module-level variables:
- `editingId` — UUID of the item being edited (null = creating new)
- `formData` — current form state object (mutated by UI handlers)
- `savedFormSnapshot` — JSON snapshot taken at form open/save for dirty detection
- Dirty check: `JSON.stringify(formData) !== JSON.stringify(savedFormSnapshot)`

**Important**: This pattern is used for template/encounter/party forms ONLY. Combat edits are direct mutations on `state.combat.combatants[]` + `saveCombat()`. Do not use `formData` inside combat code.

### Form Pattern
All forms (template, party, encounter) follow Save/Close/Cancel with dirty tracking:
- **Save**: persists to localStorage, stays in form, flashes status bar green
- **Close**: returns to list view (warns if dirty)
- **Cancel**: discards changes (warns if dirty)

### State Management
```javascript
let state = { templates: [], groups: [], encounters: [], parties: [], combats: [], players: [] };
let activeCombatId = null; // transient, not persisted — ID of the combat being viewed
```
- `state.combats` is an array of combat objects (supports multiple simultaneous combats)
- `getActiveCombat()` returns the combat matching `activeCombatId`, or null
- `state.players` is a shared roster — names reusable across any party via toggle chips
- `state.parties` own their own `players[]` arrays (copies, not references)
- `state.encounters` contain only monsters (no players) — party is selected at combat start
- Every mutation triggers `save(key)` immediately (fire-and-forget async IndexedDB write; in-memory `state` is the source of truth)

### Storage (IndexedDB)
- **Database**: `pf_encounter`, version 1, single object store `state`
- **Keys** (same `pf_enc_` namespace): `pf_enc_templates`, `pf_enc_groups`, `pf_enc_encounters`, `pf_enc_parties`, `pf_enc_combats`, `pf_enc_players`
- **`load()`** is async — reads from IndexedDB, auto-migrates from localStorage on first run (copies data, then clears localStorage)
- **`save(key)`** is fire-and-forget — kicks off IndexedDB write without awaiting, callers don't need to change
- **Fallback**: if IndexedDB is unavailable (`file://` + Safari edge cases), falls back to localStorage silently via `_useLocalStorage` flag
- **Init**: `load().then(render)` — async load completes before first render
- **Capacity**: ~50MB+ vs localStorage's ~5-10MB

### Data Schemas

**Monster Template** — canonical definition with all stats, attacks, features, legendary actions. Key fields: `abilities` (object with str/dex/con/int/wis/cha), `attacks[]` (each with `damages[]` array of `{dice, type, note}`), `features[]` (with optional `recharge` and `uses/usesMax`), `passivePerception` (null = auto-calc from WIS mod or Perception skill), `critRange` (default 20, customizable for homebrew).

**Encounter** — `{ id, name, location, campaign, notes, monsters: [{ templateId, qty }] }`. No players — party selected at combat time.

**Party** — `{ id, name, players: ["name1", "name2"] }`. Players are strings, not objects.

**Combat State** — `{ id, name, encounterId, partyId, round, turnIndex, combatants[], damageLog[], active }`. `round: 0` = initiative setup phase. `round: 1+` = active combat. Multiple combats can exist simultaneously in `state.combats[]`.

**Combatant Instance** — sparse delta pattern. Instances only store overrides from template defaults. Value resolution: `instance.overrides?.[field] ?? template[field]`.

Monster: `{ id, templateId, name, type:'monster', init, initRollText, currentHp, rollLog[] }`
Player: `{ id, name, type:'player', init, ac }` — init entered manually, AC editable in detail panel.
Ad-hoc: `{ id, name, type:'adhoc', init, notes }` — lair actions, environment effects.

Fields added on mutation only: `tempHp`, `overrides`, `reactionUsed`, `notes`, `dead`, `rollLog[]`, `_expanded` (transient UI flag), `featureUses` (sparse map: featureIndex → remaining), `legendaryActionsRemaining`, `legendaryResistancesRemaining`, `conditions[]`, `concentration` (string or null).

**Condition** — `{ name, durationType, duration?, saveDC?, saveAbility?, source? }`. Three types:
- `durationType: 'rounds'` — has `duration` (integer), auto-decrements on turn start, auto-expires at 0
- `durationType: 'save'` — has `saveDC` + `saveAbility`, system reminds DM at turn start, DM removes manually
- `durationType: 'indefinite'` — no auto-expiry, DM removes manually

### Combat Flow
1. DM clicks "Combat" on an encounter
2. App validates monsters exist and at least one party is saved
3. `pendingCombatEncounterId` is set (transient variable, not persisted) — bridges encounter list to combat view
4. Combat view shows party selection cards
5. DM picks a party → `launchCombat(partyId)` creates a new combat in `state.combats[]`, sets `activeCombatId`, clears `pendingCombatEncounterId`
6. Initiative setup: monsters pre-rolled (with roll breakdown shown), players blank
7. DM enters player inits, clicks "Begin Combat" → sorts descending, round 1 starts
8. Active combat: expandable combatant rows with HP, attacks, roll log, conditions, concentration

### Combat Sub-States

| State | Condition | Renders |
|-------|-----------|---------|
| Empty | No combats, no `pendingCombatEncounterId` | "Go to Encounters..." message |
| Party Select | `pendingCombatEncounterId` set | Party selection cards |
| Combat List | `state.combats.length > 0`, no `activeCombatId` | Cards for each combat (click to resume) |
| Initiative Setup | Active combat, `round === 0` | Editable init list + "Begin Combat" |
| Active Combat | Active combat, `round >= 1` | Turn tracker with expandable rows |

## Key Functions

### Dice Engine
`rollDice(notation, opts)` — parses `NdN+M` notation, returns `{ rolls, modifier, total, text }`. Supports advantage/disadvantage for 1d20 rolls. Never interprets results (no "hit"/"miss").

### Inline Dice
- `renderDiceText(text, combatantIdx, sourceType, sourceIdx)` — scans text for dice notation patterns (`NdN+M`), returns HTML with clickable `.dice-link` spans. `sourceType` (`'feature'` or `'la'`) and `sourceIdx` are optional — when provided, roll log entries include the feature/LA name.
- `rollInlineDice(combatantIdx, notation, sourceType, sourceIdx)` — rolls dice from inline text, resolves source name from template, appends to rollLog with label like "Fire Breath: 2d6+3", live-updates via `renderRollLog()`

### Attack Rolling
- `rollAttack(cIdx, aIdx, mode)` — mode: undefined (normal), `'advantage'`, `'disadvantage'`. Crit detection uses `template.critRange` (default 20). Appends to combatant's `rollLog[]`.
- `rollDamageForAttack(cIdx, aIdx, crit)` — crit doubles dice count (NdM → 2NdM), not modifiers. Appends to `rollLog[]`.
- `renderRollLog(cIdx)` — live-updates the roll log DOM without full re-render.

### Roll Log
- Stored on each combatant as `rollLog[]` array, persisted to localStorage.
- Entries stack chronologically — attacks, damage, crits, inline dice, recharge rolls, condition notifications all accumulate.
- Cleared at the start of that combatant's next turn (in `applyTurnStartEffects()`).
- Survives page refreshes, panel close/open, re-renders.

### HP Tracking
- Single input + 3 buttons: Dmg (red), Heal (green), THP (yellow).
- `applyDamage(idx)` — THP absorbed first, overflow to HP, auto-mark dead at 0. Warns about concentration save DC if concentrating.
- `applyHeal(idx)` — cap at maxHp, auto-revive if healed above 0.
- `applyTempHp(idx)` — THP doesn't stack, takes higher value (D&D rules).

### Initiative Values
- Init can be a number (including 0) or `null` (not yet set). These are distinct states — `0` displays as "0", `null` displays as empty input.
- All init comparisons use `c.init !== null && c.init !== undefined` (never `!c.init`, which treats 0 as falsy).
- All sorting uses `(c.init ?? -Infinity)` so null-init combatants sort to the bottom.
- `setCombatantInit()` and `editInit()` convert empty string input to `null`, numeric input to number.
- `beginCombat()` validates that all players have non-null init before starting.

### Init Editing
- `editInit(idx, value)` — if current combatant drops init, their turn ends and combat advances to next (via `applyTurnStartEffects()`). Non-current combatant edits preserve the current turn.

### Turn-Start Logic
`applyTurnStartEffects(idx)` is the consolidated turn-start hook, called from both `nextTurn()` and `editInit()` (turn-advance branch). It:
- Resets `reactionUsed` on the combatant
- Clears `rollLog`
- Adds recharge prompts for depleted features with `recharge` (DM-initiated — notification only, DM clicks Roll Recharge in detail panel)
- Recharges legendary actions to full budget
- Decrements condition round durations (auto-expires at 0 with notification)
- Adds save reminders for save-based conditions

### Feature Use Tracking
- `useFeature(cIdx, fIdx)` / `restoreFeature(cIdx, fIdx)` — increment/decrement feature uses, stored in `c.featureUses` sparse map
- `rollRecharge(cIdx, fIdx)` — DM-initiated: rolls 1d6, if >= threshold restores uses. Result shown in roll log.

### Legendary Actions & Resistances
- `useLegendaryAction(cIdx, laIdx)` — deducts cost from `c.legendaryActionsRemaining`, logs LA name to rollLog, disables button if insufficient budget
- `useLegendaryResistance(cIdx)` — decrements `c.legendaryResistancesRemaining`, logs to rollLog
- `restoreLegendaryAction(cIdx)` — increments LA remaining (capped at budget), for DM take-backs
- `restoreLegendaryResistance(cIdx)` — increments LR remaining (capped at max), for DM take-backs
- LA/LR badges shown on combatant rows (e.g., "LA:2/3 LR:1/3")
- LA budget auto-recharges at start of monster's turn
- Restore LA button in section header, Restore LR button next to Use LR

### Ability Check/Save Rolling
- `rollAbilityCheck(cIdx, ability)` — rolls 1d20 + ability modifier, appends to rollLog
- `rollSavingThrow(cIdx, ability)` — rolls 1d20 + save bonus (proficient if in template.savingThrows) or ability modifier, appends to rollLog

### Condition Tracking
- Conditions work on ALL combatant types (monsters, players, ad-hoc)
- Standard D&D conditions list + custom text option for non-standard effects
- Three duration types: rounds (auto-decrement), save-based (DM reminder), indefinite (manual remove)
- Condition chips visible on combatant rows without expanding
- `addCondition(idx)` reads from inline form inputs (select + duration type dropdown + fields)
- `removeCondition(idx, condIdx)` — splices out with × button on chip

### Concentration
- Works on monsters and players (not ad-hoc)
- `setConcentration(idx, spellName)` / `dropConcentration(idx)`
- Concentration chip shown on combatant row
- `applyDamage()` warns with CON save DC when concentrating combatant takes damage (DC = max(10, damage/2))

### Passive Perception
- `calcPassivePerception(t)` — 10 + Perception skill bonus (or WIS mod if no Perception skill)
- `getPassivePerception(t)` — returns manual override if set, otherwise auto-calc
- `updatePPDisplay()` — refreshes PP display when WIS score changes mid-edit

### HTML Escaping
`esc(s)` escapes `& < > "` for safe HTML rendering. **Never put user-provided strings into inline onclick handlers** — use array indices or UUIDs instead. The apostrophe bug (e.g., "D'Amore" breaking `onclick="fn('D'Amore')"`) is why all roster chip handlers use numeric indices.

## Inline Event Handler Safety

All `onclick`/`onchange` handlers that interpolate variables use ONLY:
- **UUIDs** (`t.id`, `e.id`, `p.id`) — safe, no special chars
- **Numeric indices** (`i`, `j`, `idx`) — safe
- **Hardcoded strings** (`'str'`, `'advantage'`, etc.) — safe

User input flows through `this.value` in onchange handlers (reads from DOM element, never interpolated as string literal). This pattern must be maintained for any new UI that references user data in event handlers.

**Exception for inline dice**: `renderDiceText()` interpolates dice notation strings (e.g., `'2d6+3'`) into onclick handlers. This is safe because the notation comes from `esc()`-escaped template data and the regex only matches `\d+d\d+([+-]\d+)?` patterns.

## Encounter/Combat Lifecycle
- Deleting an encounter removes all combats linked to that encounter from `state.combats`.
- Multi-combat supported: `state.combats` is an array, `activeCombatId` selects which one is displayed.
- Combat list view shows all active combats when none is selected.
- "Back to List" button appears in the combat bar when multiple combats exist.

## Testing
- No build step, no test suite. Open `index.html` in a browser.
- Inspect IndexedDB in DevTools: Application > IndexedDB > `pf_encounter` > `state`
- Check storage capacity: `navigator.storage.estimate()` in console
- Works from `file://` — no server needed (falls back to localStorage if IndexedDB unavailable).
- Works on GitHub Pages at `https://promptferret.github.io/tools/encounter/`

## Phased Build Status

- **Phase 1** (done): Monster CRUD, party CRUD, encounter CRUD, dice engine, player roster, form dirty tracking, status flash, PP auto-calc
- **Phase 2** (done): Combat core — initiative rolling with roll text display, turn management, HP tracking (single input + Dmg/Heal/THP), attack rolling (normal/adv/dis), crit damage rolling, crit/fumble highlighting (nat 20/nat 1), stacking roll log (persisted, clears on turn start), expandable combatant rows with column headers, ad-hoc combatants (sorted on add), reaction toggle, player AC, multiattack styling, init drop ends turn
- **Phase 3** (done): Crit range on template (customizable crit threshold, labeled "Crits On"), full feature/trait text display in combat with inline clickable dice rolling (roll log labels include source feature/LA name), feature use tracking with DM-initiated recharge rolls, legendary actions (budget tracking, use/restore buttons, auto-recharge on turn) and resistances (use/restore counter), ability check/save rolling from detail panel, condition/effect tracking (rounds/save-based/indefinite, all combatant types, custom effects, condition chips on rows, auto-decrement), concentration tracking with damage DC warnings, turn-start notification badges on combatant rows
- **Phase 3.5** (done): IndexedDB migration (auto-migrates from localStorage, localStorage fallback), no-cache meta tags, init 0 display fix
- **Phase 4** (done): Multi-combat support (combats array + combat list selector UI), SquishText import/export (file-based backup/restore with embedded compression), searchable monster picker in encounter form, storage indicator in footer
- **Phase 5+** (future): External importers — CritterDB, 5etools, Bestiary Builder (each in own phase for discovery/mapping). See `PLAN.md` for backlog.

## Development Notes

- Ability score inputs use inline DOM updates (`document.getElementById('mod_X').textContent = ...`) to preserve tab order — a full re-render would steal focus
- Feature and legendary action sections show column headers (`dyn-header`) only when rows exist
- Attack damage rows are nested inside attack cards with a left border indent
- Roll log uses `renderRollLog()` for live DOM updates without full re-render when rolling
- Condition form uses `updateConditionFields()` for dynamic field switching based on duration type dropdown
- `applyTurnStartEffects()` is the single consolidated turn-start hook — all turn-start additions go here, not in `nextTurn()` or `editInit()` directly
- Recharge rolls are DM-initiated (notification + button), NOT auto-rolled on turn start
- **Button classes**: Use `.btn-accent`, `.btn-success`, `.btn-warning`, `.btn-danger` for colored buttons — never inline `style="color:var(--accent)"` as it breaks hover contrast. Each class has proper `:hover` with `color: white !important`. `.btn-remove` is the global class for × delete/remove buttons (red border, red text, red bg on hover).
- **Multi-combat**: All combat functions use `getActiveCombat()` instead of `state.combat`. `activeCombatId` is transient (not persisted). `saveCombat()` saves the entire `state.combats` array. Migration from old single-combat format (`pf_enc_combat`) is automatic in `load()`.
- **Searchable monster picker**: Encounter form uses `filterMonsterPicker()` + `selectMonster()` with a `.search-select` dropdown. Monster ID stored in `data-template-id` on the input element.
- **Import/Export**: Uses embedded SquishText-compatible compression (deflate-raw + CRC32). Export = file download. Import = file upload modal. Accepts both SquishText blobs and raw JSON.
- No backwards compatibility concerns yet — tool is pre-release
- Context: heroic/homebrew D&D — CR100 monsters, level 60 players, non-standard rules are expected
