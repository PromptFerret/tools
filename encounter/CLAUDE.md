# Encounter Manager

## What This Is
A single self-contained HTML file (`index.html`) for D&D encounter management and initiative tracking. DMs build monster templates, organize parties, construct encounters, and run combat — all in the browser with localStorage persistence. No server, no accounts, no dependencies.

## Code Map

The file is ~2300 lines. Rough section layout:

| Section | Lines (approx) | Contents |
|---------|----------------|----------|
| CSS | 20–600 | Full theme, all component styles, responsive breakpoints |
| HTML | 600–700 | Header, toolbar nav tabs, status bar, 4 view containers, footer |
| Constants & State | 700–755 | `ABILITIES`, `state`, `load()`/`save()`, `uuid()`, `esc()`, `modStr()` |
| Dice Engine | 755–785 | `rollDice(notation, opts)` |
| Combat Helpers | 785–805 | `saveCombat()`, `getTemplate()`, `resolveField()` |
| View Management | 805–830 | `switchView()`, `handleNew()`, `render()` dispatcher |
| Monster Templates | 830–1200 | List, form, CRUD, passive perception |
| Encounters | 1200–1410 | List, form, CRUD, delete (clears combat if linked) |
| Parties | 1410–1560 | List, form, CRUD, player roster chips |
| Combat Entry | 1560–1700 | `startCombat()`, party select cards, `launchCombat()` |
| Initiative Setup | 1700–1765 | `renderInitiativeSetup()`, `beginCombat()`, `addAdhocCombatant()` |
| Active Combat | 1765–1960 | `renderActiveCombat()`, `renderCombatantDetail()`, `toggleCombatantDetail()` |
| HP Tracking | 2000–2120 | `applyDamage()`, `applyHeal()`, `applyTempHp()` |
| Attack Rolling | 2120–2210 | `rollAttack()`, `rollDamageForAttack()`, `renderRollLog()` |
| Turn Management | 2210–2270 | `nextTurn()`, `prevTurn()`, `toggleReaction()` |
| Mid-Combat Editing | 2270–2340 | `editInit()`, `killCombatant()`, `reviveCombatant()`, `removeCombatant()`, `endCombat()` |
| Init | 2340–2345 | `load(); render();` |

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
let state = { templates: [], groups: [], encounters: [], parties: [], combat: null, players: [] };
```
- `state.players` is a shared roster — names reusable across any party via toggle chips
- `state.parties` own their own `players[]` arrays (copies, not references)
- `state.encounters` contain only monsters (no players) — party is selected at combat start
- Every mutation triggers `save(key)` immediately (synchronous localStorage write)

### localStorage Keys
All namespaced with `pf_enc_`:
- `pf_enc_templates` — monster template array
- `pf_enc_groups` — monster groups array (Phase 4)
- `pf_enc_encounters` — encounter array
- `pf_enc_parties` — party array
- `pf_enc_combat` — active combat state (null if none)
- `pf_enc_players` — shared player roster

### Data Schemas

**Monster Template** — canonical definition with all stats, attacks, features, legendary actions. Key fields: `abilities` (object with str/dex/con/int/wis/cha), `attacks[]` (each with `damages[]` array of `{dice, type, note}`), `features[]` (with optional `recharge` and `uses/usesMax`), `passivePerception` (null = auto-calc from WIS mod or Perception skill).

**Encounter** — `{ id, name, location, campaign, notes, monsters: [{ templateId, qty }] }`. No players — party selected at combat time.

**Party** — `{ id, name, players: ["name1", "name2"] }`. Players are strings, not objects.

**Combat State** — `{ encounterId, partyId, round, turnIndex, combatants[], damageLog[], active }`. `round: 0` = initiative setup phase. `round: 1+` = active combat.

**Combatant Instance** — sparse delta pattern. Instances only store overrides from template defaults. Value resolution: `instance.overrides?.[field] ?? template[field]`.

Monster: `{ id, templateId, name, type:'monster', init, initRollText, currentHp, rollLog[] }`
Player: `{ id, name, type:'player', init, ac }` — init entered manually, AC editable in detail panel.
Ad-hoc: `{ id, name, type:'adhoc', init, notes }` — lair actions, environment effects.

Fields added on mutation only: `tempHp`, `overrides`, `reactionUsed`, `notes`, `dead`, `rollLog[]`, `_expanded` (transient UI flag).

### Combat Flow
1. DM clicks "Combat" on an encounter
2. App validates monsters exist and at least one party is saved
3. `pendingCombatEncounterId` is set (transient variable, not persisted) — bridges encounter list to combat view
4. Combat view shows party selection cards
5. DM picks a party → `launchCombat(partyId)` initializes `state.combat`, clears `pendingCombatEncounterId`
6. Initiative setup: monsters pre-rolled (with roll breakdown shown), players blank
7. DM enters player inits, clicks "Begin Combat" → sorts descending, round 1 starts
8. Active combat: expandable combatant rows with HP, attacks, roll log

### Combat Sub-States

| State | Condition | Renders |
|-------|-----------|---------|
| Empty | No `state.combat`, no `pendingCombatEncounterId` | "Go to Encounters..." message |
| Party Select | `pendingCombatEncounterId` set | Party selection cards |
| Initiative Setup | `state.combat.round === 0` | Editable init list + "Begin Combat" |
| Active Combat | `state.combat.round >= 1` | Turn tracker with expandable rows |

## Key Functions

### Dice Engine
`rollDice(notation, opts)` — parses `NdN+M` notation, returns `{ rolls, modifier, total, text }`. Supports advantage/disadvantage for 1d20 rolls. Never interprets results (no "hit"/"miss").

### Attack Rolling
- `rollAttack(cIdx, aIdx, mode)` — mode: undefined (normal), `'advantage'`, `'disadvantage'`. Appends to combatant's `rollLog[]`.
- `rollDamageForAttack(cIdx, aIdx, crit)` — crit doubles dice count (NdM → 2NdM), not modifiers. Appends to `rollLog[]`.
- `renderRollLog(cIdx)` — live-updates the roll log DOM without full re-render.

### Roll Log
- Stored on each combatant as `rollLog[]` array, persisted to localStorage.
- Entries stack chronologically — attacks, damage, crits all accumulate.
- Cleared at the start of that combatant's next turn (in `nextTurn()`).
- Survives page refreshes, panel close/open, re-renders.

### HP Tracking
- Single input + 3 buttons: Dmg (red), Heal (green), THP (yellow).
- `applyDamage(idx)` — THP absorbed first, overflow to HP, auto-mark dead at 0.
- `applyHeal(idx)` — cap at maxHp, auto-revive if healed above 0.
- `applyTempHp(idx)` — THP doesn't stack, takes higher value (D&D rules).

### Init Editing
- `editInit(idx, value)` — if current combatant drops init, their turn ends and combat advances to next. Non-current combatant edits preserve the current turn.

### Turn-Start Logic
`nextTurn()` is the main turn-start hook. Currently it:
- Advances to next living combatant (wraps → increments round)
- Resets `reactionUsed` on the new combatant
- Clears `rollLog` on the new combatant

**Phase 3 will add to this hook**: recharge prompts, condition duration decrement, legendary action recharge. Note that `editInit()` also has a turn-advance branch (when current combatant drops init) that mirrors this logic — both locations need Phase 3 additions.

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

## Encounter/Combat Lifecycle
- Deleting an encounter that has an active combat clears the combat state.
- Currently single-combat only (`state.combat` is one object or null). Multi-combat support (array of combats with selector UI) planned for Phase 4.

## Testing
- No build step, no test suite. Open `index.html` in a browser.
- Inspect localStorage in DevTools console: `JSON.stringify(localStorage)`
- Works from `file://` — no server needed.
- Works on GitHub Pages at `https://promptferret.github.io/tools/encounter/`

## Phased Build Status

- **Phase 1** (done): Monster CRUD, party CRUD, encounter CRUD, dice engine, player roster, form dirty tracking, status flash, PP auto-calc
- **Phase 2** (done): Combat core — initiative rolling with roll text display, turn management, HP tracking (single input + Dmg/Heal/THP), attack rolling (normal/adv/dis), crit damage rolling, crit/fumble highlighting (nat 20/nat 1), stacking roll log (persisted, clears on turn start), expandable combatant rows with column headers, ad-hoc combatants (sorted on add), reaction toggle, player AC, multiattack styling, init drop ends turn
- **Phase 3** (next): Conditions, concentration, legendary actions/resistances (with recharge rolls), feature use tracking + recharge prompts, full feature text display in combat with inline dice rolling, ability check/save rolling, crit range on template schema. See `PLAN.md` for full details.
- **Phase 4**: Monster groups, multi-combat support (combat array + selector UI for split sessions), import/export (CritterDB, 5etools, SquishText-encoded), storage indicator, update root index.html. See `PLAN.md` for full details.

## Development Notes

- Ability score inputs use inline DOM updates (`document.getElementById('mod_X').textContent = ...`) to preserve tab order — a full re-render would steal focus
- Feature and legendary action sections show column headers (`dyn-header`) only when rows exist
- Attack damage rows are nested inside attack cards with a left border indent
- Roll log uses `renderRollLog()` for live DOM updates without full re-render when rolling
- No backwards compatibility concerns yet — tool is pre-release
- Context: heroic/homebrew D&D — CR100 monsters, level 60 players, non-standard rules are expected
