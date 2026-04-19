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

## Node Tree — 6 nodes

HEAD (prestige, pre-filled), NECK (beginner), SPINE1 (intermediate), SPINE2 (intermediate), FRONT_UPPER (advanced, branches from SPINE1), SPINE3 (advanced, end of spine).

Typed players fill 5 nodes — win spent on type pick gives no node.
Typeless players fill all 5 remaining nodes across 5 wins.

Deleted nodes — do not reference: `tail`, `back_upper`, `front_foot`, `back_foot`.

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

## Heal targeting

`healItem` targets lowest HP percentage ally (`hp/maxHp` ratio), filtered to items with `hp < maxHp`. Returns early if no healing needed (`actual <= 0`). `healAll` heals every alive item. ITEM_HEALED event is still fired.
