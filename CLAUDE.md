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

**Items use `handleEvent(event)`** for triggered effects. Passive multipliers stay in `getSkillMods()`. Skill handlers are registered as `sk._battleHandler` in `registerSkillHandlers()` from `runBattle`.

**`getNeighbors(item)`** returns `{side, row, index, left, right, behind, inFront}`.
**`item._side`** is set in `resetItemBattleState` — use this to know which board an item belongs to inside handlers.

### Item System

`getItemFactory(key)` looks up an item factory across all pools: `ITEMS, TYPED_ITEMS, PACIFIST_ITEMS, HEADBUTT_ITEMS, BURN_ITEMS, WATER_ITEMS, ELECTRIC_ITEMS, STEEL_ITEMS, POISON_ITEMS, MOM_ITEMS, MOM_CONSOLATION_ITEMS`.

`getItemsOfRarity(rarity, typeFilter)` flattens the first 9 pools (not MOM pools) into one map and filters by rarity (and optional itemType filter). Falls back to a lower rarity if none match. Typed items are mixed in — typed players see the same shop pool as typeless.

`makeItem(key, name, icon, tag, hp, actMs, desc, effectFn, sellOverride, ?, baseEffect, itemType, itemRarity)` is the constructor.

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
| `applyShield` | Lowest HP% ally (or specific `targetItem` if passed). |
| `applyShieldAll` | All alive allies with `maxHp > 0`. |
| `healItem` | Ally with lowest HP percentage (`hp < maxHp` filter). |
| `healAll` | All alive allies with `maxHp > 0`. |
| `dealDmg` | Leftmost front row enemy (or back if front empty). |
| `dealDmgRandom` | Random front row enemy (or back if front empty). |
| `dealDmgTwoRandom` | 2 random enemies, can hit same target twice. |
| `dealDmgBackRow` | Random back row enemy (falls back to front if back empty). |
| `dealDmgWithStatus` | Random front enemy + applies status to that same target. |

Items with `maxHp === 0` are excluded from targeting pools automatically. Use tag `'Untargetable'` not `'Food'`.

### Board Structure

```js
board       // {front:[3], back:[3]} — player
enemyBoard  // {front:[3], back:[3]} — enemy
```

Slot expansion (`maxFrontSlots()` / `maxBackSlots()`):

| Day | Front | Back |
|-----|-------|------|
| 1 | 1 | 1 |
| 2 | 2 | 1 |
| 3 | 2 | 1 |
| 4 | 3 | 2 |
| 5+ | 3 | 3 |

`G.neighborSlotBonus` adds extra slots beyond the day-gated cap (up to the hard max of 3 per row). `maxFrontSlots()` and `maxBackSlots()` both read this value.

### `applySlotBonuses(item, rowType)`

Called every time an item is placed onto the board (shop buy, event placement, swap paths, mom gift). Reads `G.slotBonuses[rowType]` and applies any accumulated `maxHp` or `damage` bonuses to the newly placed item. **Must be called on all placement paths** — omitting it silently skips permanent slot bonuses.

### `G.lockedSlots`

Array of `{row, idx}` objects. Slots in this list cannot be used for free placement. All placement loops must check `G.lockedSlots` before assigning. `pawnbroker_unlock` removes entries. Initialized to `[]` in `resetRun`.

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

`river_stone` — display name "Mom's Rock", tag "Definitely not Winston.", `isWinston:false`. It is not Winston.

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

`buildWildEnemyBoard(m)` sets up `enemyBoard` from a monster object and calls `applyDifficultyToBoard([...enemyBoard.front,...enemyBoard.back], G.day)`.

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
- `endurance` (×0.6 fatigue), `last_stand`, `iron_hide` (10s safety), `phoenix_downed`, `echo`, `waterproof`/`grounded`, `immunity` (poison interval 2000ms), `asbestos` (-2 burn stacks), `shatter` (enemy plating /2), `war_chest` (+gold/2 dmg), `desperation` (×1.5 effectAmt on each break — never mutates `baseEffect`), `predator` (×1.5 globalMult at 3+ enemy breaks), `flow_state` (×1.15 globalMult at 20s+).
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

---

## Monster System

### G State Fields

`G` is initialized at the top of the script and reset by `resetRun()`. Core fields:

```
gold:8, honor:13, maxHonor:13, wins:0, day:1, type:null, rivalDefeated:false,
dayStep:0, selectedSlot:null, battleRunning:false, lastResult:null,
wildChoice:null, pendingSkillNode:null, momGiftGiven:false,
monsterName:'Kip', monsterType:'fox', rivalName:'Rival', tutorialSeen:false,
secondWindUsed:false, secondWindBuff:null, steadfastActive:false,
consolationPending:false,
difficulty: localStorage.getItem('zorpDifficulty') || 'easy',
metrics: {
  wildWins:0, wildLosses:0, trainerWins:0,
  poisonApps:0, burnApps:0, itemsSold:0,
  totalDamageDealt:0, itemsLostInBattle:0, burdensTaken:0,
  eventsDeclined:[], eventsCompleted:[], npcsMet:[],
  winsWithoutHonorLoss:0, butcherDeclines:0,
  hasWinston:false, loanActive:false, oathTaken:false,
  wildLostToday:false
},
burdens: []
```

**Additional fields reset by `resetRun()` (not in initial literal):**

| Field | Type | Purpose |
|-------|------|---------|
| `G.forkBonusGold` | boolean | +1 gold on next wild win (fork_in_the_road left path). |
| `G.betPending` | boolean | Bet burden resolved — skip next skill pick. |
| `G.neighborSlotBonus` | number | Extra slots from The Generous Neighbor. Used by `maxFrontSlots()`/`maxBackSlots()`. |
| `G.inventory` | array | Non-board items (currently: Winston `{id, name, icon, tag, isWinston}`). Shown in `renderInventoryStrip()`. |
| `G.slotBonuses` | object | `{front:{maxHp,damage}, back:{maxHp,damage}}`. Accumulated from events (e.g. Oath gives back +35 maxHp). Applied on placement via `applySlotBonuses()`. |
| `G.nextEventIsWild` | boolean | Next event node triggers a wild fight instead of an event. Set by Gossip wild hint. |
| `G.lockedSlots` | array | `[{row,idx}]`. Slots blocked from free placement. Set by Pawnbroker, cleared by `pawnbroker_unlock` burden. |

**`G.metrics.wildLostToday`** — set true on wild loss, false on day advance. Used as `performance` filter for the Unimpressed Farmer event.

**`G.metrics.oathTaken`** — set true when the Oath burden is accepted. Used as `excludeFlags` filter (The Oath won't reappear) and `metricFlags` inclusion filter (Cult Accusation only appears if oathTaken).

**Metric increment sites:**
| Field | Where incremented |
|-------|-------------------|
| `wildWins` | `endBattle` — wild win path |
| `wildLosses` | `endBattle` — wild loss path |
| `trainerWins` | `endBattle` — trainer win path |
| `winsWithoutHonorLoss` | `endBattle` — trainer win path, only if `G.honor === G.maxHonor` |
| `burnApps` | `applyBurn` — when `attackerSide==='player'` and `item._side==='enemy'` |
| `poisonApps` | `applyPoison` — when `target._side==='enemy'` |
| `totalDamageDealt` | `applyDmgTo` — when `side==='enemy'` (player attacking), after HP reduction |
| `itemsLostInBattle` | `checkAndBreak` — when `side==='player'`, after `item.broken=true` |
| `itemsSold` | sell button handler |
| `eventsCompleted` | `resolveEventChoice` — choiceIdx===0 (accept) + specific push for 'priest' on upgrade_pick |
| `eventsDeclined` | `resolveEventChoice` — choiceIdx>0 (decline) |
| `butcherDeclines` | `case 'butcher_decline'` effect handler |
| `wildLostToday` | `endBattle` — wild loss; reset to false in `advanceDay()` |

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
getSkillMods()                  // returns {maxHpBonus, speedMult, backSpeedMult, dmgBonus, platingMult, fatigueMult, globalMult, slot2SpeedMult, slot5SpeedMult}
getSkillOffers(nodeRarity)
```

### `battleState` fields (initBattleState)

Generic / global:
`momentumBonus:0`, `fatigueDamageMultiplier:1`, `totalSidePlating:0`, `burnApplicationsToEnemy:0`, `fireworksThreshold:6`, `outbreakTriggered:false`, `slowProcsThisBattle:0`, `floodTriggered:false`, `wetDurationMult:1`, `joltCumulativeMs:0`, `thunderFired:false`, `poisonApplicationsToEnemy:0`, `phoenixDownAvailable:true`, `resilienceUsed:false`, `crescendoCount:0`, `crescendoReady:false`, `benedictionMs:0`, `kickstartCount:0`, `frictionCount:0`, `assembledActive:false`, `dualityApplied:false`, `ironHideActive:(skill active)`, `ironHideExpired:false`, `flowStateActive:false`, `predatorActive:false`, `warChestBonus:(Math.floor(G.gold/2) if active else 0)`.

Electric: `electricActivationCount:0`, `capacitorFired:false`, `thunderclapFired:false`, `dischargeCount:0`, `dischargeFired:false`, `overchargeCount:0`, `conductorBonuses:{}`.

Fire: `emberStormFired:false`, `hearthMs:0`, `activatingItem:null`.

Water: `wetSecondsAccumulated:0`, `deepCurrentBonus:0`, `tidalForceActive:false`, `maelstromMs:0`, `tsunamiFired:false`, `floodGateFired:false`.

Steel: `forgeTick:0`, `pressurePlateFired:false`, `platingAbsorbedTotal:0`, `ironFormationActive:false`, `impenetrableUsed:{}`, `bulwarkUsed:{}`.

Toxic: `virulenceCount:0`, `seepingHealMs:0`, `tippingPointFired:false`, `criticalMassFired:false`, `necrosisMs:0`, `pandemicActive:false`, `colonySide:false`, `lastKillWasPoison:false`.

### `resetItemBattleState` (line ~3550)

Per-item resets: `hp=maxHp, shield=0, maxShield=0, actElapsed=0, actPct=0, broken=false, uses=maxUses, hasteMs=0, slowMs=0, _slowMaxMs=0, burnStack=0, burnTickMs=0, poisonStack=0, poisonTickMs=0, _side=('player'|'enemy')`.

Also runs second passes for `apotheosis`, `refrigeration`, `duality` (with `dualityApplied` flag), `assembled`, `iron_formation` (sets `ironFormationActive` if all 3 front filled).

### Status Effects

| Status | Field | Tick interval | Damage rule |
|---|---|---|---|
| Burn | `burnStack`, `burnTickMs` | 500ms | `shield × 2` absorbs, remainder hits HP. `burnStack--` per tick. |
| Poison | `poisonStack`, `poisonTickMs` | 1000ms (2000 with Immunity, 900 enemy w/ Festering) | Bypasses shield. Poison stacks are permanently permanent. `poisonStack` is never decremented anywhere in the codebase. The only writes to `poisonStack` are: `applyPoison` (adds stacks) and `resetItemBattleState` (zeroes at battle start). The tick loop deals `item.poisonStack` damage each tick but never reduces the stack. Stacks accumulate indefinitely until battle ends. |
| Haste | `hasteMs` | Countdown | Speed multiplier 2.0 (faster activation). |
| Slow | `slowMs` | Countdown | Speed multiplier 0.5 (slower activation). |

Haste and slow cancel when both active (net 1.0×). `applyHaste` uses `Math.max`; `applySlow` uses additive `+` (Wet stacks duration).

### `applyShield` behavior

- Default target: lowest HP% ally (filtered to `maxHp > 0`). This is intentional, not a regression.
- Optional 4th param `targetItem`: if provided and not broken, shield that specific item.
- `ITEM_SHIELDED` event no longer fired here — `cinder_block` etc fire their own.
- `totalSidePlating` incremented for player side.

### onBreak pattern

`item.onBreak(item)` is called after `item.broken=true` in all break paths: `checkAndBreak`, `applyDmgTo` hitCount path, `tickSide` uses-depletion, burn-hitCount, poison-hitCount. Burn/poison HP paths go through `checkAndBreak`. Currently used by Wrecking Ball.

### Special item mechanics

- **hitCount items:** Training Dummy uses `hitCount` instead of HP. `targetable()` checks `isTargetable===true` as override. `applyDmgTo` checks `hitCount` before shield logic and absorbs damage entirely. Burn and poison decrement hitCount per tick.
- **Technique items:** `isTechnique=true`. Bought as tomes, consumed on buy, placed as passive items. Show tome modal on purchase. Sell via `showTechSellModal`. Use `handleEvent` for all effects.
- **Chrysalis:** 1 HP, 1s activation, gains plating each tick. Breaks instantly from any damage.
- **Headbutt:** Deals damage to leftmost enemy front, then takes half as self-damage. Self-damage fires full `DAMAGE_RECEIVED` events.
- **Ricochet:** No HP/timer. `handleEvent` on `DAMAGE_RECEIVED` fires when item directly behind takes damage. Deals 3 to random enemy.
- **Arsonist:** No timer. `handleEvent` on `BATTLE_START` applies 5 burn to every non-broken item on both boards. 160 HP — can be killed.
- **Spite:** 50hp, 1s activation. Uses `_spiteCounter` (not effectAmt). `handleEvent` on `DAMAGE_RECEIVED` (matching side, not self) increments counter by 1. Resets to 1 in `resetItemBattleState`.
- **Venom Fang:** Position-aware. If `self===board.front[0]` or `enemyBoard.front[0]`, uses 3s timer (sets `actMs` and `baseActMs`); otherwise 4s. Applies 1 poison to random front enemy.
- **`battleState.fatigueDamageMultiplier`:** Default 1, set by `Patience` on `FATIGUE_START`. Applied to enemy fatigue damage in `startFatigue`.

### Poison mechanic — important

Poison stacks are permanently permanent. `poisonStack` is never decremented anywhere in the codebase. The only writes to `poisonStack` are: `applyPoison` (adds stacks) and `resetItemBattleState` (zeroes at battle start). The tick loop deals `item.poisonStack` damage each tick but never reduces the stack. Stacks accumulate indefinitely until battle ends.

`applyPoison` order: Pandemic (`stacks *= 2`, enemy targets only) → apply → Virulence (every 3rd player-applied application to enemy, +1 extra) → `poisonApplicationsToEnemy++` (enemy targets) → `checkOutbreak()` → Outbreak extra +1 (if triggered) → fireEvent → log.

---

## Run Event System

Separate from the battle event bus. Handles narrative events on the day track between locations.

### EVENTS Array — 23 entries

Event schema: `{id, name, tier, image, unlock:{minDay,maxDay,minWins,maxWins,excludeFlags,performance,minHonor,metricFlags}, validSlots, state, prompt:{title,flavor,dialogue}, choices:[{id,label,cost,requiresCost,effect,burden,outcomeText}], resolution, tags, easter_egg}`

- `tier` — `'beginner'` | `'intermediate'` | `'advanced'`. Controls frequency via `TIER_WEIGHTS`.
- `validSlots` — `['pre_wild']` | `['post_wild']` | `null` (any). Restricts to slot context.
- `state` — `'boardNotFull'` blocks event when all 6 slots are filled. `null` = no state check.
- `tags` — `'free'` (Slot A), `'paid'`/`'burden'`/`'forced'` (Slot B), or mixed.
- `resolution` — `'immediate'` (player picks) | `'automatic'` (no choice, fires immediately).
- `outcomeText` — string shown in `showBurdenPanel` after an immediate/null-effect choice before advancing. `null` = advance directly.
- `requiresCost:false` with a `cost` — cost is taken regardless of availability (forced payment, e.g. Tax Collector).

| ID | Name | Tier | Notes |
|----|------|------|-------|
| `the_rock` | The Rock | beginner | Free imbue pick: +5 dmg or +10 HP to one item. |
| `sharpening_stone` | The Sharpening Stone | beginner | Free +3 dmg, or 2 gold for +7 dmg, to one item. |
| `tax_collector` | The Tax Collector | intermediate | Forced 2 gold (min of 2 or current gold), gives +3 dmg to all items. |
| `fork_in_the_road` | The Fork In The Road | beginner | pre_wild only. Left: +1 gold on next wild win. Right: skip wild, choose typed shop. |
| `butcher` | The Butcher | intermediate | Accept: +6 gold + butcher_loan burden (repay 7 in 15 locations). Decline: tracks butcherDeclines. |
| `the_bet` | The Bet | advanced | Accept: all items hit softer (bet_debuff burden, restores in 10 locations) + free any-node skill pick after. Decline: nothing. |
| `the_kid` | The Kid | beginner | 2 gold: random imbue (random type) on one random item. |
| `old_mans_cart` | The Old Man's Cart | beginner | Free: places a random item (respects lockedSlots, calls applySlotBonuses). Board-full: show swap picker. |
| `unimpressed_farmer` | The Unimpressed Farmer | beginner | post_wild only, requires `wildLostToday`. Free: upgrade_weapon_pick on one item. |
| `generous_neighbor` | The Generous Neighbor | beginner | pre_wild only, requires board not full. Automatic: unlocks one extra slot (neighborSlotBonus+1). |
| `the_cartographer` | The Cartographer | intermediate | Free: reward based on NPC variety (npcsMet count) — gold + possible imbue. |
| `the_hermit` | The Hermit | intermediate | Indulge: adds Winston to G.inventory, sets hasWinston. Walk away: minor buff (upgrade random item). |
| `the_blacksmith` | The Blacksmith | intermediate | Free: she picks and upgrades one item. 4 gold: player picks which item to upgrade. |
| `merchants_daughter` | The Merchant's Daughter | intermediate | 2 gold entry: browse 3 randomised items at reduced/random prices. Board-full: swap picker + discard option. |
| `the_appraiser` | The Appraiser | intermediate | Free: sell one item for 150% of normal sell price. |
| `the_oath` | The Oath | intermediate | Excludes if oathTaken. Accept: debuffs back row effectAmt + oath_resolve burden (restores + gives back +35 maxHp after 15 locations). |
| `the_pawnbroker` | The Pawnbroker | intermediate | Deal: gold for a locked slot (pawnbroker_unlock burden returns it). |
| `the_surgeon` | The Surgeon | intermediate | Requires minHonor:5. 4 honor: upgrade one item (checked after viability guard — no honor loss if all maxed). |
| `the_gambler` | The Gambler | intermediate | Throw: -3 honor, all item HP halved (gambler burden, restores after 2 battles). Refuse: nothing. |
| `the_collector` | The Collector | intermediate | Trade: sell worst item, get a random item of higher rarity. |
| `the_gossip` | The Gossip | intermediate | Three choices: clue (show hint panel), damage (+dmg imbue + gossip_resolve burden), wild (nextEventIsWild=true). |
| `the_stranger` | The Stranger | beginner | pre_wild only. Peek (free): shows wild pool options. Peek+swap (3 gold): shows pool + lets player sell an item and buy a beginner replacement. |
| `cult_accusation` | A Familiar Look | advanced | Requires oathTaken. Flavour only — deny or admit, no mechanical effect. |

### Easter Egg — The Priest

Not in `EVENTS` array. Lives in `EASTER_EGGS.priest`. Triggered by `checkEasterEggs()`.

- **Trigger:** `G.metrics.butcherDeclines >= 2`. On trigger, resets `butcherDeclines = 0` so it can re-trigger later.
- **Effect:** `upgrade_pick` — player picks any item to upgrade one level.
- **eventsCompleted:** `resolveEventChoice` pushes `'priest'` (short form, not `'the_priest'`) when `event.id==='the_priest'` inside `case 'upgrade_pick':`.
- **Gossip clue hint[1]** checks `eventsCompleted.includes('priest')` to reveal the "tell the Butcher no twice" hint.

### Event System Functions

```js
getEventPool(day, slotCtx='any')
  // Filters EVENTS by 8 criteria:
  // 1. validSlots — if set, slotCtx must be in the array
  // 2. state === 'boardNotFull' — blocks if all 6 slots filled
  // 3. unlock.minDay / maxDay
  // 4. unlock.minWins / maxWins
  // 5. unlock.excludeFlags — {field:true} blocks if G.metrics[field] is truthy
  // 6. unlock.performance — event only appears if G.metrics[performance] is truthy
  // 7. unlock.minHonor — blocks if G.honor < minHonor
  // 8. unlock.metricFlags — all listed fields must be truthy in G.metrics

pickEvent(day)
  // Legacy single-event weighted pick (beginner=3×, intermediate=2×, advanced=1×).
  // Used by the old showEvent path. Returns null if pool empty.

pickEvents(day, slotCtx='any', count=3)
  // 3-slot picker system. Uses TIER_WEIGHTS[day] for weighted selection.
  // Slot A — guaranteed free event (tag:'free'). Falls back to any eligible.
  // Slot B — guaranteed paid/burden/forced event. Falls back to any eligible.
  // Slot C — 10% chance WILD_FIGHT_OPTION (pre_wild only), else any eligible.
  // Never repeats the same event ID in one picker. Returns array of up to 3 events.

showEventPicker(events)
  // Renders a card picker UI. Player clicks one event card to select it.
  // Selected event becomes currentEvent; shows its full prompt + choices.

resolveEventChoice(idx)
  // Called with choice index. Captures event and choice before clearing currentEvent.
  // Defines advance() and done() closures:
  //   advance() = G.dayStep++; updateUI(); renderTrack(); activateCurrentNode()
  //   done() = choice.outcomeText ? showBurdenPanel(title, outcomeText, advance) : advance()
  // Cost check (gold/honor deduction) happens before the effect switch.
  // Effect switch uses done() instead of raw advance() for immediate/null effects.
  // Pushes to eventsCompleted (idx===0 accept) or eventsDeclined (idx>0 decline).
```

`TIER_WEIGHTS` — object keyed by day (1–10). Each entry: `{beginner, intermediate, advanced, mastered}` relative weights. Day 1: beginners only. By day 8+: mastered 50%, advanced 50%.

`WILD_FIGHT_OPTION` — special constant `{id:'bonus_wild_fight', name:'Something Stirs', tags:['wild'], choices:[{investigate},{keep_moving}]}`. Slot C wildcard, pre_wild only.

`currentEvent` — module-level. Holds the active event during player choice. Set to `null` at start of `resolveEventChoice` (before the switch). The local `const event` and `const choice` captured before the null assignment remain in scope throughout.

`showEvent(null)` — gracefully skips: `G.dayStep++; updateUI(); renderTrack(); activateCurrentNode()`.

### Burden System

`G.burdens` — array of active burden objects. `tickBurdens()` is called at `runBattle()` start and decrements `countdown`; when it hits 0, calls `fireBurdenHandler(burden, onDone)`.

`fireBurdenHandler(burden, onDone)` — removes burden from `G.burdens`, then switches on `burden.onResolve`:

| onResolve | Effect |
|-----------|--------|
| `butcher_collect` | Deducts repayAmount (default 7) from G.gold (min 0). Shows panel. Sets `loanActive=false`. |
| `bet_resolve` | Restores original effectAmts from payload. Sets `G.betPending=true` (skips next skill pick). Shows panel. |
| `oath_resolve` | Restores original back row effectAmts. Applies +35 maxHp to current back items. Sets `G.slotBonuses.back.maxHp+=35`. Shows panel. |
| `pawnbroker_unlock` | Removes matching entry from `G.lockedSlots`. Shows panel "The slot is yours again." |
| `gambler_resolve` | Restores original maxHp values from payload. Shows panel. |
| `gossip_resolve` | Restores original effectAmts from payload. Calls `onDone()` directly (no panel). |
| *(default)* | Calls `onDone()` directly. |

`showBurdenPanel(title, bodyText, onDone)` — renders a panel with Continue button and 2500ms auto-advance timeout. Stores `onDone` in `_burdenContinue`.

`resolveBurden(burdenId)` — removes burden by id, calls `renderCounterStrip()`. For manual resolution (not countdown-based).

### Counter Strip UI

`<div id="counterStrip">` — positioned between `encTitle` and `encBody`. Displays compact run stats.

`renderCounterStrip()` — called from `updateUI()`. Shows: `⚔️ wildWins W`, `🏆 trainerWins W`, `💀 itemsLostInBattle` (if >0), `🔗 N burden(s)` (if any). Hides element when `G.metrics` is absent.

### slotCtx — Event Slot Context

Computed in `activateCurrentNode()` when routing to an event node:

```js
const slotCtx = wildIdx < 0 ? 'any' : G.dayStep < wildIdx ? 'pre_wild' : 'post_wild';
```

- `pre_wild` — this event node appears before the wild fight in the day track.
- `post_wild` — this event node appears after the wild fight.
- `any` — no wild fight in this day's nodes (DAY1_NODES, which has no event nodes).

`getEventPool` and `pickEvents` both accept `slotCtx`. Events with `validSlots:['pre_wild']` only appear before the wild fight; `validSlots:['post_wild']` only after.

If `G.nextEventIsWild` is true, `activateCurrentNode()` skips the event entirely and calls `showEncounterWildPick()` instead, then clears the flag.

---

## Winston Arc

### The Hermit → Inventory

`the_hermit` event (intermediate, days 2–9). Two choices:

- **Indulge** (`hermit_indulge`): Adds `{id:'winston', name:'Winston', icon:'🪨', tag:'He knew everything.', isWinston:true}` to `G.inventory`. Sets `G.metrics.hasWinston=true`. Pushes `'hermit'` to `npcsMet`.
- **Walk away** (`hermit_walk_away`): Minor item upgrade. Pushes `'hermit'` to `npcsMet`.

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
- Appends 🪨 and `"Thank you for saving me."` plus a random `WINSTON_WISDOM` line to the victory screen subtitle.

`WINSTON_WISDOM` — array of 10 lines, defined before `EASTER_EGGS`. Selected with `Math.floor(Math.random()*WINSTON_WISDOM.length)`.

### Title Screen — zorpWinstonFound

`showTitleScreen()` checks `localStorage.getItem('zorpWinstonFound')`. If set, renders a 🪨 element (fixed bottom-left, z-index:201) that calls `showWinstonPanel()` on click.

`showWinstonPanel()` — creates and appends a self-contained modal div (z-index:300, covers full screen). Displays `"He knew everything."` Close button removes the div. Uses `document.createElement` because `showBurdenPanel` writes to the encounter body which is behind the title overlay.

---

## Day Structure

`DAY1_NODES` (5 nodes): mom → shop1 → wild → shop2 → rival

`DAY2_NODES` (5 nodes): shop1 → event → wild → event → rival

`getDayNodes()` returns `DAY2_NODES` when `G.rivalDefeated && G.day > 1`, otherwise `DAY1_NODES`.

`activateCurrentNode()` routes on `node.id`:
- `'mom'` → `showEncounterShop(0)`
- `'shop1'` → `showEncounterShop(1)`
- `'shop2'` → `showEncounterShop(2)`
- `'wild'` → `showEncounterWildPick()`
- `'event'` → computes `slotCtx`, checks `G.nextEventIsWild` (if true: clear flag, call `showEncounterWildPick()`), else `checkEasterEggs()` then `showEventPicker(pickEvents(G.day, slotCtx))`
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
- Scruffling art bottom-left, Thornback art bottom-right (decorative)
- 🪨 Winston rock (fixed bottom-left, z-index:201) — only shown when `localStorage.getItem('zorpWinstonFound')` is set; calls `showWinstonPanel()` on click

`startFromTitle()` — reads `zorpDifficulty` from localStorage into `G.difficulty`, removes the title screen overlay, shows a small logo watermark bottom-right, then calls `showProfessorFlow()`.

`selectDifficulty(d)` — writes `zorpDifficulty` to localStorage and updates pill border/background styles in place.

`showWinstonPanel()` — self-contained modal (createElement, z-index:300). Shows `"He knew everything."` with a Close button.

---

## Professor Flow

Module-level: `professorFlowActive` (boolean) and `professorStep` (0-6 counter).

```
showProfessorFlow()              // sets professorFlowActive=true, professorStep=0, calls showProfessorStep
  └ showProfessorStep()          // switch on professorStep:
      0 → showProfessorIntro()
      1 → showProfessorShop()    // → professorShopBrowse() or professorShopDecline()
      2 → showMonsterSelection() // FOX vs HYDRA
      3 → showMonsterNaming()
      4 → if checkTutorialSeen() → step=5 + recurse
          else                  → showTutorial()
      5 → showRivalNaming()      // → confirmRivalName()
      6 → beginRun()             // sets professorFlowActive=false
```

`profLayout(dialogueHTML, extraHTML)` wraps content with the 90px professor portrait on the left and content on the right. Used on every professor screen.

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
body {
  position: fixed; inset: 0;
  overflow: hidden;
  display: flex; flex-direction: column;
  background: var(--bg);
  color: var(--text);
  font-family: 'Crimson Pro', serif;
  background-image: radial-gradient(...);
}

.encounter {
  flex: 1;
  min-height: 0;
  overflow: hidden;
  display: flex; flex-direction: column;
  padding: 8px 12px;
  gap: 6px;
}

/* Encounter HTML structure (top to bottom):
   #encTitle  — screen title text
   #counterStrip — run stats bar (flex, display:none until populated)
   #encBody   — main content area
*/

.player-section {
  height: 362px;          /* mobile */
  flex-shrink: 0;
  overflow: hidden;
  display: flex; flex-direction: column;
}
@media(min-width:600px) {
  html { zoom: 0.8; }
  .player-section { height: 402px; }
}

.board-row {
  flex: 1; min-height: 0;
  display: flex; align-items: center;
  gap: 4px;
}

.slots {                 /* used by #frontSlots and #backSlots */
  flex: 1; min-width: 0;
  height: 100%;
  display: flex; gap: 4px;
}
```

The encounter expands to fill available space above the fixed-height player section. Front and back rows share the player section via two `.slots` containers inside `.board-row` rows.

---

## localStorage Keys

Seven keys are used:

| Key | What it stores |
|-----|----------------|
| `zorpRivalName` | Persisted rival name across runs. Read at init, written by `confirmRivalName()`. |
| `zorpTutorialSeen` | `'true'` once tutorial completed or skipped. Read by `checkTutorialSeen()`, written by `markTutorialSeen()`. |
| `zorpDifficulty` | `'easy'` / `'normal'` / `'hard'`. Read at G init and in `startFromTitle()`. Written by `selectDifficulty()`. |
| `zorpBeatGame` | Set to `'1'` on win 6. Unlocks difficulty selector on title screen. |
| `zorpWinstonFound` | Set to `'1'` when player wins with Winston in inventory. Unlocks 🪨 on title screen. |
| `rivalWins` | Cumulative rival win count. |
| `rivalLosses` | Cumulative rival loss count. |

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

Win 6 → game over. `continueAfterResult` calls `resetRun()` when `G.wins>=6`.

`advanceDay()` only increments the day and fires `activateCurrentNode()`. It does NOT trigger type select — that happens entirely in the post-win chain before `advanceDay` is called.

If `G.betPending` is true at the start of `advanceDay`, clears flag and skips the skill pick (The Bet burden consumed the pick).

### Second Wind

`pickSecondWindBuff(id)` → `showSecondWindItemSelect` if `requiresItemSelect`, else `applySecondWindBuff`.
`applySecondWindBuff(buff,targetItem)` switches on `buff.id`. All non-Clarity buffs call `advanceDay()` at end. Clarity sets `secondWindClarityPending=true` then `showSkillPick()`.
`secondWindClarityPending` checked at top of `advanceDay`: if true, clears flag, increments `dayStep`, calls `activateCurrentNode()` (no day++).
`renderSecondWindNode()` inserts/updates red badge in player header next to type badge — called from `updateUI`.
`G.steadfastActive` halves player fatigue damage in `startFatigue`.

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

**Functions removed:** `potMult()` — deleted; all heal scaling now uses `getSkillMods().globalMult`.

---

## Pending Work

- **Steel-typed Win 6 counter board** — currently a placeholder; replace when Toxic enemy boards are designed against.
- **Rival entry 6 dialogue/img polish** — `getDialogueBefore` / `getDialogueAfterLoss` exist as functions on the entry; check call sites stay correct as content evolves.
- **Burden authoring** — `tickBurdens`, `fireBurdenHandler`, and `resolveBurden` are complete. Six burden handlers exist. New burdens can be added by extending the `onResolve` switch.
- **Event content** — 23 events live. Effect handlers for all events are wired. Event flavour/dialogue text can be polished without architectural changes.

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
