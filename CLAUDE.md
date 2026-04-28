# ZORP — Technical Brief for Claude Code Sessions

## Game Overview

Single self-contained HTML file game. All game logic, CSS, and HTML live in `index.html`. No build step, no dependencies, no external files. Open in browser to play.

- Hosted on GitHub Pages from this repo.
- Target device: mobile portrait (with desktop media-query upscaling at 600px+).
- Run length: 6 wins to victory. Honor pool 13. Max ~11 days. Perfect run (6-0, honor 13) → FLAWLESS screen.
- Board: 3 front × 3 back = 6 slots max.

---

## Architecture

### Event System

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

Slot expansion (`maxFrontSlots()` / `maxBackSlots()` at line 1905-1906):

| Day | Front | Back |
|-----|-------|------|
| 1 | 1 | 1 |
| 2 | 2 | 1 |
| 3 | 2 | 1 |
| 4 | 3 | 2 |
| 5+ | 3 | 3 |

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

`TYPED_ITEM_POOLS` constant exists at line 981 as a per-type listing — currently used as a reference, not as a shop filter. Toxic pool also lists `caustic_mantle, mithridate, miasma, outbreak` (which actually live in `POISON_ITEMS`).

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

`getSkillOffers(nodeRarity)` (line 4056):

- Typed player **with available typed skills at this rarity** → 3 offers: slot 1 always typed; slots 2-3 each 25% chance typed (75% generic).
- Typeless player or empty typed pool → 3 generic offers.
- `selectNode` shows 3 skill choices.

### Skill Implementation Wiring

All 60 typed skills are wired to battle effects. Generic skills are wired in `getSkillMods()` (passive multipliers), `initBattleState`, `resetItemBattleState`, `tickSide`, `runBattle` interval, `checkAndBreak`, and the action functions. Specific notes:

**Generic — notable:**
- `potential` — +25% all effects passive bonus (typeless only).
- `thick_fur` (+20 maxHp), `quick_reflexes` (×0.90 speed), `scrappy` (+6 dmg), `fortified` (+7 plating, scales with globalMult).
- `resilience` — first DAMAGE_RECEIVED grants +70 plating (`resilienceUsed`).
- `refrigeration` — +1 use to all provisions in `resetItemBattleState`.
- `momentum` (+4 per player activation), `crescendo` (5th activation ×2 effectAmt), `epicenter` (slot2/slot5 ×0.75 speed), `assembled` (×0.82 speed when 6/6 filled), `duality` (odd slots +10 effect, even +25 maxHp), `kickstart`/`friction` (slot 6 jolt/wet, max 3×), `benediction` (15 HP back row every 5s).
- `endurance` (×0.6 fatigue), `last_stand`, `iron_hide` (10s safety), `phoenix_downed`, `echo`, `waterproof`/`grounded`, `immunity` (poison interval 2000ms), `asbestos` (-2 burn stacks), `shatter` (enemy plating /2), `war_chest` (+gold/2 dmg), `desperation` (×1.5 effect on each break), `predator` (×1.5 globalMult at 3+ enemy breaks), `flow_state` (×1.15 globalMult at 20s+).
- `apotheosis` — every player item with baseEffect gets effectAmt = round(baseEffect × UPGRADE_MULTS[3]).
- `fizzle` — halves effectAmt and baseEffect of all enemy untargetable passives.

**Electric — wired in `tickSide`, `applyHaste`, `applyBurn`, `dealDmg*`, `checkAndBreak`.**

**Fire — wired in `applyBurn` (Kindle), `dealDmg*`, `tickSide` burn ticks, `checkAndBreak`, `runBattle` interval.**

**Water — wired in `applySlow` (Saturation), `dealDmg*`, `tickSide` enemy activation, `checkAndBreak`, `getSkillMods`, `runBattle` interval.**

**Steel — wired in `applyDmgTo` (Temper, Cold Iron, Pressure Plate, Bulwark, Impenetrable, Iron Formation), `dealDmg*`, `tickSide` player activation, `getSkillMods`, `runBattle` interval (Forge), `resetItemBattleState` (Iron Formation init).**

**Toxic — wired in `applyPoison` (Pandemic, Virulence, Outbreak), `tickSide` poison ticks (Festering, Seeping, Tipping Point trigger), `dealDmg*`, `checkAndBreak`, `runBattle` interval (Necrosis).**

---

## Monster System

### G State Fields

`G` is initialized at line 812 with:

```
gold:8, honor:13, maxHonor:13, wins:0, day:1, type:null, rivalDefeated:false,
dayStep:0, selectedSlot:null, battleRunning:false, lastResult:null,
wildChoice:null, pendingSkillNode:null, momGiftGiven:false,
monsterName:'Kip', monsterType:'fox', rivalName:'Rival', tutorialSeen:false,
secondWindUsed:false, secondWindBuff:null, steadfastActive:false,
consolationPending:false
```

- `G.monsterType` — `'fox'` or `'hydra'`. Set during professor flow.
- `G.monsterName` — player-named, defaults `'Kip'` (fox) or `'Hydrax'` (hydra).
- `G.rivalName` — player-named, persists in localStorage `'zorpRivalName'`. Default `'Rival'`.
- `G.tutorialSeen` — module flag (also mirrored via `localStorage 'zorpTutorialSeen'` and read via `checkTutorialSeen()`).

Module-level state outside G: `bannedTypes[]`, `rivalRecord{wins,losses}` (localStorage-backed), `lastOfferedTypes`, `board`, `enemyBoard`, `pendingSecondWindBuff/Row/Idx`, `secondWindClarityPending`, `professorFlowActive`, `professorStep`.

### Tree Nodes

`FOX_TREE_NODES` (lines 370-377) — 6 nodes:
- `head` (prestige, parent null) — pre-filled with `potential`.
- `front_shoulder` (beginner, parent head).
- `back_shoulder` (intermediate, parent front_shoulder).
- `front_foot` (intermediate, parent front_shoulder).
- `back_foot` (advanced, parent back_shoulder).
- `tail_tip` (mastered, parent back_shoulder).

Typed players skip `front_shoulder` and fill 4 nodes. Typeless fills all 5 non-HEAD nodes.

`HYDRA_TREE_NODES` (lines 378-385) — 6 nodes:
- `body` (prestige, parent null).
- `neck_split` (beginner, parent body).
- `mid_left` (beginner, parent neck_split).
- `mid_right` (beginner, parent neck_split).
- `head_left` (mastered, parent mid_left).
- `head_right` (mastered, parent mid_right).

Helpers:
- `getTreeNodes()` (line 386) — returns the array for current `G.monsterType`.
- `getSkillTreeSVGContent()` (line 703) — returns SVG markup for current monster.
- `rebuildSkillTreeSVG()` (line 763) — rebuilds SVG, resets `treeState`, syncs prestige root.
- `getAvailableNodes()` — returns nodes with parent filled and self empty (excluding HEAD).

`treeState` — `{nodeId: skill object or null}`. `activeSkills` — array of currently active skill objects.

### Image Constants — all populated (Cloudinary URLs)

Fox: `FOX_DEFAULT, FOX_ELECTRIC, FOX_FIRE, FOX_WATER, FOX_STEEL, FOX_TOXIC` (lines 846-851).
Hydra: `HYDRA_DEFAULT_IMG, HYDRA_STEEL, HYDRA_ELECTRIC, HYDRA_TOXIC, HYDRA_FIRE, HYDRA_WATER` (lines 854-859).
`PROFESSOR_IMG` (line 853).

`getFoxImage()` returns the right image for both fox and hydra typed variants. `updateFoxImage()` checks `professorFlowActive` to hide art during the professor flow.

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

### `battleState` fields (initBattleState, line 590-655)

Generic / global:
`momentumBonus:0`, `fatigueDamageMultiplier:1`, `totalSidePlating:0`, `burnApplicationsToEnemy:0`, `fireworksThreshold:6`, `outbreakTriggered:false`, `slowProcsThisBattle:0`, `floodTriggered:false`, `wetDurationMult:1`, `joltCumulativeMs:0`, `thunderFired:false`, `poisonApplicationsToEnemy:0`, `phoenixDownAvailable:true`, `resilienceUsed:false`, `crescendoCount:0`, `crescendoReady:false`, `benedictionMs:0`, `kickstartCount:0`, `frictionCount:0`, `assembledActive:false`, `dualityApplied:false`, `ironHideActive:(skill active)`, `ironHideExpired:false`, `flowStateActive:false`, `predatorActive:false`, `warChestBonus:(Math.floor(G.gold/2) if active else 0)`.

Electric: `electricActivationCount:0`, `capacitorFired:false`, `thunderclapFired:false`, `dischargeCount:0`, `dischargeFired:false`, `overchargeCount:0`, `conductorBonuses:{}`.

Fire: `emberStormFired:false`, `hearthMs:0`, `activatingItem:null`.

Water: `wetSecondsAccumulated:0`, `deepCurrentBonus:0`, `tidalForceActive:false`, `maelstromMs:0`, `tsunamiFired:false`, `floodGateFired:false`.

Steel: `forgeTick:0`, `pressurePlateFired:false`, `platingAbsorbedTotal:0`, `ironFormationActive:false`, `impenetrableUsed:{}`, `bulwarkUsed:{}`.

Toxic: `virulenceCount:0`, `seepingHealMs:0`, `tippingPointFired:false`, `criticalMassFired:false`, `necrosisMs:0`, `pandemicActive:false`, `colonySide:false`, `lastKillWasPoison:false`.

### `resetItemBattleState` (line 3387)

Per-item resets: `hp=maxHp, shield=0, maxShield=0, actElapsed=0, actPct=0, broken=false, uses=maxUses, hasteMs=0, slowMs=0, _slowMaxMs=0, burnStack=0, burnTickMs=0, poisonStack=0, poisonTickMs=0, _side=('player'|'enemy')`.

Also runs second passes for `apotheosis`, `refrigeration`, `duality` (with `dualityApplied` flag), `assembled`, `iron_formation` (sets `ironFormationActive` if all 3 front filled).

### Status Effects

| Status | Field | Tick interval | Damage rule |
|---|---|---|---|
| Burn | `burnStack`, `burnTickMs` | 500ms | `shield × 2` absorbs, remainder hits HP. `burnStack--` per tick. |
| Poison | `poisonStack`, `poisonTickMs` | 1000ms (2000 with Immunity, 900 enemy w/ Festering) | Bypasses shield. `poisonStack--` only inside the tick loop after dealing dmg. Stacks otherwise persist (no decay outside tick). |
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

`poisonStack` is **only decremented inside the tick loop in `tickSide`** (one stack per tick after damage is dealt). It is not decremented anywhere else. Outside ticking, poison stays. Stack only resets to 0 in `resetItemBattleState` at battle start.

`applyPoison` order: Pandemic (`stacks *= 2`) → apply → Virulence (every 3rd application, +1 extra) → `poisonApplicationsToEnemy++` (enemy targets) → `checkOutbreak()` → Outbreak extra +1 (if triggered) → fireEvent → log.

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
| steel | Placeholder generic (replace when Toxic items implemented) |
| null/typeless | Strong generic board |

Front row Advanced (upgradeLevel 2), back row Intermediate-Advanced (upgradeLevel 1-2).

`rivalRecord` tracked in localStorage. `getDialogueBefore`/`getDialogueAfterLoss` on entry 6 are functions, not strings — `showEncounterBattle` and `endBattle` call them as functions.

### Trainer Image Constants

`TRAINER_RIVAL_IMG, TRAINER_INVESTOR_IMG, TRAINER_ARCHITECT_IMG, TRAINER_MONK_IMG, TRAINER_MIRROR_IMG` — populated (lines 860-864). `TRAINER_RIVAL_RETURNS_IMG = TRAINER_RIVAL_IMG` (line 865).

`showEncounterBattle` uses `img` if URL valid, falls back to emoji.

---

## Custom Art

All listed below are non-empty Cloudinary URL constants:

- Items: `WORN_LOCKET_GOLD, WORN_LOCKET_SILVER, HEAVY_WALKING_STICK, HEAVY_WALKING_STICK_BROKEN, SHARPENING_STONE, SHARPENING_STONE_BROKEN` (lines 866-871).
- Fox: `FOX_DEFAULT, FOX_ELECTRIC, FOX_FIRE, FOX_WATER, FOX_STEEL, FOX_TOXIC` (lines 846-851).
- Hydra: `HYDRA_DEFAULT_IMG, HYDRA_STEEL, HYDRA_ELECTRIC, HYDRA_TOXIC, HYDRA_FIRE, HYDRA_WATER` (lines 854-859).
- NPC: `PROFESSOR_IMG` (line 853).
- Trainers: see Trainer section above.

`getItemIcon(item)` returns the right image for items with custom art, else falls back to the emoji icon.

---

## Layout

Single-page mobile portrait, no scroll, fixed viewport.

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

.player-section {
  height: 362px;          /* mobile */
  flex-shrink: 0;
  overflow: hidden;
  display: flex; flex-direction: column;
}
@media(min-width:600px) {
  .player-section { height: 402px; }   /* desktop */
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

Exactly four keys are used:

| Key | What it stores |
|-----|----------------|
| `zorpRivalName` | Persisted rival name across runs. Read at init, written by `confirmRivalName()`. |
| `zorpTutorialSeen` | `'true'` once tutorial completed or skipped. Read by `checkTutorialSeen()`, written by `markTutorialSeen()`. |
| `rivalWins` | Cumulative rival win count. |
| `rivalLosses` | Cumulative rival loss count. |

---

## Run / Game Flow Constants

- Honor: 13 (`G.honor`, `G.maxHonor`).
- Win target: 6 (game over check at `G.wins>=6`).
- Max days: ~11.
- Perfect run (6-0, `G.honor===13` at completion) → FLAWLESS screen with gold glow.
- Starting gold: 8.
- `DAY1_NODES` (5 nodes: mom, shop1, wild, shop2, rival) and `DAY2_NODES` (4 nodes: shop1, wild, shop2, rival). `getDayNodes()` returns `DAY2_NODES` after rival defeated and `G.day>1`.

---

## Post-Win Flow

After every trainer win (wins 1-5; not win 6):

```
postTrainerWin()
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

---

## Pending Work

- **Steel-typed Win 6 counter board** — currently a placeholder; replace when Toxic enemy boards are designed against.
- **Rival entry 6 dialogue/img polish** — `getDialogueBefore` / `getDialogueAfterLoss` exist as functions on the entry; check call sites stay correct as content evolves.

---

## What NOT to Change Without Discussion

- The `effectFn` three-argument signature.
- The `side` convention in damage functions (the `attackerMods` check).
- HEAD node pre-fill in init and `resetRun`.
- Post-win flow sequence — intentionally structured and fragile to reordering.
- `dealDmgWithStatus` applies status to the same target that was damaged (not a separate random target).
- Poison stack permanence (only ticks decrement, only `resetItemBattleState` zeroes).
