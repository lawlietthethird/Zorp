# ZORP — Technical Brief for Claude Code Sessions

## Game Overview

Single self-contained HTML file game. All game logic, CSS, and HTML live in `index.html`. No build step, no dependencies, no external files. Open in browser to play.

- Hosted on GitHub Pages from this repo.
- Target device: mobile portrait (with desktop `html{zoom:0.8}` at `@media(min-width:600px)` replicating 80% browser zoom).
- Run length: 6 wins to victory. Honor pool 13. Max ~11 days. Perfect run (6-0, honor 13) → FLAWLESS screen.
- Board: 3 front × 3 back = 6 slots max.

---

## Architecture

### Battle Event System

The battle engine uses a global event bus. `fireEvent(event)` dispatches to all non-broken items and to active skill handlers.

**Event payload:** `{type, item, side, value, source}`

**14 event types:** `BATTLE_START`, `FATIGUE_START`, `BATTLE_END`, `ITEM_ACTIVATED`, `ITEM_BROKEN`, `ITEM_HEALED`, `ITEM_SHIELDED`, `DAMAGE_DEALT`, `DAMAGE_RECEIVED`, `BURN_APPLIED`, `POISON_APPLIED`, `HASTE_APPLIED`, `SLOW_APPLIED`.

**`NEGATIVE_STATUSES` Set:** `BURN_APPLIED`, `POISON_APPLIED`, `SLOW_APPLIED`. Add new debuff event types here when created.

**ITEM_BROKEN fires BEFORE `item.broken=true`** — handlers can still act on the item while it exists.

**Re-entrant guard:** `firingEvent` flag prevents cascade beyond one level. Action functions (`applyDmgTo`, `applyShield`, `healItem`, `applyBurn`, `applyPoison`, `applyHaste`, `applySlow`) already fire their own events — never call `fireEvent` from inside an action function.

**Items use `handleEvent(event)`** for triggered effects. Passive multipliers stay in `getSkillMods()`. Skill handlers are registered as `sk._battleHandler` from `runBattle` (inline, not via `registerSkillHandlers` — that function has been removed).

**`getNeighbors(item)`** returns `{side, row, index, left, right, behind, inFront}`. `behind` is only set (non-null) for front-row items pointing to the back-row item at the same index. `inFront` is only set for back-row items pointing to the front-row item at the same index. Neither is set for the other row direction.
**`item._side`** is set in `resetItemBattleState` — use this to know which board an item belongs to inside handlers.

### Item System

`getItemFactory(key)` looks up an item factory across all pools: `ITEMS, TYPED_ITEMS, PACIFIST_ITEMS, HEADBUTT_ITEMS, BURN_ITEMS, WATER_ITEMS, ELECTRIC_ITEMS, STEEL_ITEMS, POISON_ITEMS, MOM_ITEMS, MOM_CONSOLATION_ITEMS`.

`getItemsOfRarity(rarity, typeFilter)` flattens the first 9 pools (not MOM pools) into one map and filters by rarity (and optional itemType filter). Falls back to a lower rarity if none match. Typed items are mixed in — typed players see the same shop pool as typeless.

`makeItem(key, name, icon, tag, maxHp, actMs, desc, effectFn, cost=0, uses=null, baseEffect=0, itemType='', itemRarity='')` is the constructor. Items also have a `sellOverride` field (initialized to `null`); if non-null, `getSellValue` returns it directly instead of the tier-based price.

### `effectFn` signature

Every item's effect function takes three arguments: `(side, amount, self)`.
- `side` — the **attacker's** side (`'player'` or `'enemy'`).
- `amount` — the item's current `effectAmt` (scales with upgrade tier).
- `self` — the item object itself (for self-targeting effects).

When calling damage functions from inside `effectFn`, pass `opp(side)` as the target side.

### `side` convention in damage functions

`side` always refers to the **target** side (where damage lands), not the attacker.
- `side === 'enemy'` → player is attacking → apply player skill mods.
- `side === 'player'` → enemy is attacking → no player mods.

The correct check is:
```js
const attackerMods = side === 'player' ? null : getSkillMods();
```

### Targeting Rules

| Effect type | Targets |
|---|---|
| `applyShield` | Lowest HP% ally with `maxHp > 0` (or specific `targetItem` if passed and `maxHp > 0`). |
| `applyShieldAll` | All alive allies with `maxHp > 0`. |
| `healItem` | Ally with lowest HP percentage (`hp < maxHp` filter). |
| `healAll` | All alive allies with `maxHp > 0`. |
| `dealDmg` | Leftmost front row enemy (or back if front empty). |
| `dealDmgRandom` | Random front row enemy (or back if front empty). |
| `dealDmgTwoRandom` | 2 random front-row enemies (falls back to back if front empty), can hit same target twice. |
| `dealDmgBackRow` | Random back row enemy (falls back to front if back empty). |
| `dealDmgWithStatus` | Random front enemy + applies status to that same target. |

Items with `maxHp === 0` are excluded from targeting pools automatically. Use tag `'Untargetable'` not `'Food'`. Untargetable items are also skipped by Arsonist's global burn effect.

### Board Structure

```js
board       // {front:[3], back:[3]} — player
enemyBoard  // {front:[3], back:[3]} — enemy
```

Slot expansion (`maxFrontSlots()` / `maxBackSlots()`):

| Day | Front | Back | Total |
|-----|-------|------|-------|
| 1 | 1 | 1 | 2 |
| 2 | 2 | 1 | 3 |
| 3 | 2 | 2 | 4 |
| 4 | 3 | 2 | 5 |
| 5+ | 3 | 3 | 6 |

Slots are numbered 1–6: front[0]=1, front[1]=2, front[2]=3, back[0]=4, back[1]=5, back[2]=6. Locked slots display their number with a 🔒 icon.

`G.neighborSlotBonus` adds extra slots beyond the day-gated cap (overflow goes to front first, then back, capped at hard max of 3 per row). `maxFrontSlots()` and `maxBackSlots()` both read this value.

### `applySlotBonuses(item, rowType)`

Called every time an item is placed onto the board (shop buy, event placement, swap paths, mom gift). Reads `G.slotBonuses[rowType]` and applies any accumulated `maxHp` or `damage` bonuses to the newly placed item. **Must be called on all placement paths** — omitting it silently skips permanent slot bonuses.

Guard: checks `item._slotBonusApplied` before applying and sets it to `true` immediately after. Prevents double-application on swap paths where the same item object may pass through placement code twice.

### `G.lockedSlots`

Array of `{row, idx}` objects. Slots in this list cannot be used for free placement. All placement loops must check `G.lockedSlots` before assigning. `pawnbroker_unlock` removes entries. Initialized to `[]` in `resetRun`.

---

## Binding System

Bindings link two board slots together so that battle events on one slot trigger effects on its partner.

### `BINDING_SLOT_PAIRS` constant

Five preset pairs. Only front-slot items can be slotA:

```js
const BINDING_SLOT_PAIRS=[
  {a:{row:'front',idx:0}, b:{row:'front',idx:1}},
  {a:{row:'front',idx:0}, b:{row:'back', idx:0}},
  {a:{row:'front',idx:1}, b:{row:'front',idx:2}},
  {a:{row:'front',idx:1}, b:{row:'back', idx:1}},
  {a:{row:'front',idx:2}, b:{row:'back', idx:2}},
];
```

### `G.bindings` array

Each binding object: `{id, type, slotA:{row,idx}, slotB:{row,idx}, visualIndex}`.

`visualIndex` is 1, 2, or 3 (assigned as `G.bindingCount+1` before the push). Controls the border CSS class and symbol shown on bound item cards.

| visualIndex | CSS class | Symbol |
|-------------|-----------|--------|
| 1 | `binding-gold` | `✦` |
| 2 | `binding-teal` | `◆` |
| 3 | `binding-purple` | `❖` |

### `G.bindingCount`

Integer, 0–3. Incremented by `applyBinding`. When `bindingCount >= 3`, `G.metrics.bindingMaxed` is set to `true` and all further `applyBinding` calls return immediately. Max 3 bindings per run.

### `getAvailableBindingPairs()`

Filters `BINDING_SLOT_PAIRS` to pairs where neither slot is already present in `G.bindings` (as slotA or slotB). Returns array of eligible pairs.

### `applyBinding(type)`

1. Returns immediately if `G.metrics.bindingMaxed`.
2. Calls `getAvailableBindingPairs()`. Returns immediately if empty.
3. Picks a random pair.
4. Creates binding object, pushes to `G.bindings`, increments `G.bindingCount`.
5. Sets `G.metrics.bindingMaxed = true` if `bindingCount >= 3`.
6. Calls `renderBoard()`.

### `getBindingPartner(row, idx)`

Searches `G.bindings` for any binding where slotA or slotB matches `{row, idx}`. Returns `{binding, partner}` where `partner` is the other slot object, or `null` if not found.

### `getItemAtSlot(row, idx)`

Returns `board.front[idx]` (if `row==='front'`) or `board.back[idx]`. Returns `null` for empty slots.

### `getEnemyItemAtSlot(row, idx)`

Same as `getItemAtSlot` but reads from `enemyBoard`.

### `triggerBindingOnActivation(row, idx, side)`

Called from `tickSide` when a player item activates (only fires when `side==='player'`). Looks up the binding partner via `getBindingPartner(row, idx)`. If the partner item exists, is not broken, and `!partnerItem._bindingFiring`:

| Binding type | Effect |
|---|---|
| `investor` | Partner's `effectAmt` is boosted ×1.2 (rounded) for 1000ms, then restored. `_bindingFiring` is cleared in the timeout callback. |
| `architect` | Applies 1000ms of haste to the partner item (`applyHaste`). |
| `kelpie` | Applies 1000ms of slow (`applySlow`) to a random live enemy with `maxHp > 0`. |

`_bindingFiring` is set on `partnerItem` before the switch and cleared after (except `investor`, which clears it in its 1s timeout).

### `triggerBindingOnDamage(row, idx, side, amount)`

Called from `applyDmgTo` after damage lands on a player item (`target._side==='player'`), when `target._row != null`. Guards on `side==='player'` and `!partnerItem._bindingFiring`:

| Binding type | Effect |
|---|---|
| `monk` | Applies 15 plating to the partner item (`applyShield('player', 15, 'Monk Binding', partnerItem)`). |
| `dryad` | Heals the partner item for 5 HP (direct `partnerItem.hp` add, capped at `maxHp`). |

### Mirror binding — damage intercept in `applyDmgTo`

Before shield processing (and before hitCount checks), `applyDmgTo` checks:
- `target._side === 'player'`
- `target._row != null` and `target._idx != null`
- `!target._bindingFiring`

If the target has a `mirror` binding and its partner is a live player item (`!broken`, `!_bindingFiring`), **all damage is rerouted to the partner**. Both `_bindingFiring` flags are set for the duration of the recursive `applyDmgTo` call on the partner, then cleared. The original target takes zero damage.

### Visual rendering

Bound item cards get a CSS class from `{1:'binding-gold', 2:'binding-teal', 3:'binding-purple'}` keyed on `binding.visualIndex`. A small overlay `<div class="binding-sym">` at top-right shows the symbol (`✦`/`◆`/`❖`). `.binding-sym` is `position:absolute; top:2px; right:3px; font-size:7px`.

### `item._row`, `item._idx`

Set in `resetItemBattleState` second pass for player items only. `board.front[idx]._row='front'` and `._idx=idx`, similarly for back row. Enemy items have `_row=null` and `_idx=null`. Used by `triggerBindingOnActivation`, `triggerBindingOnDamage`, and the mirror intercept.

### `item._bindingFiring`

Boolean. Set to `false` in `resetItemBattleState`. Set to `true` on `partnerItem` before a binding trigger fires; cleared after. Prevents recursive binding triggers (e.g., if item A is bound to item B and item B is also bound to item A via a different binding).

### `item._breakHandled`

Boolean. Set to `false` in `resetItemBattleState`. Set to `true` inside `checkAndBreak` at the very start of break processing. If `item.broken && item._breakHandled`, `checkAndBreak` returns immediately — prevents double-break firing within the same battle tick.

---

## Item Pools

### `ITEMS` (basic, 3 keys)

`left_claws` (Hook), `spark_charm` (Surge), `bubble_shield` (Bubble Shield).

`SHOP_POOLS` — 3 day-indexed arrays of these 3 keys.

### `TYPED_ITEMS` (11 keys across 5 types)

| Type | Keys |
|------|------|
| electric | `lightning_claw`, `arc_bolt` |
| fire | `ember_fang` (Inferno), `cinder_claw` (Scorch) |
| water | `tide_fang` (Hydro Blast), `tidal_shell` (Coral), `riptide`, `current_band` (Water Gun) |
| steel | `bulwark_stance` (Reinforce), `alloy_fang` (Steel Jaw) |
| toxic | `venom_fang` |

`TYPED_ITEM_POOLS` constant exists as a per-type listing — currently used as a reference, not as a shop filter. Toxic pool also lists `caustic_mantle, mithridate, miasma, outbreak` (which actually live in `POISON_ITEMS`).

**Typed items are in the general shop pool** — `getItemsOfRarity` mixes all pools regardless of `G.type`. Type only affects which typed skills are offered at bond tree node picks.

### `PACIFIST_ITEMS` (8)

`solidarity, emergency_ration, honey_pot, thorn_mantle, training_dummy, chrysalis, patience, callus`.

### `HEADBUTT_ITEMS` (4)

`headbutt, ricochet, arsonist, spite`. `Relic` is the tag for items like Arsonist with passive world-altering effects.

### `BURN_ITEMS` (5)

`stoke, blaze, cinder_block, cauterize, fireworks`.

### `WATER_ITEMS` (6)

`glacier, hydro_cannon, whirlpool, deluge, flood, hot_springs`. **Slow displays as "Wet" and Haste as "Jolt"** in all item descriptions — keyword rename only, mechanic names (`slowMs`, `hasteMs`, `applySlow`, `applyHaste`) unchanged.

### `ELECTRIC_ITEMS` (5)

`live_wire, dual_shock, charging_station, feedback, thunder`.

### `STEEL_ITEMS` (5)

`ironclad, aegis, temper, wrecking_ball, thornback_armor`.

### `POISON_ITEMS` (4)

`caustic_mantle, mithridate, miasma, outbreak`.

### `MOM_ITEMS` (6)

`worn_locket, knitted_cap, dads_knife, sharpening_stone, heavy_walking_stick, grandmas_coin_purse`.

### `MOM_CONSOLATION_ITEMS` (3)

`bandage_roll, river_stone, salad_bowl`.

`river_stone` — display name "Mom's Rock", tag "She said you could throw it.", `isWinston:false`. It is not Winston.

---

## Wild Monster Pools

Wild enemies are drawn from day-specific pools. `getWildPool()` returns the correct array for `G.day`. `currentWildMonster` is a module-level variable set at battle start for all paths (day 1 auto, day 2+ pick) — used for post-battle name lookup.

`mkw(key)` helper creates an item from `getItemFactory(key)` with all battle fields initialized (same as `mk()` but works across all pools).

| Day | Pool constant | Monsters |
|-----|--------------|---------|
| 1 | `DAY1_WILD` (fixed, not a pool) | Scruffling |
| 2 | `WILD_MONSTERS` | Thornback, Mudpup, Gloomwing |
| 3 | `WILD_MONSTERS_DAY3` | Grubcap, Flicktail, Irongrub |
| 4 | `WILD_MONSTERS_DAY4` | Cindermaw, Bogwump, Cragback |
| 5 | `WILD_MONSTERS_DAY5` | Voltmaw, Ashwalker, Deepcrawl |
| 6 | `WILD_MONSTERS_DAY6` | Smogwing, Galvanite, Brimstone |
| 7 | `WILD_MONSTERS_DAY7` | Sporemantle, Whelkin, Slaghorn |
| 8 | `WILD_MONSTERS_DAY8` | Dreadshell, Nullshell, Ashencoil |
| 9 | `WILD_MONSTERS_DAY9` | Veinrot, Stormclad, Cinderwall |
| 10+ | `WILD_MONSTERS_DAY10` | Abyssalback, Ironveil, Plagueborn |

Each pool entry: `{name, icon, img?, tag, front:[3], back:[3]}`.

Day 1 is always Scruffling — auto-starts, no pick screen. Day 2+ shows a 3-option pick screen using `getWildPool()`.

`buildWildEnemyBoard(m)` sets up `enemyBoard` from a monster object, calls `applyDifficultyToBoard`, then checks `G.strangerMark` — if set, finds the matching item on the enemy board (by `itemKey` or `id`) and halves its starting HP. Clears `G.strangerMark` after applying.

---

## Difficulty System

`G.difficulty` — `'easy'` | `'normal'` | `'hard'`. Initialized from `localStorage.getItem('zorpDifficulty')||'easy'`. Written to localStorage by `selectDifficulty(d)`.

`applyDifficultyToBoard(items, winNumber)` — applies `upgradeItem()` passes to all living enemy items based on difficulty and current win number. On `'easy'` does nothing.

| difficulty | win 1 | win 2 | win 3 | win 4 | win 5 | win 6 |
|------------|-------|-------|-------|-------|-------|-------|
| easy | — | — | — | — | — | — |
| normal | — | +1 random | +2 random | all+1 | all+1, +2 random | all+2 |
| hard | all+1 | all+1, +2 random | all+2 | all+2, +2 random | all+3 | all+3, +2 random |

Called for wild boards (passing `G.day` as winNumber) and trainer boards (passing win index).

`selectDifficulty(d)` — writes to `localStorage`, updates difficulty pill styling on the title screen without re-rendering it. `startFromTitle()` reads difficulty into `G.difficulty` before beginning the professor flow.

Difficulty selector only appears on the title screen after `zorpBeatGame` localStorage key is set (i.e., after first run completion).

---

## Shop System

### SHOPS Array (7 entries)

```js
const SHOPS = [
  {id:'the_market',    coverName:'The Market',        coverEmoji:'🏪', coverLine:'Open for business.',                  tier:'beginner',     type:'market',        unlock:{minDay:1,requiresType:false}},
  {id:'typed_stall',   coverName:null,                coverEmoji:null, coverLine:'She only carries one kind.',           tier:'beginner',     type:'typed',         unlock:{minDay:1,requiresType:true}},
  {id:'the_clearance', coverName:'The Clearance',      coverEmoji:'🏷️', coverLine:'Everything must go.',                  tier:'beginner',     type:'clearance',     unlock:{minDay:1,requiresType:false}},
  {id:'the_back_room', coverName:'The Back Room',      coverEmoji:'🚪', coverLine:'He has better stuff in the back.',     tier:'intermediate', type:'back_room',     unlock:{minDay:2,requiresType:false}},
  {id:'upgrade_bench', coverName:'The Upgrade Bench',  coverEmoji:'🔨', coverLine:"She doesn't sell. She improves.",      tier:'intermediate', type:'upgrade_bench', unlock:{minDay:2,requiresType:false}},
  {id:'loot_crate',    coverName:'Loot Crate',         coverEmoji:'📦', coverLine:"He's not sure what's inside.",         tier:'intermediate', type:'loot_crate',    unlock:{minDay:3,requiresType:false}},
  {id:'wholesale',     coverName:'Wholesale',          coverEmoji:'🛒', coverLine:'The more you take the cheaper it gets.',tier:'intermediate', type:'wholesale',     unlock:{minDay:3,requiresType:false}},
];
```

**typed_stall** — only appears if `G.type !== null`. coverName and coverEmoji are null in the array; rendered dynamically using the player's type name and type icon.

### Shop type behaviours

| type | Items | Price |
|---|---|---|
| `market` | 3 items from `generateShopSlots(G.day)` | Standard via `getShopItemCost` |
| `typed` | 3 random items from `getTypedItemKeys(G.type)` | Standard via `getShopItemCost` |
| `clearance` | 3 items of lowest day-weighted rarity | 50% of standard price |
| `back_room` | 4 items of second-highest day-weighted rarity | Standard via `getShopItemCost` |
| `upgrade_bench` | Up to 3 upgradeable player items | `TIER_COSTS[nextDisplayRarity]` via `getShopItemCost` |
| `loot_crate` | 2–3 blind crate options (typed, rarity, weapon) | Fixed: 4–10g by rarity |
| `wholesale` | 5 items with discount scaling | Decreases per item bought |

### Key shop functions

`getShopPool(day)` — filters SHOPS by `minDay ≤ day` and `requiresType` vs `G.type`.

`showShopPicker()` — called at Day 2+ `shop1` node. Picks up to 3 shops via `TIER_WEIGHTS`, presents as card picker. Falls back to `showEncounterShop(1)` if pool empty. If only 1 shop available, opens it directly.

`showShop(shop)` — first calls `checkTrenchcoat(shop, cb)` (10% chance trenchcoat easter egg), then calls `_openShop(shop)`.

`_openShop(shop)` — switches on `shop.type` to render the appropriate UI.

`getShopItemCost(key, baseCost)` — if player already owns a copy of this item at `upgradeLevel < 3`, returns `TIER_COSTS[nextRarity]` where `nextRarity = getDisplayRarity({itemRarity, upgradeLevel: existing.upgradeLevel+1})`. Otherwise returns `baseCost`. **Never uses UPGRADE_MULTS for pricing.**

`getDisplayRarity(item)` — returns the item's effective rarity tier string, bumped by `upgradeLevel` (capped at mastered). Used by `getSellValue()`, `getShopItemCost()`, and card border/badge rendering so upgraded items show and sell at higher rarity.

`getSellValue(item)` — uses `getDisplayRarity(item)` for sell price via `TIER_SELL = {beginner:1, intermediate:2, advanced:3, mastered:4}`. No tier bonuses. Returns `item.sellOverride` if set.

### Pricing Constants

```js
// Canonical shop price per rarity tier. Single source of truth for item pricing.
const TIER_COSTS = {beginner:2, intermediate:4, advanced:6, mastered:8};

// Used ONLY for scaling item stats (effectAmt, maxHp) on upgrade. NOT used for pricing.
const UPGRADE_MULTS = [1.0, 1.25, 1.6, 2.0];
```

`UPGRADE_MULTS` touches `effectAmt` and `maxHp` exclusively — it is never involved in gold cost calculations. All pricing goes through `TIER_COSTS` keyed on `getDisplayRarity(item)`.

### Trenchcoat Easter Egg

`checkTrenchcoat(shop, onDone)` — 10% chance on any shop entry. Shows "A figure. Two items visible." with 2 items from one rarity tier above the current most common. Items may have reduced HP (75%; 5% chance full HP). Sets `G.metrics.trenchcoatFound=true`. Player can buy one or walk away; walk away calls `onDone()` (opens the real shop).

### Slot B — shop card in event picker

`pickEvents()` Slot B: 35% chance to inject a shop card instead of a paid/burden event. The shop card has `_isShopCard:true` and `_shopRef` pointing to a `SHOPS` entry chosen by `TIER_WEIGHTS`. Selecting it calls `showShop(shopRef)`.

### Day 1 vs Day 2+ shop routing

- `DAY1_NODES` `shop1` node → `showEncounterShop(1)` (fixed mom's market).
- `DAY2_NODES` `shop1` node → `showShopPicker()` (picker with up to 3 shop types).
- `DAY1_NODES` `shop2` (Road Stall) → `showEncounterShop(2)`.

---

## Skill System

### `SKILLS` object — generic skills by tier

- **prestige (1):** `potential` — +25% all effects while typeless. Pre-filled at HEAD on init and on every reset.
- **beginner (6):** `thick_fur`, `quick_reflexes`, `scrappy`, `resilience`, `refrigeration`, `fortified`.
- **intermediate (8):** `momentum`, `duality`, `kickstart`, `friction`, `benediction`, `assembled`, `crescendo`, `epicenter`.
- **advanced (14):** `endurance`, `last_stand`, `iron_hide`, `phoenix_downed`, `echo`, `waterproof`, `grounded`, `immunity`, `asbestos`, `shatter`, `war_chest`, `desperation`, `predator`, `flow_state`.
- **mastered (2):** `apotheosis`, `fizzle`.

### `TYPED_SKILLS[type][tier]` — by type and tier (12 per type, 60 total)

| Type | Beginner (3) | Intermediate (3) | Advanced (3-4) | Mastered (2) |
|------|--------------|------------------|----------------|--------------|
| electric | `feedback_loop, conductor, charged` | `resonance, capacitor, arc` | `overcharge, thunderclap, node, grounding_wire` | `infinite_loop, discharge` |
| fire | `kindle, stoked, backfire` | `searing, flash_point, ember_storm` | `conflagration, meltdown, hearth` | `wildfire, backdraft` |
| water | `current, pressure, drag` | `undertow, saturation, ebb` | `maelstrom, deep_current, tidal_force` | `tsunami, flood_gate` |
| steel | `temper, alloy, forge` | `cold_iron, iron_will, pressure_plate` | `bulwark, rampart, vanguard, iron_formation` | `impenetrable, plate_mail` |
| toxic | `virulence, festering, toxin` | `seeping, carrier, infected` | `sepsis, tipping_point, colony` | `pandemic, necrosis` |

### Skill Offer Logic

`getSkillOffers(nodeRarity)`:

- Typed player **with available typed skills at this rarity** → 3 offers: slot 1 always typed; slots 2-3 each 25% chance typed (75% generic).
- Typeless player or empty typed pool → 3 generic offers.
- `selectNode` shows 3 skill choices.

### Skill Implementation Wiring

All 60 typed skills are wired to battle effects. Generic skills are wired in `getSkillMods()` (passive multipliers), `initBattleState`, `resetItemBattleState`, `tickSide`, `runBattle` interval, `checkAndBreak`, and the action functions. Specific notes:

**Generic — notable:**
- `potential` — +25% all effects passive bonus (typeless only). Applied via `getSkillMods().globalMult`.
- `thick_fur` (+20 maxHp), `quick_reflexes` (×0.90 speed), `scrappy` (+6 dmg), `fortified` (+7 plating, scales with globalMult).
- `resilience` — first DAMAGE_RECEIVED grants +70 plating (`resilienceUsed`).
- `refrigeration` — +1 use to all provisions in `resetItemBattleState`.
- `momentum` (+4 per player activation), `crescendo` (5th activation ×2 effectAmt), `epicenter` (slot2/slot5 ×0.75 speed), `assembled` (×0.82 speed when 6/6 filled), `duality` (odd slots +10 effect, even +25 maxHp), `kickstart`/`friction` (slot 6 jolt/wet, max 3×), `benediction` (15 HP back row every 5s).
- `endurance` (×0.6 fatigue), `last_stand`, `iron_hide` (10s safety), `phoenix_downed`, `echo`, `waterproof`/`grounded`, `immunity` (poison interval 2000ms — same as base, redundant but present), `asbestos` (-2 burn stacks), `shatter` (enemy plating /2), `war_chest` (+gold/2 dmg), `desperation` (×1.5 effectAmt on each break — never mutates `baseEffect`), `predator` (×1.5 globalMult at 3+ enemy breaks), `flow_state` (×1.15 globalMult at 20s+).
- `apotheosis` — every player item with baseEffect gets effectAmt = round(baseEffect × UPGRADE_MULTS[3]).
- `fizzle` — halves effectAmt and baseEffect of all enemy untargetable passives.

**Electric — wired in `tickSide`, `applyHaste`, `applyBurn`, `dealDmg*`, `checkAndBreak`.**

**Fire — wired in `applyBurn` (Kindle), `dealDmg*`, `tickSide` burn ticks, `checkAndBreak`, `runBattle` interval.**

**Water — wired in `applySlow` (Saturation), `dealDmg*`, `tickSide` enemy activation, `checkAndBreak`, `getSkillMods`, `runBattle` interval.**

**Steel — wired in `applyDmgTo` (Temper, Cold Iron, Pressure Plate, Bulwark, Impenetrable, Iron Formation), `dealDmg*`, `tickSide` player activation, `getSkillMods`, `runBattle` interval (Forge), `resetItemBattleState` (Iron Formation init).**

**Toxic — wired in `applyPoison` (Pandemic, Virulence, Outbreak), `tickSide` poison ticks (Festering, Seeping, Tipping Point trigger), `dealDmg*`, `checkAndBreak`, `runBattle` interval (Necrosis).**

### Confirmed Skill Bug Fixes

The following bugs were audited and fixed. Do not reintroduce them:

- **`desperation`** — mutates only `effectAmt`, never `baseEffect`. Correct; `baseEffect` is the permanent baseline that `resetItemBattleState` derives from.
- **`duality`** — mutates only `effectAmt` (+10) and `maxHp` (+25). Never mutates `baseEffect` or `baseMaxHp`. Uses `dualityApplied` flag to prevent double-application.
- **`conductor`** — writes to `actMs` (not `baseActMs`). `tickSide` reads `(item.actMs||item.baseActMs)` so the within-battle mutation takes effect.
- **`virulence`** — guarded on `target._side==='enemy'`. Player skills must not apply to friendly targets.
- **`pandemic`** — guarded on `target._side==='enemy'`. Same reason.

### Other Battle Function Fixes

- **`dealDmgBackRow` fallback** — when back row is empty, the fallback call passes already-scaled `dmg` (not unscaled `base`) to `dealDmgRandom` to avoid applying `globalMult` twice.
- **`healItem` / `healAll`** — use `getSkillMods().globalMult` for heal scaling. The old separate `potMult()` function has been deleted.
- **`Backdraft`** — reads `battleState.totalBurnApplied` (cumulative burn applied to enemies this battle, tracked in `initBattleState` and incremented in `applyBurn`). Does not count active stacks on board.
- **`Shame`** — uses `DAMAGE_RECEIVED` event with `event.rawValue` (pre-shield damage), not `DAMAGE_DEALT`. Guards on `event.side==='enemy'`.
- **`Arsonist`** — `handleEvent` on `BATTLE_START` skips items where `targetable(i)` is false (i.e., `maxHp===0` items such as provisions and relics). Only items with HP are burned.
- **`applyShield` targetItem guard** — if `targetItem` is passed, checks `targetItem.maxHp > 0` before shielding. Items with `maxHp===0` cannot receive shields.
- **`gambler_throw`** — honor deduction is handled by the generic cost handler (`cost:{type:'honor',amount:3}`). The `gambler_throw` case body does not deduct honor itself; the cost handler is authoritative.
- **Training Dummy `hitCount` death path** — when `hitCount` reaches 0 inside `applyDmgTo`, `checkAndBreak` is called so break skills (Desperation, Phoenix Down, etc.) fire correctly. Previously the item was marked broken directly without going through `checkAndBreak`.
- **`_breakHandled` flag** — `checkAndBreak` checks `item._breakHandled` before processing; sets it to `true` immediately. Prevents double-break firing within the same battle tick when multiple damage sources converge.
- **`resetRun`** — no longer calls `showTitleScreen()` itself. All callers that need to go to the title screen (RESET button, PLAY AGAIN button, `goToTitleFromVictory`) call both `resetRun()` then `showTitleScreen()` explicitly.
- **Venom Fang** — `effectFn` mutates only `actMs`, not `baseActMs`. `baseActMs` remains 2000 throughout the battle so `resetItemBattleState` restores correctly on next battle.
- **Upgrade bench** — uses `getShopItemCost()` for upgrade pricing (which returns `TIER_COSTS[nextDisplayRarity]`), not `UPGRADE_MULTS`.
- **`getNeighbors`** — `behind` returns the back-slot item for front-row items only (`row==='front'`). `inFront` returns the front-slot item for back-row items only (`row==='back'`). Neither returns values cross-direction.
- **`applySlotBonuses`** — `_slotBonusApplied` guard prevents double-application when same item passes through placement code twice (e.g., on swap paths).
- **`resolveQuest`** — `'upgrade_pick'` falls through to `'quest_upgrade_pick'` handler (`case 'upgrade_pick': // fallthrough`). Both use the same board-picker upgrade flow.
- **`completedEvents` in `gossip_clue` handler** — the local variable is named `completedEvents` (reads `G.metrics.eventsCompleted`), not `done2`. It no longer shadows the outer `done` closure.
- **`backSpeedMult` removed from `getSkillMods`** — it was always 1.0, never written by any skill. Removed to eliminate dead state.
- **`Benediction`** — heals the lowest HP% back-row item each 5s tick (uses `reduce` to find minimum `hp/maxHp`), not the first back-row item.
- **`getSkillOffers`** — filters `activeSkills` from both typed and generic pools before sampling (`takenSkillIds` dedup applied to both `typedPool` and `genericPool` to prevent offering already-owned skills).
- **Wholesale** — removes single card DOM node on purchase (`document.getElementById('wholesale-card-'+idx).remove()`) without rebuilding the full list.
- **`fatigueFired`** — captured as `const fatigueFired=fatigueInterval!==null` before `stopBattle()` clears `fatigueInterval`. Used in analytics/logEvent calls.
- **`fireEvent` dropped events** — `fireEvent` logs `console.warn('fireEvent dropped:', event.type, event)` when the re-entrant guard is active. Makes cascade debugging visible.
- **`localStorage` try/catch** — `G.difficulty` initial value wrapped in IIFE with try/catch for private-browsing safety: `(()=>{try{return localStorage.getItem('zorpDifficulty')||'easy';}catch(e){return 'easy';}})()`.
- **Appraiser pay formula** — uses `Math.floor(getSellValue(item) * 1.5)` (not `Math.floor(item.cost * 1.5)`), so upgraded items sell at their upgrade-tier sell value × 1.5.

---

## Monster System

### G State Fields

`G` is initialized at the top of the script and reset by `resetRun()`. Core fields (initial literal):

```
gold:8, honor:13, maxHonor:13, wins:0, day:1, type:null, rivalDefeated:false,
dayStep:0, selectedSlot:null, battleRunning:false, lastResult:null,
wildChoice:null, pendingSkillNode:null, momGiftGiven:false,
monsterName:'Kip', monsterType:'fox', rivalName:'Rival', tutorialSeen:false,
secondWindUsed:false, secondWindBuff:null, steadfastActive:false,
consolationPending:false, betPending:false,
difficulty: (()=>{try{return localStorage.getItem('zorpDifficulty')||'easy';}catch(e){return 'easy';}})(),
metrics: {
  wildWins:0, wildLosses:0, trainerWins:0, trainersFought:0,
  poisonApps:0, burnApps:0, itemsSold:0,
  totalDamageDealt:0, itemsLostInBattle:0, burdensTaken:0,
  eventsDeclined:[], eventsCompleted:[], npcsMet:[],
  winsWithoutHonorLoss:0, butcherDeclines:0,
  hasWinston:false, loanActive:false, oathTaken:false,
  wildLostToday:false, bindingMaxed:false,
  trenchcoatFound:false, hermitVisited:false
},
burdens: [],
forkBonusGold:false, neighborSlotBonus:0,
inventory:[], slotBonuses:{front:{maxHp:0,damage:0},back:{maxHp:0,damage:0}},
nextEventIsWild:false, lockedSlots:[],
bindings:[], bindingCount:0, strangerMark:null
```

Note: All fields above — including `trenchcoatFound`, `hermitVisited`, `bindingMaxed`, `trainersFought`, `bindings`, `bindingCount`, and `strangerMark` — are now in the initial literal AND set in `resetRun()`. The `difficulty` field is wrapped in a try/catch IIFE for private-browsing safety.

**`G.metrics` — increment sites:**

| Field | Where incremented | Where read |
|-------|-------------------|------------|
| `wildWins` | `endBattle` — wild win path | Counter strip |
| `wildLosses` | `endBattle` — wild loss path | — |
| `trainerWins` | `endBattle` — trainer win path | Counter strip |
| `winsWithoutHonorLoss` | `endBattle` trainer win, only if `G.honor === G.maxHonor` | — |
| `burnApps` | `applyBurn` — when `attackerSide==='player'` and `item._side==='enemy'` | — |
| `poisonApps` | `applyPoison` — when `target._side==='enemy'` | Herbalist unlock (`>=5`) |
| `totalDamageDealt` | `applyDmgTo` — when `side==='enemy'` (player attacking), after HP reduction | — |
| `itemsLostInBattle` | `checkAndBreak` — when `side==='player'`, after `item.broken=true` | Counter strip |
| `itemsSold` | sell button handler | — |
| `eventsCompleted` | `resolveEventChoice` idx===0 + specific pushes (e.g. `'priest'` on upgrade_pick) | Gossip clue, Monk arc |
| `eventsDeclined` | `resolveEventChoice` idx>0 | — |
| `butcherDeclines` | `case 'butcher_decline'` effect handler | `checkEasterEggs()` |
| `wildLostToday` | `endBattle` wild loss; reset to `false` in `advanceDay()` | Unimpressed Farmer unlock |
| `oathTaken` | `case 'oath_debuff'` | Oath excludeFlags |
| `loanActive` | Set true when Butcher burden accepted; false when `butcher_collect` fires | Butcher excludeFlags |
| `hasWinston` | `hermit_walk_away` effect | Win 6 screen, Monk arc |
| `hermitVisited` | `hermit_indulge` and `hermit_walk_away` handlers | Hermit `excludeFlags:{hermitVisited:true}` |
| `trenchcoatFound` | `checkTrenchcoat` on trigger | — |
| `npcsMet` | push on meeting each NPC | Cartographer reward scaling |
| `burdensTaken` | `fireBurdenHandler` when burden accepted | — |
| `trainersFought` | `endBattle` — whenever `currentBattleType!=='wild'` (win or loss) | — |
| `bindingMaxed` | `applyBinding` when `bindingCount >= 3` | Binding event `excludeFlags` |

**`G.metrics.wildLostToday`** — set true on wild loss, false on day advance. Used as `performance` filter for the Unimpressed Farmer event.

`G.burdens` — array of active burden objects. Structure: `{id, icon, countdown, unit, colorAmber, colorRed, payload, onResolve, active}`. Managed by `tickBurdens()` and `fireBurdenHandler()`.

- `G.monsterType` — `'fox'` or `'hydra'`. Set during professor flow.
- `G.monsterName` — player-named, defaults `'Kip'` (fox) or `'Hydrax'` (hydra).
- `G.rivalName` — player-named, persists in localStorage `'zorpRivalName'`. Default `'Rival'`.
- `G.tutorialSeen` — module flag (also mirrored via `localStorage 'zorpTutorialSeen'` and read via `checkTutorialSeen()`).

Module-level state outside G: `bannedTypes[]`, `rivalRecord{wins,losses}` (localStorage-backed), `lastOfferedTypes`, `board`, `enemyBoard`, `pendingSecondWindBuff/Row/Idx`, `secondWindClarityPending`, `professorFlowActive`, `professorStep`, `currentWildMonster`, `currentEvent`.

### Tree Nodes

`FOX_TREE_NODES` — 6 nodes:
- `head` (prestige, parent null) — pre-filled with `potential`.
- `front_shoulder` (beginner, parent head).
- `back_shoulder` (intermediate, parent front_shoulder).
- `front_foot` (intermediate, parent front_shoulder).
- `back_foot` (advanced, parent back_shoulder).
- `tail_tip` (mastered, parent back_shoulder).

Typed players skip `front_shoulder` and fill 4 nodes. Typeless fills all 5 non-HEAD nodes.

`HYDRA_TREE_NODES` — 6 nodes:
- `body` (prestige, parent null).
- `neck_split` (beginner, parent body).
- `mid_left` (beginner, parent neck_split).
- `mid_right` (beginner, parent neck_split).
- `head_left` (mastered, parent mid_left).
- `head_right` (mastered, parent mid_right).

Helpers:
- `getTreeNodes()` — returns the array for current `G.monsterType`.
- `getSkillTreeSVGContent()` — returns SVG markup for current monster.
- `rebuildSkillTreeSVG()` — rebuilds SVG, resets `treeState`, syncs prestige root.
- `getAvailableNodes()` — returns nodes with parent filled and self empty (excluding HEAD).

`treeState` — `{nodeId: skill object or null}`. `activeSkills` — array of currently active skill objects.

### Image Constants — all populated (Cloudinary URLs)

**Background:** `BACKGROUND_IMG` — full-page radial background.

**Fox:** `FOX_DEFAULT, FOX_ELECTRIC, FOX_FIRE, FOX_WATER, FOX_STEEL, FOX_TOXIC`.
**Hydra:** `HYDRA_DEFAULT_IMG, HYDRA_STEEL, HYDRA_ELECTRIC, HYDRA_TOXIC, HYDRA_FIRE, HYDRA_WATER`. Note: `HYDRA_DEFAULT` (no `_IMG`) is an empty string — do not reference it.
**NPC:** `PROFESSOR_IMG`.

**Title screen / wild:** `ZORP_LOGO_IMG`, `SCRUFFLING_IMG`, `THORNBACK_IMG`.
**Wild day 2:** `MUD_PUP_IMG` (Mudpup), `GLOOMWING_IMG` (Gloomwing). Both Cloudinary URLs.

**Trainers:** `TRAINER_RIVAL_IMG, TRAINER_INVESTOR_IMG, TRAINER_ARCHITECT_IMG, TRAINER_MONK_IMG, TRAINER_MIRROR_IMG`. `TRAINER_RIVAL_RETURNS_IMG = TRAINER_RIVAL_IMG`.

**Items:** `WORN_LOCKET_GOLD, WORN_LOCKET_SILVER, HEAVY_WALKING_STICK, HEAVY_WALKING_STICK_BROKEN, SHARPENING_STONE, SHARPENING_STONE_BROKEN`.

`getFoxImage()` returns the right image for both fox and hydra typed variants. `updateFoxImage()` checks `professorFlowActive` to hide art during the professor flow. `getItemIcon(item)` returns the custom art URL or falls back to the emoji icon.

### TYPES Array

5 types: `electric, fire, water, steel, toxic`. No typeless entry — typeless is handled separately in `showTypeSelect`. Ice does not exist; if you see `id:'ice'` anywhere it is a regression (replace with water).

---

## Battle System

### Function Signatures (current)

```js
applyPoison(target, stacks, src)
applyBurn(item, stacks, src, attackerSide='player')
applyHaste(item, durationMs, src, sourceIsEnemy=false)
applySlow(item, durationMs, src, side='enemy')
applyDmgTo(target, dmg, src, side, attackerItem=null)
applyShield(side, amount, src, targetItem)
applyShieldAll(side, amount, src)
healItem(side, amount, src)
healAll(side, amount, src)
dealDmg(side, base, src)
dealDmgRandom(side, base, src)
dealDmgTwoRandom(side, base, src)
dealDmgBackRow(side, base, src)
dealDmgWithStatus(side, base, src, statusType, statusVal)
tickSide(b, side, tickMs)
resetItemBattleState(b)
initBattleState()
checkAndBreak(item, side)
fireEvent(event)
getSkillMods()                  // returns {maxHpBonus, speedMult, dmgBonus, platingMult, fatigueMult, globalMult, slot2SpeedMult, slot5SpeedMult}
                                // NOTE: backSpeedMult was removed — it was always 1.0 and never written by any skill.
getSkillOffers(nodeRarity)
getFatigueTarget(side)          // returns first non-broken maxHp>0 item: front[0]→front[2], back[0]→back[2]
```

### `battleState` fields (initBattleState)

Cached arrays (refreshed at battle start and in `checkAndBreak` after `item.broken=true`):
`playerItems:[]`, `enemyItems:[]`.

Generic / global:
`momentumBonus:0`, `fatigueDamageMultiplier:1`, `totalSidePlating:0`, `burnApplicationsToEnemy:0`, `totalBurnApplied:0`, `fireworksThreshold:6`, `outbreakTriggered:false`, `slowProcsThisBattle:0`, `floodTriggered:false`, `wetDurationMult:1`, `joltCumulativeMs:0`, `thunderFired:false`, `poisonApplicationsToEnemy:0`, `phoenixDownAvailable:true`, `resilienceUsed:false`, `crescendoCount:0`, `crescendoReady:false`, `benedictionMs:0`, `kickstartCount:0`, `frictionCount:0`, `assembledActive:false`, `dualityApplied:false`, `ironHideActive:(skill active)`, `ironHideExpired:false`, `flowStateActive:false`, `predatorActive:false`, `warChestBonus:(Math.floor(G.gold/2) if active else 0)`.

`totalBurnApplied` — cumulative burn stacks applied to enemy items this battle. Incremented in `applyBurn` when `item._side==='enemy'`. Used by **Backdraft** skill.

Electric: `electricActivationCount:0`, `capacitorFired:false`, `thunderclapFired:false`, `dischargeCount:0`, `dischargeFired:false`, `overchargeCount:0`, `conductorBonuses:{}`.

Fire: `emberStormFired:false`, `hearthMs:0`, `activatingItem:null`.

Water: `wetSecondsAccumulated:0`, `deepCurrentBonus:0`, `tidalForceActive:false`, `maelstromMs:0`, `tsunamiFired:false`, `floodGateFired:false`.

Steel: `forgeTick:0`, `pressurePlateFired:false`, `platingAbsorbedTotal:0`, `ironFormationActive:false`, `impenetrableUsed:{}`, `bulwarkUsed:{}`.

Toxic: `virulenceCount:0`, `seepingHealMs:0`, `tippingPointFired:false`, `criticalMassFired:false`, `necrosisMs:0`, `pandemicActive:false`, `colonySide:false`, `lastKillWasPoison:false`.

### `resetItemBattleState`

Per-item resets: `hp=maxHp, shield=0, maxShield=0, actElapsed=0, actPct=0, broken=false, _breakHandled=false, uses=maxUses, hasteMs=0, slowMs=0, _slowMaxMs=0, burnStack=0, burnTickMs=0, poisonStack=0, poisonTickMs=0, _side=('player'|'enemy'), _bindingFiring=false, _row=null, _idx=null, _slotBonusApplied=false`.

`item._row` and `item._idx` are then set in a second pass for player items: `board.front.forEach((item,idx)=>{if(item){item._row='front';item._idx=idx;}})` and similarly for back. Used by the binding system to locate items.

Also runs second passes for `apotheosis`, `refrigeration`, `duality` (with `dualityApplied` flag), `assembled`, `iron_formation` (sets `ironFormationActive` if all 3 front filled).

### Status Effects

| Status | Field | Tick interval | Damage rule |
|---|---|---|---|
| Burn | `burnStack`, `burnTickMs` | **2000ms** | `shield × 2` absorbs, remainder hits HP. `burnStack--` per tick. |
| Poison | `poisonStack`, `poisonTickMs` | **2000ms** base (900ms enemy w/ Festering) | Bypasses shield. Poison stacks are permanently permanent. `poisonStack` is never decremented anywhere in the codebase. The only writes to `poisonStack` are: `applyPoison` (adds stacks) and `resetItemBattleState` (zeroes at battle start). The tick loop deals `item.poisonStack` damage each tick but never reduces the stack. Stacks accumulate indefinitely until battle ends. |
| Haste | `hasteMs` | Countdown | Speed multiplier 2.0 (faster activation). |
| Slow | `slowMs` | Countdown | Speed multiplier 0.5 (slower activation). |

Haste and slow cancel when both active (net 1.0×). `applyHaste` uses `Math.max`; `applySlow` uses additive `+` (Wet stacks duration).

`immunity` skill sets `poisonInterval=2000` on player items — same as base, so no effective change.

### Fatigue System

`getFatigueTarget(side)` — returns the **first** non-broken item with `maxHp > 0` scanning front[0]→front[2] then back[0]→back[2]. Returns `null` if none found.

`startFatigue()` fires `FATIGUE_START` event, then runs a 500ms interval. Each tick: fatigue deals to `getFatigueTarget('player')` and `getFatigueTarget('enemy')` separately — **one item per side per tick**. Battle log shows `'Fatigue → [item name]'`. Base damage scales with time (starts at 2, +3 per 2s, cap 40). Player fatigue applies `fatigueMult` from skills. Enemy fatigue applies `battleState.fatigueDamageMultiplier` (set by Patience). `G.steadfastActive` halves player fatigue damage.

### `applyDmgTo` — mirror binding intercept

Before shield processing, `applyDmgTo` checks if the target has a `mirror` binding and a valid partner: if `target._side==='player'` and `target._row!=null` and `target._idx!=null` and `!target._bindingFiring`, it looks up the binding partner. If found and the partner is a live player item and `!partner._bindingFiring`, damage is rerouted entirely to the partner (both `_bindingFiring` flags are set during the recursive call to prevent loops). The original target takes zero damage. This intercept runs before hitCount checks and before shield processing.

### `applyShield` behavior

- Default target: lowest HP% ally with `maxHp > 0`. This is intentional, not a regression.
- Optional 4th param `targetItem`: if provided, not broken, and `maxHp > 0`, shield that specific item.
- `ITEM_SHIELDED` event no longer fired here — `cinder_block` etc fire their own.
- `totalSidePlating` incremented for player side.

### onBreak pattern

`item.onBreak(item)` is called after `item.broken=true` in all break paths: `checkAndBreak`, `applyDmgTo` hitCount path, `tickSide` uses-depletion, burn-hitCount, poison-hitCount. Burn/poison HP paths go through `checkAndBreak`. Currently used by Wrecking Ball.

### Special item mechanics

- **hitCount items:** Training Dummy uses `hitCount` instead of HP. `targetable()` checks `isTargetable===true` as override. `applyDmgTo` checks `hitCount` before shield logic and absorbs damage entirely. Burn and poison decrement hitCount per tick.
- **Technique items:** `isTechnique=true`. Bought as tomes, consumed on buy, placed as passive items. Show tome modal on purchase. Sell via `showTechSellModal`. Use `handleEvent` for all effects.
- **Chrysalis:** 1 HP, **0.8s activation**, gains 2 plating each tick. Breaks instantly from any damage.
- **Headbutt:** Deals damage to leftmost enemy front, then takes half as self-damage. Self-damage fires full `DAMAGE_RECEIVED` events.
- **Ricochet:** No HP/timer. `handleEvent` on `DAMAGE_RECEIVED` fires when item directly behind takes damage. Deals 3 to random enemy.
- **Arsonist:** No timer. `handleEvent` on `BATTLE_START` applies 5 burn to every non-broken item on both boards **where `targetable(i)` is true** (skips `maxHp===0` items). 160 HP — can be killed.
- **Spite:** 50hp, **0.8s activation**. Increments `effectAmt` by 1 each time any ally takes damage (guards on `event.item._side==='player'`, `event.item!==item`, `!item.broken`). Deals effectAmt to random enemy on activation. `effectAmt` starts at 1, resets to 1 in `resetItemBattleState`.
- **Venom Fang:** Position-aware. `effectFn` checks board slot and sets only `self.actMs` (not `baseActMs`) to 1500 if in slot 1, otherwise 2000. Base actMs stays 2000 — the within-activation `actMs` mutation changes the effective next activation without altering the battle-reset baseline. Applies 1 poison to random front enemy.
- **`battleState.fatigueDamageMultiplier`:** Default 1, set by `Patience` on `FATIGUE_START`. Applied to enemy fatigue damage in `startFatigue`.

### Poison mechanic — important

Poison stacks are permanently permanent. `poisonStack` is never decremented anywhere in the codebase. The only writes to `poisonStack` are: `applyPoison` (adds stacks) and `resetItemBattleState` (zeroes at battle start). The tick loop deals `item.poisonStack` damage each tick but never reduces the stack. Stacks accumulate indefinitely until battle ends.

`applyPoison` order: Pandemic (`stacks *= 2`, enemy targets only) → apply → Virulence (every 3rd player-applied application to enemy, +1 extra) → `poisonApplicationsToEnemy++` (enemy targets) → `checkOutbreak()` → Outbreak extra +1 (if triggered) → fireEvent → log.

### Imbue Types

`imbueItem(item, type, amount)` — applies an imbue effect to an item and records it in `item.imbues[]`.

| type | Effect |
|------|--------|
| `damage` | Adds `amount` to `effectAmt` and `baseEffect`. |
| `hp` | Adds `amount` to `maxHp`, `baseMaxHp`, and `hp`. |
| `burn_on_act` | On activation: applies `amount` burn stacks to a random enemy (uses `handleEvent` wrapper). |
| `poison_on_act` | On activation: applies `amount` poison stacks to a random enemy (uses `handleEvent` wrapper). |
| `act_speed` | Reduces `baseActMs` by `amount`% (floor 500ms). Updates `actMs` immediately. |
| `hp_pct` | Increases `maxHp` by `amount`% of current value. Updates `baseMaxHp` and `hp`. |
| `jolt_on_act` | On activation: applies `amount` seconds of haste to a random ally (excluding self). |
| `wet_on_act` | On activation: applies `amount` seconds of slow to a random enemy with `maxHp > 0`. |

---

## Run Event System

Separate from the battle event bus. Handles narrative events on the day track between locations.

### EVENTS Array — 39 entries

Event schema: `{id, name, coverName, coverEmoji, coverLine, tier, image, unlock:{minDay,maxDay,minWins,maxWins,excludeFlags,performance,minHonor,metricFlags}, validSlots, state, prompt:{title,flavor,dialogue}, choices:[{id,label,cost,requiresCost,effect,burden,outcomeText}], resolution, tags, easter_egg}`

- `tier` — `'beginner'` | `'intermediate'` | `'advanced'`. Controls frequency via `TIER_WEIGHTS`.
- `validSlots` — `['pre_wild']` | `['post_wild']` | `null` (any). Restricts to slot context.
- `state` — `'boardNotFull'` blocks event when all 6 slots are filled. `null` = no state check.
- `tags` — `'free'` (Slot A), `'paid'`/`'burden'`/`'forced'` (Slot B), or mixed.
- `resolution` — `'immediate'` (player picks) | `'automatic'` (no choice, fires immediately).
- `outcomeText` — string shown in `showBurdenPanel` after choice before advancing. `null` = advance directly.
- `requiresCost:false` with a `cost` — cost is taken regardless of availability (forced payment, e.g. Tax Collector).
- `excludeFlags` — **can be array or object**. Array `['fieldName']`: blocks if `G.metrics[fieldName]` is truthy. Object `{fieldName:true}`: same. `getEventPool` handles both formats.

| ID | Name | Tier | Unlock | ValidSlots | Tags | Key mechanic |
|----|------|------|--------|------------|------|-------------|
| `the_rock` | The Rock | beginner | D1-4 | any | free,imbue | Pick: +5 dmg or +10 HP to one weapon/item |
| `sharpening_stone` | The Sharpening Stone | beginner | D1-4 | any | paid,imbue | Free +3 dmg; or 4g for +7 dmg (weapon) |
| `tax_collector` | The Tax Collector | intermediate | D2-8 | any | forced,imbue | Forced 2g (requiresCost:false), +3 dmg all items |
| `fork_in_the_road` | The Fork In The Road | beginner | D1-6 | pre_wild | free,fork | Left: +1g on wild win; Right: skip wild, typed shop |
| `butcher` | The Butcher | intermediate | D2-8, W0-5, excl:loanActive | any | burden,gold | Accept +6g + butcher_loan (repay 7 in 15 locs); Decline tracks butcherDeclines |
| `the_bet` | The Bet | intermediate | D3+, W0-5 | any | burden,risk | Accept: items hit softer (bet_debuff, 10 locs) + free any-node pick after |
| `the_kid` | The Kid | beginner | D1-4 | any | paid,imbue | 2g: random imbue (type first, then eligible item) |
| `old_mans_cart` | The Old Man's Cart | beginner | D1-4 | any | free | Places a random item (respects lockedSlots, applySlotBonuses) |
| `unimpressed_farmer` | The Unimpressed Farmer | beginner | D1-5, perf:wildLostToday | post_wild | free,performance | Free upgrade_weapon_pick |
| `generous_neighbor` | The Generous Neighbor | beginner | D2-5, boardNotFull | pre_wild | free,forced | Auto: neighborSlotBonus+1 (no choice, 1.5s auto-advance) |
| `the_cartographer` | The Cartographer | intermediate | D4-10 | any | free,performance | Reward scales with npcsMet count |
| `the_hermit` | The Hermit | intermediate | D2-9, excl:hermitVisited | any | free,risk | Indulge: downgrade item + burn imbue; Walk away: item loses 10% HP, gives Winston |
| `the_blacksmith` | The Blacksmith | intermediate | D2-9 | any | paid,imbue | Free: she picks upgrades; 4g: player picks |
| `merchants_daughter` | The Merchant's Daughter | intermediate | D2-8 | any | paid | Free browse: 3 intermediate items at ~50% prices; board-full: swap picker |
| `the_appraiser` | The Appraiser | intermediate | D2-9 | any | free,paid | Sell one item for `Math.floor(getSellValue(item) * 1.5)` gold |
| `the_oath` | The Oath | intermediate | D2-8, excl:oathTaken | any | burden,risk | Accept: debuffs back row + oath_resolve burden (restores + +35 maxHp back after 15 locs) |
| `the_pawnbroker` | The Pawnbroker | intermediate | D3-9, W1+ | any | paid,burden | Shows warning panel first; Deal: gold for a locked slot (pawnbroker_unlock returns it in 5 locs) |
| `the_surgeon` | The Surgeon | intermediate | D3-9, W1+, minHonor:6 | any | paid,honor | 5 honor: upgrade **two** items (pick first, then second) |
| `the_gambler` | The Gambler | intermediate | D3-9, W1-5 | any | paid,burden,honor | Throw: -3 honor + receive a random advanced item; Refuse: HP ×0.75 (gambler_odds burden restores in 7 locs) |
| `the_collector` | The Collector | intermediate | D3-9 | any | free,risk | Trade: sell worst item, get random higher rarity item |
| `the_gossip` | The Gossip | intermediate | D3-9, W1+ | any | free | Three choices: clue / gossip_damage (burden) / gossip_wild (nextEventIsWild=true) |
| `the_stranger` | The Stranger | beginner | D1-5 | pre_wild | free,paid | Free: shows ONE random wild monster board, tap item to mark it (50% HP start); 3g: shows all options then sell/replace |
| `the_herbalist` | The Herbalist | beginner | D2-8, perf:poisonApps>=5 | any | free,performance | Free herbalist_imbue on a weapon |
| `notice_board` | The Notice Board | beginner | D1-4 | any | free | Shows 3 random quests (50/50 better/worse); player picks one |
| `the_puddle` | The Puddle | beginner | D1-4 | any | free,forced | Forced single choice "Let it finish" → puddle_drink (50/50 good/bad) |
| `the_shrine` | The Shrine | intermediate | D2-9 | any | free | Auto: random item gains `act_speed` -8% imbue. Shows burden panel with result. |
| `the_molt` | The Molt | intermediate | D2-9 | any | free,risk | Eat: pick upgraded item → downgrade + gain +20% maxHp at new tier; Leave: random item gains `hp_pct` +5% |
| `the_crow` | The Crow | intermediate | D2-9 | any | free | Auto: +1 gold. Shows burden panel. |
| `the_ruins` | The Ruins | intermediate | D2-9 | any | free | Search: shows a random day-rarity item; player takes it or leaves it |
| `the_veteran` | The Veteran | intermediate | D2-9, perf:wildLosses>=1 | any | free,performance | Take it: receives typed item (if typed) or day-weighted item; board full → swap picker |
| `traveling_merchant` | The Traveling Merchant | intermediate | D2-9 | any | free | Three favour options: clearance item / generic day-weighted item / typed item |
| `the_cache` | The Cache | intermediate | D2-9 | any | free | Pick 1 of 2 random provision items |
| `the_letter` | The Letter | intermediate | D2-9 | any | free | 50/50: gold (1–2g) OR random item gains `jolt_on_act` or `wet_on_act` +1s imbue |
| `event_investor_binding` | The Investor | beginner | D1-6, excl:bindingMaxed | any | paid | Pay 4g: apply `investor` binding to a random available slot pair |
| `event_architect_binding` | The Architect | intermediate | D2-8, excl:bindingMaxed | any | paid | Pay 4g: apply `architect` binding to a random available slot pair |
| `event_monk_binding` | The Monk | intermediate | D3-9, excl:bindingMaxed | any | paid | Pay 4g: apply `monk` binding to a random available slot pair |
| `event_kelpie_binding` | The Kelpie | intermediate | D2-8, excl:bindingMaxed | any | free | Auto-fires (no choice): apply `kelpie` binding immediately on event display |
| `event_dryad_binding` | The Dryad | advanced | D5-10, excl:bindingMaxed | any | free | Accept gift (free): apply `dryad` binding to a random available slot pair |
| `event_mirror_binding` | The Mirror | advanced | D6-10, excl:bindingMaxed | any | free | Accept binding (free): apply `mirror` binding to a random available slot pair |

### WILD_FIGHT_OPTION

Special constant — not in `EVENTS` array. Slot C wildcard (pre_wild only, 10% chance):

```js
{
  id:'bonus_wild_fight', name:'Something Stirs',
  coverName:'Something Stirs', coverLine:'You could ignore it.', coverEmoji:'🌿',
  tags:['wild'],
  choices:[{investigate → trigger_wild_fight}, {keep_moving → null}]
}
```

### Easter Egg — The Priest

Not in `EVENTS` array. Lives in `EASTER_EGGS.priest`. Triggered by `checkEasterEggs()`.

- **Trigger:** `G.metrics.butcherDeclines >= 2`. On trigger, resets `butcherDeclines = 0` so it can re-trigger later.
- **Effect:** `upgrade_pick` — player picks any item to upgrade one level.
- **eventsCompleted:** `resolveEventChoice` pushes `'priest'` (short form, not `'the_priest'`) when `event.id==='the_priest'` inside `case 'upgrade_pick':`.
- **Gossip clue hint[1]** checks `eventsCompleted.includes('priest')` to reveal the "tell the Butcher no twice" hint.
- No coverName/coverEmoji/coverLine — replaces the event picker entirely via `showEvent(egg)`.

### Event System Functions

```js
getEventPool(day, slotCtx='any')
  // Filters EVENTS by criteria:
  // 1. validSlots — if set, slotCtx must be in the array
  // 2. state === 'boardNotFull' — blocks if all 6 slots filled
  // 3. unlock.minDay / maxDay
  // 4. unlock.minWins / maxWins
  // 5. unlock.excludeFlags — array ['field'] or object {field:true}; blocks if G.metrics[field] truthy
  // 6. unlock.performance — truthy metric key OR numeric comparison 'poisonApps>=5'
  // 7. unlock.minHonor — blocks if G.honor < minHonor
  // 8. unlock.metricFlags — all listed fields must be truthy in G.metrics

pickEvent(day)
  // Legacy single-event weighted pick (beginner=3×, intermediate=2×, advanced=1×).
  // Used by the old showEvent path. Returns null if pool empty.

pickEvents(day, slotCtx='any', count=3)
  // 3-slot picker system. Uses TIER_WEIGHTS[day] for weighted selection.
  // Slot A — guaranteed free event (tag:'free'). Falls back to any eligible.
  // Slot B — 35% chance: shop card from getShopPool(day). 65%: paid/burden event. Falls back to any eligible.
  // Slot C — 10% chance WILD_FIGHT_OPTION (pre_wild only), else any eligible.
  // Never repeats the same event ID in one picker. Returns array of up to 3 events.

showEventPicker(events)
  // Renders a card picker UI. Player clicks one event card to select it.
  // Shop cards (isShopCard:true) call showShop directly on select.
  // Selected event becomes currentEvent; shows its full prompt + choices.

resolveEventChoice(idx)
  // Called with choice index. Captures event and choice before clearing currentEvent.
  // Defines advance() and done() closures:
  //   advance() = G.dayStep++; updateUI(); renderTrack(); activateCurrentNode()
  //   done() = choice.outcomeText ? showBurdenPanel(title, outcomeText, advance) : advance()
  // Cost check (gold/honor deduction) happens before the effect switch.
  // Pushes to eventsCompleted (idx===0 accept) or eventsDeclined (idx>0 decline).
```

**`apply_binding` effect handler** (`case 'apply_binding':` in `resolveEventChoice`): calls `applyBinding(effect.bindingType)`. If a new binding was created (detected by `G.bindings.length` before/after), shows a burden panel naming the two bound slots with the binding's visual symbol (`✦`/`◆`/`❖`). If no pair was available (all pairs already bound), calls `done()` directly. The Kelpie event is special: it has empty `choices[]` and fires `applyBinding('kelpie')` automatically from the no-choice handler before the generic auto-advance.

`TIER_WEIGHTS` — object keyed by day (1–10). Each entry: `{beginner, intermediate, advanced, mastered}` relative weights. Day 1: beginners only. By day 8+: mastered 50%, advanced 50%.

`currentEvent` — module-level. Holds the active event during player choice. Set to `null` at start of `resolveEventChoice` (before the switch). The local `const event` and `const choice` captured before the null assignment remain in scope throughout.

`showEvent(null)` — gracefully skips: `G.dayStep++; updateUI(); renderTrack(); activateCurrentNode()`.

No-choice events (empty `choices[]`) fire their top-level `effect` immediately, show flavor text, and auto-advance after 1500ms. Used by `generous_neighbor`. Shrine, Crow, and Cache have dedicated handlers in the no-choice block before the generic auto-advance.

---

## Quest System

### QUESTS Array (5 entries)

Each quest has `{id, label, better:{text,effect}, worse:{text,effect}}`. Outcome is 50/50 random.

| id | label | better effect | worse effect |
|----|-------|---------------|--------------|
| `q_scrufflings` | "Kill 3 Scrufflings. Easy enough." | imbue_row front damage +4 | imbue_row front damage +2 |
| `q_guard_road` | "Guard the road. Someone has to." | imbue_row back hp +8 | imbue_row back hp +4 |
| `q_collect_herbs` | "Collect herbs for the village. Shouldn't take long." | gold +3 | gold +1 |
| `q_investigate_cave` | "Investigate a cave. Probably nothing." | imbue_random_item_random_type | imbue_pick hp +5 |
| `q_deliver_message` | "Deliver a message to the next town. Shouldn't take long." | upgrade_pick | gold +3 |

Both better and worse outcomes always give something — no quest has a null effect.

### Quest functions

`noticeQuestPick(idx)` — called when player taps a quest card. Retrieves quest from `window._noticeQuests[idx]`, clears it, calls `resolveQuest(quest)`.

`resolveQuest(quest)` — 50/50 coin flip. Runs the effect from `outcome.effect`, then calls `showBurdenPanel('NOTICE BOARD', text, advance)`. Effect types handled: `gold`, `imbue_row`, `imbue_random_item`, `hurt_random_item`, `imbue_random_item_random_type`, `quest_upgrade_pick` (shows board picker, then burden panel on pick), `upgrade_pick` (for deliver_message better outcome — same as event upgrade_pick). Quest `imbue_random_item` filters to items with effectAmt>0 or maxHp>0 before picking.

The **notice_board** event handler (in `showEvent`) renders 3 random QUESTS as tappable cards without going through `resolveEventChoice` — it intercepts before the normal choice flow.

---

## Burden System

`G.burdens` — array of active burden objects. `tickBurdens()` is called at the start of `activateCurrentNode()` (via `tickBurdens()` check) and decrements `countdown`; when it hits 0, calls `fireBurdenHandler(burden, onDone)`.

`fireBurdenHandler(burden, onDone)` — removes burden from `G.burdens`, calls `renderCounterStrip()`, then switches on `burden.onResolve`:

| onResolve | Effect |
|-----------|--------|
| `butcher_collect` | Deducts repayAmount (default 7) from G.gold (min 0). Shows panel. Sets `loanActive=false`. |
| `bet_resolve` | Restores original effectAmts from payload. Sets `G.betPending=true` (skips next skill pick). Shows panel. |
| `oath_resolve` | Restores original back row effectAmts. Applies +35 maxHp to current back items. Sets `G.slotBonuses.back.maxHp+=35`. Shows panel. |
| `pawnbroker_unlock` | Removes matching entry from `G.lockedSlots`. Shows panel "The slot is yours again." |
| `gambler_resolve` | Restores original maxHp values from payload. Shows panel. |
| `gossip_resolve` | Restores original effectAmts from payload. Calls `onDone()` directly (no panel). |
| *(default)* | Calls `onDone()` directly. |

`showBurdenPanel(title, bodyText, onDone)` — renders a panel with a Continue button. Stores `onDone` in `_burdenContinue`. **No auto-dismiss** — the panel requires a manual tap on Continue. `_burdenAutoTimeout` still exists as a variable but is never assigned a timeout in `showBurdenPanel`. `clearTimeout(_burdenAutoTimeout)` calls throughout the code are safe no-ops.

`resolveBurden(burdenId)` — removes burden by id, calls `renderCounterStrip()`. For manual resolution (not countdown-based).

### Counter Strip UI

`<div id="counterStrip">` — positioned in `right-col` above the day track (`#dayTrack`). Shows burden countdown chips only.

`renderCounterStrip()` — called from `updateUI()`, `activateCurrentNode()`, and after any burden state change. For each active burden: renders `{icon} {countdown}` colored by colorRed/colorAmber thresholds. Hides element when no burdens. Does **not** show wildWins/trainerWins/itemsLostInBattle — those metrics are not displayed in the strip.

### slotCtx — Event Slot Context

Computed in `loadCurrentNode()` when routing to an event node:

```js
const slotCtx = wildIdx < 0 ? 'any' : G.dayStep < wildIdx ? 'pre_wild' : 'post_wild';
```

- `pre_wild` — this event node appears before the wild fight in the day track.
- `post_wild` — this event node appears after the wild fight.
- `any` — no wild fight in this day's nodes (DAY1_NODES has no event nodes).

`getEventPool` and `pickEvents` both accept `slotCtx`. Events with `validSlots:['pre_wild']` only appear before the wild fight; `validSlots:['post_wild']` only after.

If `G.nextEventIsWild` is true, `loadCurrentNode()` skips the event and calls `showEncounterWildPick()` instead, then clears the flag.

---

## Winston Arc

### The Hermit → Inventory

`the_hermit` event (intermediate, days 2–9, excludes if `hermitVisited:true` — fires once per run).

- **Indulge** (`hermit_indulge`): Shows board picker of all items (preferring upgraded ones). Chosen item is downgraded one tier and gains `burn_on_act` +2 imbue. Sets `hermitVisited=true`. Pushes `'hermit'` to `npcsMet`. **Does not give Winston.**
- **Walk away** (`hermit_walk_away`): A random item loses 10% maxHp. Adds `{id:'winston', name:'A Rock', icon:'🪨', tag:'He knew everything.', isWinston:true}` to `G.inventory`. Sets `hasWinston=true`, `hermitVisited=true`. Pushes `'hermit'` to `npcsMet`. **Walk away gives Winston.**

Winston is displayed in `renderInventoryStrip()` (called from `renderBoard()`). He has no battle effect. He is not river_stone.

### Win 4 — The Monk Interaction

`postTrainerWin()` checks `G.wins===4 && G.metrics.hasWinston && !eventsCompleted.includes('monk_winston')` before the normal type/skill flow.

Three-panel arc:

1. **Panel 1** — "Can I see that rock?" → [Show him] → `monkWinstonShowHim()`
2. **Panel 2** — "A great amount of energy..." → [Return the rock] or [Keep Winston]
3. **Outcome A — Return** (`monkWinstonReturn`): `hasWinston=false`, removes Winston from `G.inventory`, pushes `'monk_winston_returned'` and `'monk_winston'` to eventsCompleted. Shows burden panel, then board picker: chosen item gains +20% of its current maxHp (min 1). Then continues to normal type/skill flow.
4. **Outcome B — Keep** (`monkWinstonKeep`): pushes `'monk_winston_kept'` and `'monk_winston'` to eventsCompleted. Shows burden panel "He watches you leave." Then continues to normal type/skill flow.

Subsequent Win 4 runs (after `monk_winston` is in eventsCompleted) skip directly to normal flow.

### Win 6 — Victory Screen

If `G.metrics.hasWinston` at win 6:
- Sets `localStorage.zorpWinstonFound = '1'`
- Appends clickable 🪨 to the victory screen subtitle. Tapping it cycles through `WINSTON_WISDOM` lines via `cycleWinstonWisdom()` (increments `_winstonWisdomIdx`).
- Calls `showWinstonToast()` — 5s fixed top-center toast with gold border, "🏆 ACHIEVEMENT UNLOCKED / He knew everything."

`WINSTON_WISDOM` — array of 10 lines, defined near the end of the file. Selected starting at a random index.

Win 6 button: `CONTINUE` → `goToTitleFromVictory()` which calls `resetRun()` then `showTitleScreen()`. (Not "PLAY AGAIN" — win 6 always routes to title, not a new run directly.)

### Title Screen — zorpWinstonFound

`showTitleScreen()` checks `localStorage.getItem('zorpWinstonFound')`. If set:
- Randomises `_titleWinstonIdx` from `WINSTON_WISDOM`.
- Renders large 🪨 bottom-left with a wisdom line below it. Clicking cycles to the next wisdom line via `cycleTitleWinston()`.
- Scruffling art is **replaced** by Winston when found (not shown alongside).

If `zorpWinstonFound` not set: Scruffling art bottom-left (decorative, not clickable).

Thornback art bottom-right always (both cases).

---

## Achievements

### Minimalist

- **Trigger:** Win the game (win 6) with 5 or fewer non-broken items on the board.
- **Storage:** `localStorage.setItem('zorpAchievementMinimalist', '1')`.
- **Display:** Achievement banner appended to the win 6 victory screen subtitle.
- **Unlocks:** Random monster option (🎲 "SURPRISE ME") on the monster selection screen. Shown when `localStorage.getItem('zorpBeatGame')` is set (same gate as difficulty selector).

### Winston Found (zorpWinstonFound)

- **Trigger:** Win the game (win 6) while `G.metrics.hasWinston` is true.
- **Storage:** `localStorage.setItem('zorpWinstonFound', '1')`.
- **Display:** Achievement toast on win screen + 🪨 replaces Scruffling on title screen permanently.

---

## Day Structure

```
DAY1_NODES: mom → shop1 → wild → shop2 → rival
DAY2_NODES: shop1 → event → wild → event → rival
```

`getDayNodes()` returns `DAY2_NODES` when `G.rivalDefeated && G.day > 1`, otherwise `DAY1_NODES`.

`activateCurrentNode()` calls `tickBurdens()` first (returns early if a burden fires), then `renderCounterStrip()`, then `loadCurrentNode()`.

`loadCurrentNode()` routes on `node.id`:
- `'mom'` → `showEncounterShop(0)` (Mom's gift)
- `'shop1'` → Day 1: `showEncounterShop(1)`. Day 2+: `showShopPicker()` (picker with up to 3 shop types).
- `'shop2'` → `showEncounterShop(2)` (Road Stall, Day 1 only)
- `'wild'` → `showEncounterWildPick()`
- `'event'` → computes `slotCtx`, checks `G.nextEventIsWild` (if true: clear flag, `showEncounterWildPick()`), else `checkEasterEggs()` then `showEventPicker(pickEvents(G.day, slotCtx))`
- `'rival'` → `showEncounterBattle('rival')`

### checkEasterEggs

```js
checkEasterEggs(day, locationIndex)
```

Called at event node activation. Returns an event object or `null`.

Current behavior: if `G.metrics.butcherDeclines >= 2`, resets `butcherDeclines=0` and returns `EASTER_EGGS.priest`. The returned event (if non-null) replaces the normal event picker — `showEvent(egg)` is called instead of `showEventPicker(...)`.

---

## Title Screen

`showTitleScreen()` — renders a fixed overlay (`z-index:200`) containing:
- ZORP logo image (`ZORP_LOGO_IMG`)
- Difficulty selector pills (easy / normal / hard) — only shown when `localStorage.getItem('zorpBeatGame')` is set
- START button → calls `startFromTitle()`
- Bottom-left: Winston 🪨 (clickable, cycles wisdom) if `zorpWinstonFound` is set; otherwise Scruffling art (decorative)
- Bottom-right: Thornback art (decorative, always shown)

`startFromTitle()` — reads `zorpDifficulty` from localStorage into `G.difficulty`, removes the title screen overlay, appends a small ZORP logo watermark (position:absolute, top-left of `#encounterArea`, opacity 0.22, width 72px), then calls `showProfessorFlow()`.

`selectDifficulty(d)` — writes `zorpDifficulty` to localStorage and updates pill border/background styles in place.

`showWinstonPanel()` — self-contained modal (createElement, z-index:300). Shows `"He knew everything."` with a Close button. (Legacy — used on older title screen path; current title screen uses the cycling 🪨 div instead.)

---

## Professor Flow

Module-level: `professorFlowActive` (boolean) and `professorStep` (0-6 counter).

```
showProfessorFlow()              // sets professorFlowActive=true, professorStep=0, calls showProfessorStep
  └ showProfessorStep()          // switch on professorStep:
      0 → showProfessorIntro()
      1 → showProfessorShop()    // → professorShopBrowse() or professorShopDecline()
      2 → showMonsterSelection() // FOX vs HYDRA (+ RANDOM if zorpBeatGame set)
      3 → showMonsterNaming()
      4 → if checkTutorialSeen() → step=5 + recurse
          else                  → showTutorial()
      5 → showRivalNaming()      // → confirmRivalName()
      6 → beginRun()             // sets professorFlowActive=false
```

`profLayout(dialogueHTML, extraHTML)` wraps content with the 90px professor portrait on the left and content on the right. Used on every professor screen.

**Random monster option** (`🎲 SURPRISE ME`) — shown on `showMonsterSelection()` when `zorpBeatGame` is set. `confirmMonsterSelection()` resolves `'random'` to fox or hydra (50/50) before proceeding to naming screen.

### Tutorial

`tutorialStep` is a local counter. `showTutorial()` sets `tutorialStep=0` and calls `showTutorialStep()`. Cases:
- 0 → `showTutorialCards()` — card anatomy + demo battle.
- 1 → `showTutorialFight()` — auto-battle + demo battle.
- 2 → `showTutorialKeywords()` — burn/poison/jolt/wet.
- 3 → `showTutorialJourney()` — honor/wins/bond tree.
- 4 → marks tutorial seen, advances `professorStep=5` + `showProfessorStep()`.

`skipTutorial()` marks tutorial seen and advances to step 5 immediately.

### Demo Battle (isolated from main game)

State: `demoItems{shield, spark}`, `demoInterval`, `demoMs`, `demoFatigueMs`, `demoFatigueActive`, `demoResetTimeout`.

Functions: `makeDemoItem`, `initDemoItems`, `tickDemo`, `renderDemoCards`, `setDemoCommentary`, `startDemoBattle`, `stopDemoBattle`.

Demo runs at 100ms tick. Bubble Shield (200 HP) outlasts Spark Charm (80 HP) due to fatigue. Demo no longer auto-loops — stops on completion, REPLAY button after 1.5s, NEXT/SKIP always accessible.

---

## Trainer System

`TRAINER_DATA` is an array with indices 1-6 (index 0 unused).

Each entry has: `name` (or getter for dynamic rival name), `img`, `icon` (emoji fallback), `tag`, `dialogueBefore` (or getter), `dialogueAfterWin` (or getter), `dialogueAfterLoss` (or getter), `buildTeam()`. Entry 1 and 6 use getters for `name` returning `G.rivalName.toUpperCase()`.

| # | Name | Notes |
|---|------|-------|
| 1 | THE RIVAL (G.rivalName) | Day 1 boss. Tag "Your old neighbour". |
| 2 | THE INVESTOR | Tag "Knows the value of things". |
| 3 | THE ARCHITECT | |
| 4 | THE MONK | |
| 5 | THE MIRROR | |
| 6 | THE RIVAL RETURNS (G.rivalName) | Win 6 boss. `buildTeam` reads `G.type` and builds counter board. |

### Win 6 — Counter Boards

| G.type | Counter board |
|--------|---------------|
| electric | Steel: Thornback Armor, Aegis, Ironclad, Temper, Wrecking Ball, Hard Shell |
| fire | Water: Glacier, Hydro Cannon, Whirlpool, Deluge, Hot Springs, Coral |
| water | Electric: Live Wire, Dual Shock, Charging Station, Feedback, Thunder, Arc Bolt |
| toxic | Fire: Scorch, Inferno, Arsonist, Stoke, Fireworks, Blaze |
| steel | Placeholder generic (replace when Toxic enemy boards are designed against) |
| null/typeless | Strong generic board |

Front row Advanced (upgradeLevel 2), back row Intermediate-Advanced (upgradeLevel 1-2).

`rivalRecord` tracked in localStorage. `getDialogueBefore`/`getDialogueAfterLoss` on entry 6 are functions, not strings — `showEncounterBattle` and `endBattle` call them as functions.

---

## Layout

Single-page mobile portrait, no scroll, fixed viewport. Desktop at ≥600px uses `html { zoom: 0.8 }` to replicate 80% browser zoom.

```css
.encounter {
  flex: 1; min-height: 0; overflow: hidden;
  display: flex; flex-direction: column;
  padding: 8px 12px; gap: 6px;
  position: relative;  /* required for absolute watermark child */
}

/* Encounter HTML structure (top to bottom):
   #encTitle    — screen title text
   #encBody     — main content area
*/

/* Player section right-col layout (top to bottom):
   .zorp-row    — day/type/wins badges (24px)
   #counterStrip — burden countdown chips (hidden when empty)
   #dayTrack    — day track nodes
*/

.player-section {
  height: 362px;   /* mobile */
  flex-shrink: 0;
}
@media(min-width:600px) {
  html { zoom: 0.8; }
  .player-section { height: 402px; }
}

/* Event card status strip — positioned absolute at bottom of card */
.ecard .sts { position: absolute; bottom: 16px; left: 0; right: 0; z-index: 3; margin: 0; min-height: 0; }
```

ZORP watermark: position:absolute, top:4px, right:6px, left:auto inside `#encounterArea`, opacity:0.22, width:72px. Added by `startFromTitle()`.

---

## localStorage Keys

| Key | What it stores |
|-----|----------------|
| `zorpRivalName` | Persisted rival name across runs. Read at init, written by `confirmRivalName()`. |
| `zorpTutorialSeen` | `'true'` once tutorial completed or skipped. Read by `checkTutorialSeen()`, written by `markTutorialSeen()`. |
| `zorpDifficulty` | `'easy'` / `'normal'` / `'hard'`. Read at G init and in `startFromTitle()`. Written by `selectDifficulty()`. |
| `zorpBeatGame` | Set to `'1'` on win 6. Unlocks difficulty selector and 🎲 random monster on title screen. |
| `zorpWinstonFound` | Set to `'1'` when player wins with Winston in inventory. Unlocks 🪨 on title screen. |
| `zorpAchievementMinimalist` | Set to `'1'` when player wins with ≤5 items. No UI gate currently — stored for future use. |
| `rivalWins` | Cumulative rival win count (win 6 only). |
| `rivalLosses` | Cumulative rival loss count (loss at win 5 only). |

---

## Run / Game Flow Constants

- Honor: 13 (`G.honor`, `G.maxHonor`).
- Win target: 6 (game over check at `G.wins>=6`).
- Max days: ~11.
- Perfect run (6-0, `G.honor===13` at completion) → FLAWLESS screen with gold glow.
- Starting gold: 8.
- `DAY1_NODES` (5 nodes: mom, shop1, wild, shop2, rival) and `DAY2_NODES` (5 nodes: shop1, event, wild, event, rival). `getDayNodes()` returns `DAY2_NODES` after rival defeated and `G.day>1`.

---

## Post-Win Flow

After every trainer win (wins 1-5; not win 6):

```
postTrainerWin()
  ├─ G.wins===4 && hasWinston && !eventsCompleted.includes('monk_winston')
  │     └─ showMonkWinstonArc() → (return the rock → board picker → type/skill flow)
  │                              → (keep Winston → type/skill flow)
  ├─ G.type === null → showTypeSelect()
  │     ├─ Pick a type → remove Potential from activeSkills, overwrite treeState['head'],
  │     │                add type awakening to activeSkills → advanceDay() (no node pick)
  │     └─ Stay typeless → showBanScreen() if 2+ types non-banned, else showSkillPick()
  │           └─ banType() → showSkillPick()
  │                 └─ selectNode() → pickSkill() → advanceDay()
  └─ G.type !== null → showSkillPick() → selectNode() → pickSkill() → advanceDay()
```

Win 6 → game over. `goToTitleFromVictory()` calls `resetRun()` then `showTitleScreen()` (via CONTINUE button).

`advanceDay()` only increments the day and fires `activateCurrentNode()`. It does NOT trigger type select — that happens entirely in the post-win chain before `advanceDay` is called.

If `G.betPending` is true at the start of `advanceDay`, clears flag and skips the skill pick (The Bet burden consumed the pick).

### Second Wind

`pickSecondWindBuff(id)` → `showSecondWindItemSelect` if `requiresItemSelect`, else `applySecondWindBuff`.
`applySecondWindBuff(buff,targetItem)` switches on `buff.id`. All non-Clarity buffs call `advanceDay()` at end. Clarity sets `secondWindClarityPending=true` then `showSkillPick()`.
`secondWindClarityPending` checked at top of `advanceDay`: if true, clears flag, increments `dayStep`, calls `activateCurrentNode()` (no day++).
`renderSecondWindNode()` inserts/updates red badge in player header next to type badge — called from `updateUI`.
`G.steadfastActive` halves player fatigue damage in `startFatigue`.

---

## UI Notes

- **Toast duration minimum:** `showNotif` default `duration=5000ms`. All explicit durations throughout the codebase are also ≥5000ms.
- **Burden panels:** Never auto-dismiss. Require manual Continue tap. `_burdenAutoTimeout` exists but is never assigned in `showBurdenPanel`. All `clearTimeout(_burdenAutoTimeout)` calls are safe no-ops.
- **getDisplayRarity(item):** Returns effective rarity bumped by upgradeLevel (capped at mastered). Used by `getSellValue()`, `getShopItemCost()`, and card/badge rendering. Upgraded items display and sell at the higher rarity tier.
- **Pricing:** `TIER_COSTS={beginner:2,intermediate:4,advanced:6,mastered:8}` is the single source of truth for item buy prices. `UPGRADE_MULTS` is stats-only (effectAmt/maxHp scaling) — never used for pricing.
- **DAMAGE_RECEIVED event `rawValue`:** Pre-shield damage amount. Used by Shame for accurate tracking.
- **`.ecard .sts` CSS:** `position:absolute; bottom:16px; left:0; right:0; z-index:3` — status strip is absolutely positioned at the bottom of event cards.
- **`hasOpenSlot()`:** Checks `G.lockedSlots` before returning true — locked slots are not counted as open.

---

## Item Renames (display names)

Keys unchanged, only `name` field updated:

| Key | Display name |
|-----|--------------|
| `left_claws` | Hook |
| `spark_charm` | Surge |
| `cinder_claw` | Scorch |
| `ember_fang` | Inferno |
| `current_band` | Water Gun |
| `tide_fang` | Hydro Blast |
| `tidal_shell` | Coral |
| `bulwark_stance` | Reinforce |
| `alloy_fang` | Steel Jaw |

---

## Deleted — Do Not Reference

**Items removed entirely:** `toxic_cloud`, `rot_berry`, `corrode`, `ration_bag`, `basic_berry`, `iron_paw`, `hard_shell`, `berry_pouch`, `charge_pack`, `scorched_shell`, `flame_wick`, `iron_carapace`, `forge_core`, `volt_band`.

**Skill IDs removed:** `quick_reflex` (now `quick_reflexes`), `battle_hardened`, `dense_coat`, `pack_rhythm`, `apex_strike`, `alpha`, `unbreakable`, `static_field`, `chain_lightning`, `ember_coat`, `frost_aura`, `deep_freeze`, `plating_mastery`, `fortress`. (`wildfire` is now a Fire mastered typed skill, not the old generic.)

**Tree nodes removed:** `neck`, `spine1`, `spine2`, `spine3`, `front_upper`, `tail` (old).

**Types removed:** Ice (`id:'ice'`). Replace with water if seen.

**Functions removed:** `potMult()` — deleted; all heal scaling now uses `getSkillMods().globalMult`. `registerSkillHandlers()` — deleted; skill handlers are registered inline in `runBattle`.

**Events removed:** `cult_accusation` — no longer in the EVENTS array.

---

## Pending Work

- **Steel-typed Win 6 counter board** — currently a placeholder; replace when Toxic enemy boards are designed against.
- **Rival entry 6 dialogue/img polish** — `getDialogueBefore` / `getDialogueAfterLoss` exist as functions on the entry; check call sites stay correct as content evolves.
- **Burden authoring** — `tickBurdens`, `fireBurdenHandler`, and `resolveBurden` are complete. Six burden handlers exist. New burdens can be added by extending the `onResolve` switch.
- **Event content** — 39 events live (including 6 binding events). Effect handlers for all events are wired. Event flavour/dialogue text can be polished without architectural changes.
- **Quest content** — 5 quests in QUESTS array. All effect types wired in `resolveQuest`.

---

## What NOT to Change Without Discussion

- The `effectFn` three-argument signature.
- The `side` convention in damage functions (the `attackerMods` check).
- HEAD node pre-fill in init and `resetRun`.
- Post-win flow sequence — intentionally structured and fragile to reordering.
- `dealDmgWithStatus` applies status to the same target that was damaged (not a separate random target).
- Poison stack permanence — stacks are permanently permanent. `poisonStack` is never decremented anywhere; only `applyPoison` adds and `resetItemBattleState` zeroes. The tick loop deals `item.poisonStack` damage each tick but never reduces the stack.
- `G.metrics` fields — only incremented, never decremented during a run. Reset to zero in `resetRun()`.
- Skill bug fixes listed in "Confirmed Skill Bug Fixes" — do not revert them.
- `applySlotBonuses` must be called on every item placement path — shop buy, event placement, swap paths, mom gift. Omitting silently skips permanent slot bonuses.
- `G.lockedSlots` must be checked in every free placement loop. Omitting silently places items in locked slots.
- The `advance`/`done` closure pattern in `resolveEventChoice` — `currentEvent` is null by switch time; `event` and `choice` are captured as local consts before the null assignment and remain valid throughout.
- `excludeFlags` in event unlock objects can be **array** `['field']` or **object** `{field:true}`. `getEventPool` handles both. Do not normalise one format away.
- `showBurdenPanel` has no auto-dismiss timeout. Do not add one. Panels require a manual Continue tap.
- Slot numbering: front[0-2] = slots 1-3, back[0-2] = slots 4-6. The slot schedule is Day1=1F+1B, Day2=2F+1B, Day3=2F+2B, Day4=3F+2B, Day5+=3F+3B.
- The hermit `walk_away` path gives Winston, not `indulge`. Do not swap them.
- Burn tick is 2000ms. Poison tick base is 2000ms. Do not revert to 500ms/1000ms.
- `getFatigueTarget` hits ONE item per side per fatigue tick — not all items. Do not revert to all-items fatigue.
- `item._breakHandled` — set in `checkAndBreak` after first break processing. Prevents double-break within same battle tick. Reset to `false` in `resetItemBattleState`. Do not remove.
- `battleState.playerItems` / `battleState.enemyItems` — cached item arrays. Initialized at battle start in `runBattle`, refreshed in `checkAndBreak` after `item.broken = true`. Both must stay in sync.
- Binding system `_bindingFiring` guard — set on `partnerItem` before binding trigger, cleared after (or in timeout for `investor`). Prevents recursive binding triggers when bindings chain. Do not remove or weaken.
- `G.bindings` / `G.bindingCount` / `G.metrics.bindingMaxed` — the binding system is capped at 3 bindings per run enforced by `applyBinding`. `bindingMaxed` flag is used as `excludeFlags` on all binding events.
- `applySlotBonuses` `_slotBonusApplied` guard — prevents double-application on swap. Do not call `applySlotBonuses` on items that already have `_slotBonusApplied=true`.
