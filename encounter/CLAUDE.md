# Encounter Manager

## What This Is
A single self-contained HTML file (`index.html`) for D&D encounter management and initiative tracking. DMs build monster templates, organize parties, construct encounters, and run combat — all in the browser with localStorage persistence. No server, no accounts, no dependencies.

## Architecture

### Views
Four top-level views, switched via nav tabs with show/hide divs:

| View | Purpose |
|------|---------|
| **Monsters** | CRUD for reusable monster templates |
| **Parties** | Manage player rosters (reusable across encounters) |
| **Encounters** | Build encounters from templates (monsters only, no players) |
| **Combat** | Initiative tracking, rolling, HP/condition management (Phase 2+) |

### Form Pattern
All forms (template, party, encounter) follow Save/Close/Cancel with dirty tracking:
- **Save**: persists to localStorage, stays in form, flashes status bar green
- **Close**: returns to list view (warns if dirty)
- **Cancel**: discards changes (warns if dirty)
- Dirty detection: `JSON.stringify(formData) !== JSON.stringify(savedFormSnapshot)`

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

**Combat Instance** (Phase 2) — sparse delta pattern. Instances only store overrides from template defaults. Value resolution: `instance.overrides?.[field] ?? template[field]`.

### Combat Flow (Phase 2)
1. DM clicks "Combat" on an encounter
2. App validates monsters exist and at least one party is saved
3. Combat view shows party selection cards
4. DM picks a party → `launchCombat(partyId)` initializes combat state

## Key Functions

### Dice Engine
`rollDice(notation, opts)` — parses `NdN+M` notation, returns `{ rolls, modifier, total, text }`. Supports advantage/disadvantage for 1d20 rolls. Never interprets results (no "hit"/"miss").

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
- **Hardcoded ability codes** (`'str'`, `'dex'`, etc.) — safe

User input flows through `this.value` in onchange handlers (reads from DOM element, never interpolated as string literal). This pattern must be maintained for any new UI that references user data in event handlers.

## Phased Build Status

- **Phase 1** (done): Monster CRUD, party CRUD, encounter CRUD, dice engine, player roster, form dirty tracking, status flash, PP auto-calc
- **Phase 2** (next): Combat core — initiative rolling, turn management, HP tracking, attack rolling, expandable combatant rows, ad-hoc combatants, reaction toggle
- **Phase 3**: Conditions, concentration, legendary actions/resistances, feature recharge, advantage/disadvantage, damage log
- **Phase 4**: Monster groups, import/export (CritterDB, 5etools, SquishText-encoded), storage indicator

## Development Notes

- Ability score inputs use inline DOM updates (`document.getElementById('mod_X').textContent = ...`) to preserve tab order — a full re-render would steal focus
- Feature and legendary action sections show column headers (`dyn-header`) only when rows exist
- Attack damage rows are nested inside attack cards with a left border indent
- No backwards compatibility concerns yet — tool is pre-release
