# ZORP — Technical Brief for Claude Code Sessions

Single self-contained HTML file game. All game logic, CSS, and HTML live in `Index.html.html`. No build step, no dependencies, no external files. Open in browser to play.

---

## Core Conventions

### `effectFn` signature
Every item's effect function takes three arguments: `(side, amount, self)`
- `side` — the **attacker's** side (`'player'` or `'enemy'`)
- `amount` — the item's current `effectAmt` (scales with upgrade tier)
- `self` — the item object itself (used for self-targeting effects like lightning_claw self-haste)

When calling damage functions from inside effectFn, pass `opp(side)` as the target side.

### `side` convention in damage functions
`side` always refers to the **target** side (where damage lands), not the attacker.
- `side === 'enemy'` → player is attacking the enemy → apply player skill mods
- `side === 'player'` → enemy is attacking the player → no player mods

The correct check for applying player attack bonuses is:
```js
const attackerMods = side === 'player' ? null : getSkillMods();
```
**Do NOT write** `side === 'enemy' ? null : getSkillMods()` — that is the inverted bug that existed in v4 and was fixed.

---

## Item Targeting Rules

| Effect type | Targets |
|---|---|
| `applyShield` | Random alive ally with `maxHp > 0` |
| `healItem` | Ally with lowest HP percentage |
| `healAll` | All alive allies with `maxHp > 0` |
| `dealDmg` | Leftmost front row enemy (or back if front empty) |
| `dealDmgRandom` | Random front row enemy (or back if front empty) |
| `dealDmgTwoRandom` | 2 random enemies, can hit same target twice |
| `dealDmgBackRow` | Random back row enemy (falls back to front if back empty) |
| `dealDmgWithStatus` | Random front enemy + applies status to that same target |
| `applyShieldAll` | All alive allies with `maxHp > 0` |

Food items have `maxHp === 0` and are therefore excluded from all targeting pools automatically. Their tag should read `'Untargetable'` not `'Food'`.

---

## Status Effects

All four statuses live on the item object and tick in `tickSide`.

| Status | Field | Tick interval | Damage rule |
|---|---|---|---|
| Burn | `burnStack`, `burnTickMs` | 500ms | `shield × 2` absorbs, remainder hits HP. Stack decrements by 1 each tick. |
| Poison | `poisonStack`, `poisonTickMs` | 1000ms | Bypasses shield entirely. Stack decrements by 1 each tick. |
| Haste | `hasteMs` | Countdown timer | Speed multiplier 2.0 (faster activation) |
| Slow | `slowMs` | Countdown timer | Speed multiplier 0.5 (slower activation) |

Haste and slow cancel each other when both active (net 1.0x). `applyHaste` and `applySlow` use `Math.max` — they refresh duration rather than stack additively.

Status fields must be reset to zero in `resetItemBattleState`. New items created via `buyItem` must include all status fields initialized to zero.

---

## Post-Win Flow

After every trainer win (wins 1–5 only, not win 6):

```
postTrainerWin()
  ├─ G.type === null → showTypeSelect()
  │     ├─ Pick a type → remove Potential from activeSkills, overwrite treeState['head'],
  │     │                 add type awakening to activeSkills → advanceDay() [no node pick]
  │     └─ Stay typeless → showBanScreen() if 2+ types non-banned, else showSkillPick()
  │           └─ banType() → showSkillPick()
  │                 └─ selectNode() → pickSkill() → advanceDay()
  └─ G.type !== null → showSkillPick() → selectNode() → pickSkill() → advanceDay()
```

Win 6 → game over. `continueAfterResult` calls `resetRun()` when `G.wins >= 6`.
Perfect run (6-0, G.honor===13 at completion) → FLAWLESS screen with gold glow.

`advanceDay()` only increments the day and fires `activateCurrentNode()`. It does NOT trigger type select — that happens entirely in the post-win chain before `advanceDay` is called.

---

## Node Tree — 6 nodes (Fox)

HEAD (prestige, pre-filled), FRONT_SHOULDER (beginner), BACK_SHOULDER (intermediate), FRONT_FOOT (intermediate, branches from FRONT_SHOULDER), BACK_FOOT (advanced, branches from BACK_SHOULDER), TAIL_TIP (mastered, branches from BACK_SHOULDER).

Parent relationships:
- head → front_shoulder
- front_shoulder → back_shoulder
- front_shoulder → front_foot
- back_shoulder → back_foot
- back_shoulder → tail_tip

Typed players skip FRONT_SHOULDER — fill 4 nodes max.
Typeless fill all 5 remaining nodes.

Deleted nodes — do not reference: `neck`, `spine1`, `spine2`, `spine3`, `front_upper`, `tail` (old).

Rarity by node: prestige (head), beginner (front_shoulder), intermediate (back_shoulder, front_foot), advanced (back_foot), mastered (tail_tip).

- HEAD node is pre-filled with Potential at init and on every reset. Never leave HEAD empty.
- `getAvailableNodes()` returns all nodes where parent is filled and self is empty (excluding HEAD).

---

## Key State Objects

```js
G          // global game state: gold, honor(13), wins, day, type, rivalDefeated, dayStep
board      // {front: [3 items], back: [3 items]} — player board
enemyBoard // same shape — enemy board
treeState  // {nodeId: skill object or null} — skill tree state
activeSkills // array of skill objects currently active on player
bannedTypes  // array of type id strings banned from future offerings
```

## Honor and run length

Honor: 13. Win target: 6. Max days: 11.
Perfect run (6-0): FLAWLESS screen — triggered when `G.honor===13` at `G.wins>=6`.

## Board size

Max 6 slots — 3 front, 3 back. Arrays are length 3.
Expansion: day 1 = 1 per row, day 2 = 2 per row, day 3+ = 3 per row.

---

## Skill System

SKILLS object contains: prestige (Potential only), beginner (empty), intermediate (empty), advanced (empty), mastered (empty).

All old skills removed except Potential. New skills added in Prompt 2 (Beginner/Intermediate) and Prompt 3 (Advanced/Mastered).

TYPED_SKILLS removed entirely. Typed skills to be designed and added later.

getSkillMods currently only handles Potential. New skill mods added in Prompt 2 and 3.

Deleted skill IDs — do not reference:
`thick_fur`, `quick_reflex`, `scrappy`, `resilience` (skill), `battle_hardened`, `dense_coat`, `pack_rhythm`, `momentum` (old), `apex_strike`, `endurance`, `last_stand`, `iron_hide`, `alpha`, `unbreakable`, `static_field`, `chain_lightning`, `ember_coat`, `wildfire`, `frost_aura`, `deep_freeze`, `plating_mastery`, `fortress`.

---

## TYPES Array

Five types (no typeless entry in the array — typeless is handled separately in showTypeSelect):
`electric`, `fire`, `water`, `steel`, `toxic`

Ice does not exist. If you see `id: 'ice'` anywhere, that is a regression — replace with water.

**Typed items are in the general pool.** All items in `TYPED_ITEMS` appear in shops for all players regardless of `G.type`. The `TYPED_ITEM_POOLS` constant and any filtering by `G.type` in `getItemsOfRarity` have been removed. Type selection only affects which typed skills are offered at bond tree node picks — not which items appear in shops.

---

## What NOT to Change Without Discussion

- The `effectFn` three-argument signature
- The `side` convention in damage functions (the attackerMods check)
- `applyShield` targeting random (not most wounded — that's `healItem`'s job)
- HEAD node pre-fill in init and resetRun
- Post-win flow sequence — it is intentionally structured and fragile to reordering
- `dealDmgWithStatus` applying to the same target that was damaged (not a separate random target)

---

## Event System

The battle engine uses a global event bus. `fireEvent(event)` dispatches to all non-broken items and all active skill handlers.

**Event payload:** `{type, item, side, value, source}`

**14 event types:** BATTLE_START, FATIGUE_START, BATTLE_END, ITEM_ACTIVATED, ITEM_BROKEN, ITEM_HEALED, ITEM_SHIELDED, DAMAGE_DEALT, DAMAGE_RECEIVED, BURN_APPLIED, POISON_APPLIED, HASTE_APPLIED, SLOW_APPLIED, SLOW_APPLIED

**ITEM_BROKEN fires BEFORE item.broken is set** — handlers can still act on the item while it exists.

**Re-entrant guard:** `firingEvent` flag prevents cascade beyond one level. Action functions never call `fireEvent` directly — only the engine does.

**Items use `handleEvent(event)` for triggered effects.** Passive multipliers stay in `getSkillMods()`.

**Skill handlers** use `sk._battleHandler` registered in `registerSkillHandlers()` called from `runBattle`.

**`getNeighbors(item)`** returns `{side, row, index, left, right, behind, inFront}`. Use for spatial triggers.

**`item._side`** is set in `resetItemBattleState` — use this to know which board an item belongs to inside handlers.

**Do not call `fireEvent` from inside action functions** — applyDmgTo, applyShield, healItem etc already fire their own events. Calling fireEvent again inside a handler creates re-entrant loops caught by the guard and silently dropped.

---

## New Item Mechanics

**hitCount items:** Training Dummy uses `hitCount` instead of HP. `targetable()` checks `isTargetable===true` as override. `applyDmgTo` checks for `hitCount` before shield logic and absorbs damage entirely. Burn and poison each decrement hitCount per tick.

**Technique items:** `isTechnique=true`. Purchased as tomes, consumed on buy, placed on board as passive items. Show tome modal popup on purchase. Sell via `showTechSellModal` with unique popup text. Use `handleEvent` for all effects.

**Chrysalis:** 1 HP, 1s activation, gains plating each tick. Breaks instantly from any damage.

**NEGATIVE_STATUSES:** Set of event types considered debuffs — `BURN_APPLIED`, `POISON_APPLIED`, `SLOW_APPLIED`. Add new debuff event types here when created.

**battleState.fatigueDamageMultiplier:** Default 1. Set by Patience on `FATIGUE_START`. Applied to enemy fatigue damage in `startFatigue`.

**G.monsterName:** Defaults to `'Kip'`. Used in technique popup text via `getTechniqueReading()` and `getTechniqueReaction()`.

**PACIFIST_ITEMS pool:** Contains Solidarity, Emergency Ration, Honey Pot, Thorn Mantle, Training Dummy, Chrysalis, Patience, Callus. Included in shopPool and ITEM_META.

---

## HEADBUTT_ITEMS pool

Contains: headbutt, ricochet, arsonist, spite.

**Headbutt:** Deals damage to leftmost enemy front row then takes half that damage as self-damage. Self-damage fires full DAMAGE_RECEIVED events so items like Emergency Ration and Solidarity can respond.

**Ricochet:** No HP, no activation timer. handleEvent on DAMAGE_RECEIVED — fires when item directly BEHIND it (neighbors.behind) takes damage. Deals 3 to random enemy.

**Arsonist:** No activation timer, handleEvent on BATTLE_START. Applies 5 burn to every non-broken item on both boards. Has 160hp so it can be targeted and killed.

**Spite:** 50hp, 1s activation. Uses `_spiteCounter` instead of effectAmt for current damage value. handleEvent on DAMAGE_RECEIVED where side matches and item is not itself — increments `_spiteCounter` by 1. Counter resets to 1 in resetItemBattleState.

**Relic** is a new item type tag for items like Arsonist that have passive world-altering effects.

---

## Second Wind — Part 2

`pickSecondWindBuff(id)`: routes to `showSecondWindItemSelect` if `requiresItemSelect`, else calls `applySecondWindBuff` directly.

`showSecondWindItemSelect(buff)`: shows all non-broken board items as selectable cards. Calls `selectSecondWindItem` on tap.

`selectSecondWindItem(buffId,row,idx)`: stores pending selection in `pendingSecondWindBuff/Row/Idx`, shows confirm/back screen.

`confirmSecondWindItem`: calls `applySecondWindBuff` with stored pending item, then clears pending state.

`applySecondWindBuff(buff,targetItem)`: switch on `buff.id`. All non-Clarity buffs call `advanceDay()` at end. Clarity sets `secondWindClarityPending=true` then calls `showSkillPick()`.

`secondWindClarityPending`: module-level flag checked at top of `advanceDay`. If true, clears flag, increments `dayStep`, and calls `activateCurrentNode()` (advances within current day without incrementing day counter).

`renderSecondWindNode()`: inserts/updates red badge in player header next to type badge. Called from `updateUI` so it persists across all screen changes. Clicking badge shows notif with buff name and description.

`G.steadfastActive`: checked in `startFatigue`. If true, player fatigue damage is halved via `actualPlayerDmg`.

`G.secondWindBuff`: stores the chosen buff object for the run. Used by `renderSecondWindNode` and persists until `resetRun`.

---

## BURN_ITEMS pool

Contains: stoke, blaze, cinder_block, cauterize, fireworks.

**battleState.burnApplicationsToEnemy:** Incremented in `applyBurn` by `stacks` when target is enemy. Reset in `initBattleState`.

**battleState.fireworksThreshold:** Default 6. Checked in `checkFireworks()`.

**`checkFireworks()`:** Called from `applyBurn`. Triggers fireworks item burst once threshold reached, resets counter. Repeats.

**`flashFireworks()`:** Orange screen overlay animation.

**Stoke:** Relic, 75hp, 6s timer. Applies effectAmt burn to random enemy.

**Blaze:** Weapon, 110hp, 4s timer. Deals effectAmt dmg + `_blazeCounter` burn stacks, counter increments each activation. Resets in `resetItemBattleState`.

**Cinder Block:** Armor, 200hp, no timer. handleEvent on BURN_APPLIED to enemy — gains +effectAmt shield on self.

**Cauterize:** Relic, 0hp, no timer, untargetable. handleEvent on ITEM_BROKEN (player side) — heals all surviving allies via `healAll`.

**Fireworks:** Relic, 0hp, no timer, untargetable. Pure passive — `checkFireworks()` fires when counter reaches threshold. Deals effectAmt to all enemies. Repeats.

---

## WATER_ITEMS pool

Contains: glacier, hydro_cannon, whirlpool, deluge, flood, hot_springs.

**Status effect keywords:** Slow is called **Wet** in all item descriptions. Haste is called **Jolt** in all item descriptions. Underlying CSS classes and mechanic names (`slowMs`, `hasteMs`, `applySlow`, `applyHaste`) are unchanged — keyword rename is display only.

**battleState.slowProcsThisBattle:** Incremented by 1 in `applySlow` after each application. Reset in `initBattleState`.

**battleState.floodTriggered:** Boolean, set true when Flood threshold reached. Reset in `initBattleState`.

**battleState.wetDurationMult:** Default 1. Set to 2.0 when Flood triggers. Applied in `applySlow` to multiply each duration applied. `applySlow` now uses additive `+` rather than `Math.max`.

**Glacier:** Beginner anchor. Applies 2s Wet on 4s timer. effectAmt is the raw durationMs (2000).

**Hydro Cannon:** Weapon. 15 dmg base, 30 if target has `slowMs > 0`. Double is always 2× effectAmt.

**Whirlpool:** Armor. 20 plating base, 40 if any enemy has `slowMs > 0`. Checks enemyBoard.

**Deluge:** Weapon. Damage = slowProcsThisBattle × effectAmt, minimum effectAmt. Single target.

**Flood:** Relic, 0hp, untargetable, no timer. `checkFlood()` called from `applySlow`. Triggers once per battle at threshold (effectAmt=10). Sets `wetDurationMult=2.0`.

**Hot Springs:** Provision, 0hp, 7s timer, 1 use. Heals lowest HP ally via `healItem`. `applySlow` grants +1 use to all non-broken hot_springs on player board after each slow application.

## ELECTRIC_ITEMS pool

Contains: live_wire, dual_shock, charging_station, feedback, thunder.

**volt_band removed** — deleted from TYPED_ITEMS entirely.

**Live Wire:** Beginner weapon, 80hp, 3s. Checks `self.hasteMs>0` at activation. If Jolted, applies 2 burn to target in addition to effectAmt damage.

**Dual Shock:** Beginner weapon, 80hp, 3s. Hits rightmost enemy normally. If `self.hasteMs>0` at activation also hits leftmost enemy for same effectAmt.

**Charging Station:** Intermediate armor, 150hp, 4s. Uses `getNeighbors(self)` to find left and right neighbors. Calls `applyHaste` on each non-broken neighbor for 2000ms.

**Feedback:** Intermediate relic, 0hp, untargetable, no timer. `handleEvent` on `HASTE_APPLIED` where `event.side==='player'`. Picks random alive player item and calls `applyHaste` for `effectAmt` ms (1000ms base). Re-broadcast of Feedback's triggered haste is silently dropped by re-entrant guard — no infinite loop.

**Thunder:** Advanced weapon, 150hp, no activation timer. `checkThunder()` called from `tickSide` when joltedCount>0. Fires once per battle (`thunderFired` flag). Sorts enemy items by hp ascending, fires ITEM_BROKEN then sets broken on lowest two. ITEM_BROKEN fires before broken is set per convention.

**battleState.joltCumulativeMs:** Incremented each tick by `tickMs × number of currently Jolted player items`. Reset in `initBattleState`.

**battleState.thunderFired:** Boolean, reset in `initBattleState`. Prevents Thunder firing more than once.

**HASTE_APPLIED event:** Already fired in `applyHaste` — confirmed present before Feedback was added.

---

## STEEL_ITEMS pool

Contains: ironclad, aegis, temper, wrecking_ball, thornback_armor.

**battleState.totalSidePlating:** Incremented in `applyShield` whenever plating is applied to player side. Reset in `initBattleState`. Used by Wrecking Ball death trigger.

**Ironclad:** Beginner armor, 200hp, 5s. Uses `getNeighbors(self)` to find left and right. Calls `applyShield` with specific `targetItem` for each neighbor.

**Aegis:** Intermediate armor, 220hp, 4s. Calls `applyShield` targeting self first, then reads `self.shield` for damage value. Hits random enemy for current shield total.

**Temper:** Intermediate untargetable relic, no HP. `handleEvent` on `ITEM_ACTIVATED` where `event.side==='player'` and `event.item!==self`. Plates all alive player items with `maxHp>0` for effectAmt each.

**Wrecking Ball:** Advanced armor, 80hp, 6s. Plates self via effectFn. `onBreak` callback fires after `item.broken=true` in all break paths. Deals `Math.round(totalSidePlating/2)` to frontmost enemy and item at same index in back row.

**Thornback Armor:** Intermediate armor, 240hp, no activation timer. `handleEvent` on `DAMAGE_RECEIVED` where `event.item===self`. Calculates `absorbed=(event.rawValue||0)-(event.value||0)`. Deals absorbed back to `event.source`.

**applyShield updated:** Now accepts optional 4th parameter `targetItem`. If provided and not broken, plates that specific item. Default (no targetItem) falls back to lowest HP% ally. All existing calls without targetItem continue unchanged. `ITEM_SHIELDED` event removed from applyShield (cinder_block still fires its own). `totalSidePlating` incremented for player side.

**`applyShield` targeting note:** Default target is now lowest HP% ally (not random). The "What NOT to Change Without Discussion" note in earlier CLAUDE.md referred to accidental changes — this is an intentional update.

**onBreak pattern:** `item.onBreak(item)` called after `item.broken=true` in all break paths: `checkAndBreak`, `applyDmgTo` hitCount path, tickSide uses-depletion, tickSide burn hitCount, tickSide poison hitCount. Burn/poison HP paths go through `checkAndBreak` so are covered. Currently used by Wrecking Ball.

---

## Heal targeting

`healItem` targets lowest HP percentage ally (`hp/maxHp` ratio), filtered to items with `hp < maxHp`. Returns early if no healing needed (`actual <= 0`). `healAll` heals every alive item. ITEM_HEALED event is still fired.

---

## Win 6 — The Rival Returns

TRAINER_DATA[6].buildTeam reads G.type and builds counter board:
- electric → Steel counter (Thornback Armor, Aegis, Ironclad, Temper, Wrecking Ball, Hard Shell)
- fire → Water counter (Glacier, Hydro Cannon, Whirlpool, Deluge, Hot Springs, Tidal Shell)
- water → Electric counter (Live Wire, Dual Shock, Charging Station, Feedback, Thunder, Arc Bolt)
- toxic → Fire counter (Cinder Claw, Ember Fang, Arsonist, Stoke, Fireworks, Blaze)
- steel → Placeholder generic (replace when Toxic items implemented)
- null/typeless → Strong generic board

Upgrade levels: front row items Advanced (upgradeLevel 2), back row items Intermediate-Advanced (upgradeLevel 1-2).

Type-specific tags show in enemy info panel hinting at what the rival prepared.

rivalRecord tracked in localStorage. getDialogueBefore and getDialogueAfterLoss are functions not strings on TRAINER_DATA[6] — showEncounterBattle and endBattle must call them as functions.

---

## Poison mechanic update

Poison stacks are now permanent — they never decrement through ticking. `poisonStack` only resets in `resetItemBattleState` at battle start. Poison still stops ticking when stack reaches 0 naturally but normal gameplay never reduces stacks. This is a global change affecting all poison items.

## Removed items

`toxic_cloud`, `rot_berry`, `corrode` removed from TYPED_ITEMS entirely.

## POISON_ITEMS pool

Contains: caustic_mantle, mithridate, miasma, outbreak.
`venom_fang` reworked and remains in TYPED_ITEMS as the Beginner anchor.

**Venom Fang:** Checks board position each activation. If `self===board.front[0]` or `enemyBoard.front[0]`, uses 3s timer. Otherwise 4s. Applies 1 poison to random front enemy.

**Caustic Mantle:** 300hp, no shield. `handleEvent` DAMAGE_RECEIVED where `event.item===self`. Applies 1 poison to `event.source`. Requires source to be wired in applyDmgTo to activate.

**Mithridate:** 150hp, 5s timer. Tracks `_mithridateCount`. Odd: heals front[0], front[2], back[1] for effectAmt. Even: poisons 2 random enemy back items for 1.

**Miasma:** 100hp, no timer. `handleEvent` ITEM_ACTIVATED where `event.item===neighbors.inFront`. Tracks `_miasmaTriggerCount`. Odd triggers: poison enemy front[0], front[2], back[1]. Even triggers: poison enemy front[1], back[0], back[2].

**Outbreak:** Untargetable, no HP. Uses `battleState.poisonApplicationsToEnemy` counter and `outbreakTriggered` flag. After 10 applications, extra +1 poison applied in `applyPoison` automatically.

**battleState.poisonApplicationsToEnemy:** Incremented by 1 in `applyPoison` when `item._side==='enemy'`.
**battleState.outbreakTriggered:** Set true when `poisonApplicationsToEnemy>=10` in `checkOutbreak`.
