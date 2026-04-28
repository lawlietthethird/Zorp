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

SKILLS: prestige (Potential), beginner (6 skills), intermediate (8 skills), advanced (14 skills), mastered (2 skills).
TYPED_SKILLS removed. Typed skills to be designed later.
selectNode shows 2 skill choices (slice 0,2).

## Beginner Skills (6)
`thick_fur`: +20 maxHp via mods.maxHpBonus in getSkillMods (feeds into item.maxHp formula).
`quick_reflexes`: speedMult*=0.90 in getSkillMods.
`scrappy`: dmgBonus+=6 in getSkillMods.
`resilience`: first DAMAGE_RECEIVED adds +70 plating, resilienceUsed flag in initBattleState.
`refrigeration`: +1 use to all provisions in resetItemBattleState forEach.
`fortified`: +7 extra plating (globalMult-scaled) in applyShield when side===player.

## Intermediate Skills (8)
`momentum`: momentumBonus+=4 per player activation in tickSide, consumed in dealDmg/dealDmgRandom/etc.
`duality`: odd slots (front[0],front[2],back[1]) +10 effectAmt+baseEffect, even slots (front[1],back[0],back[2]) +25 maxHp+baseMaxHp. Second pass in resetItemBattleState after forEach. dualityApplied flag prevents re-application.
`kickstart`: slot 6 activation Jolts random ally 2s, max 3× per battle. kickstartCount tracks.
`friction`: slot 6 activation Wets random enemy 2s, max 3× per battle. frictionCount tracks.
`benediction`: heals back row items 15 HP every 5000ms. benedictionMs accumulates in runBattle interval.
`assembled`: assembledActive flag set in resetItemBattleState if all 6 slots filled. speedMult*=0.82 in getSkillMods when assembledActive.
`crescendo`: crescendoCount increments each player activation. At 5: count resets, crescendoReady=true. Next effectFn call multiplies effectAmt×2. crescendoReady cleared after use.
`epicenter`: slot2SpeedMult*=0.75 and slot5SpeedMult*=0.75 in getSkillMods. Applied per-slot in tickSide via slotNum.

## Advanced Skills (14)
`endurance`: fatigueMult*=0.6 in getSkillMods (40% fatigue reduction).
`last_stand`: each player item gets `_lastStandApplied=false` in resetItemBattleState. In checkAndBreak: if player, !_lastStandApplied → set hp=1, _lastStandApplied=true, return (no break).
`iron_hide`: ironHideActive=true in initBattleState when skill active. In checkAndBreak: if ironHideActive → set hp=1, return. Expires in runBattle interval at battleMs>=10000 (sets ironHideActive=false, ironHideExpired=true).
`phoenix_downed`: phoenixDownAvailable already in initBattleState. In checkAndBreak: if phoenixDownAvailable && skill active → set hp=maxHp*0.5, phoenixDownAvailable=false, return (no break).
`echo`: in checkAndBreak after normal break: if side===player && n.row==='front' && n.behind exists && not broken → trigger n.behind.effectFn immediately.
`waterproof`: early return in applySlow when item._side==='player'.
`grounded`: early return in applyHaste when item._side==='enemy'.
`immunity`: poisonInterval=2000ms (vs 1000ms) in tickSide poison loop when side==='player'.
`asbestos`: after normal burnStack-- in tickSide HP path: if player → burnStack=Math.max(0,burnStack-2).
`shatter`: in applyShield, if side==='enemy' → actual=Math.floor(actual/2).
`war_chest`: warChestBonus=Math.floor(G.gold/2) set in initBattleState. dmgBonus+=warChestBonus in getSkillMods.
`desperation`: in checkAndBreak after break: if player → multiply effectAmt and baseEffect ×1.5 for all non-broken player items. Stacks with each additional break.
`predator`: predatorActive flag in initBattleState. Checked in runBattle interval: if 3+ enemy items broken → predatorActive=true. globalMult*=1.5 in getSkillMods when predatorActive.
`flow_state`: flowStateActive flag in initBattleState. Set in runBattle interval when battleMs>=20000. globalMult*=1.15 in getSkillMods when flowStateActive.

## Mastered Skills (2)
`apotheosis`: second pass in resetItemBattleState after forEach: all player items with baseEffect get effectAmt=Math.round(baseEffect*UPGRADE_MULTS[3]).
`fizzle`: after resetItemBattleState calls in runBattle: halve effectAmt and baseEffect of all enemy items with maxHp===0 (untargetable passives).

## battleState fields (all skills combined)
`resilienceUsed:false`, `crescendoCount:0`, `crescendoReady:false`, `benedictionMs:0`, `kickstartCount:0`, `frictionCount:0`, `assembledActive:false`, `dualityApplied:false`
`ironHideActive:(skill active)`, `ironHideExpired:false`, `flowStateActive:false`, `predatorActive:false`, `warChestBonus:(Math.floor(G.gold/2) if active else 0)`

Deleted skill IDs — do not reference:
`quick_reflex` (now `quick_reflexes`), `battle_hardened`, `dense_coat`, `pack_rhythm`, `apex_strike`, `alpha`, `unbreakable`, `static_field`, `chain_lightning`, `ember_coat`, `wildfire`, `frost_aura`, `deep_freeze`, `plating_mastery`, `fortress`.

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
- fire → Water counter (Glacier, Hydro Cannon, Whirlpool, Deluge, Hot Springs, Coral)
- water → Electric counter (Live Wire, Dual Shock, Charging Station, Feedback, Thunder, Arc Bolt)
- toxic → Fire counter (Scorch, Inferno, Arsonist, Stoke, Fireworks, Blaze)
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

Removed from ITEMS: ration_bag, basic_berry, iron_paw, hard_shell, berry_pouch.
Removed from TYPED_ITEMS: charge_pack (electric), scorched_shell (fire), flame_wick (fire), iron_carapace (steel), forge_core (steel).

Trainer and wild boards updated to use only existing valid item keys.
TYPED_ITEM_POOLS updated to reflect removals.

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

---

## Monster System

G.monsterType: 'fox' or 'hydra'. Set during professor flow.
G.monsterName: player-named. Defaults 'Kip' (fox) or 'Hydrax' (hydra). Persists per run.
G.rivalName: player-named. Persists in localStorage as 'zorpRivalName'. Default 'Rival'.

FOX_TREE_NODES: head, front_shoulder, back_shoulder, front_foot, back_foot, tail_tip.
HYDRA_TREE_NODES: body, neck_split, mid_left, mid_right, head_left, head_right.
getTreeNodes(): returns correct array based on G.monsterType.
rebuildSkillTreeSVG(): rebuilds SVG, resets treeState, calls renderSkillTree and updateFoxImage.
getSkillTreeSVGContent(): returns SVG nodes for current monster type.
drawTreeLines(): uses foxPos or hydraPos based on G.monsterType.

Professor flow order:
0: showProfessorIntro
1: showProfessorShop (browse or decline)
2: showMonsterSelection
3: showMonsterNaming
4: showRivalNaming (tutorialSeen check present but currently routes to showRivalNaming either way)
5: showRivalNaming (after tutorial, currently unused)
6: beginRun

localStorage keys:
zorpTutorialSeen: 'true' when tutorial completed once
zorpRivalName: persisted rival name across resets

Rival tag: always 'Not Gary' — the professor named them.

## Tutorial System

showTutorial(): entry point, sets tutorialStep=0, calls showTutorialStep().
showTutorialStep(): switch on tutorialStep 0-4. Cases 0-3 show screens. Case 4 marks tutorial seen and advances professor flow to step 5.
skipTutorial(): marks tutorial seen, advances professor flow to step 5 immediately.
tutorialStep: local counter, not in G object.

Tutorial screens:
0: showTutorialCards — card anatomy explanation + demo battle
1: showTutorialFight — auto-battle explanation + demo battle
2: showTutorialKeywords — burn/poison/jolt/wet
3: showTutorialJourney — honor/wins/bond tree

Demo battle: startDemoBattle() and stopDemoBattle() are stubs filled in by Prompt 3.
demoBattleContainer: div inside tutorial screens 0 and 1 where demo renders.
demoCommentary: div below demo showing professor commentary updates.

zorpTutorialSeen localStorage flag set by markTutorialSeen() when tutorial completes or is skipped.

## Demo Battle System

Completely isolated from main game state. Uses demoItems object with shield and spark properties.
demoInterval: setInterval handle. Always cleared by stopDemoBattle before starting new demo.
demoMs: elapsed battle time in ms.
demoFatigueMs: elapsed fatigue time.
demoFatigueActive: boolean, true when demoMs>=30000.

makeDemoItem(id,name,icon,maxHp,actMs,type): creates demo item. type='shield' sets effectAmt=30, type='attack' sets effectAmt=13.
initDemoItems(): creates fresh Bubble Shield (200hp, 6s) and Spark Charm (80hp, 3s). Resets all demo state.
tickDemo(): runs every 100ms. Ticks both items. Spark Charm damages shield item. Bubble Shield plates self. Fatigue at 30s. Auto-resets after 2s delay when demo ends.
renderDemoCards(): rebuilds demoBattleContainer innerHTML. Shows both cards with live bars.
setDemoCommentary(text): updates demoCommentary div with professor commentary.
startDemoBattle(): stops existing interval, inits items, starts new interval.
stopDemoBattle(): clears interval, resets demo state. Called on NEXT, SKIP, and tutorial completion.

Demo outcome: Spark Charm (80hp) dies to fatigue faster than Bubble Shield (200hp). Bubble Shield wins. Loop resets after 2s.

## Professor Screen Layout

profLayout(dialogueHTML, extraHTML): wraps content with professor image (90px) on left, content on right. Used on all professor screens.

PROFESSOR_IMG: Cloudinary URL for professor character image.
HYDRA_DEFAULT_IMG: Cloudinary URL for default hydra image.

Monster selection: compact cards with 60x60px images, shortened dialogue, SELECT button always visible.

Browse path: three beat joke — excited, player doesn't buy, disappointment.
Decline path: necklace reference.

## Demo Battle Fix
demoResetTimeout: separate timeout handle for post-battle REPLAY button. Cancelled by stopDemoBattle.
Demo no longer auto-loops — stops on completion, shows REPLAY button after 1.5s, NEXT and SKIP always accessible.

## Hydra Typed Images
HYDRA_STEEL, HYDRA_ELECTRIC, HYDRA_TOXIC, HYDRA_FIRE, HYDRA_WATER — all Cloudinary URLs.
getFoxImage() handles both fox and hydra typed variants.

## Trainer Images
TRAINER_RIVAL_IMG, TRAINER_INVESTOR_IMG — populated.
TRAINER_ARCHITECT_IMG, TRAINER_MONK_IMG, TRAINER_MIRROR_IMG, TRAINER_RIVAL_RETURNS_IMG — empty strings pending art.
Each TRAINER_DATA entry has img field. showEncounterBattle uses img if URL valid, falls back to emoji.

## Rival Name
G.rivalName drives all rival references throughout the game.
TRAINER_DATA[1] and TRAINER_DATA[6] name fields are getters returning G.rivalName.toUpperCase().
No hardcoded 'THE RIVAL' or 'RIVAL' strings remain in dialogue or battle titles.
Rival name persists in localStorage as zorpRivalName.

## Item Renames
left_claws → Hook
spark_charm → Surge
cinder_claw → Scorch
ember_fang → Inferno
current_band → Water Gun
tide_fang → Hydro Blast
tidal_shell → Coral
bulwark_stance → Reinforce
alloy_fang → Steel Jaw

Keys unchanged. Only display names updated.

## Typed Skills System

TYPED_SKILLS object structure: TYPED_SKILLS[type][rarity] = array of skill objects.
Types: electric, fire, water, steel, toxic.
Each type has: beginner (3), intermediate (3), advanced (3-4), mastered (2).

Skill offer logic — getSkillOffers(nodeRarity):
- Typed player: slot 1 always typed, slots 2 and 3 each 25% chance typed, 3 total offered
- Typeless player: 3 generic skills offered
- Falls back to generic if typed pool exhausted

Typed skill objects have extra fields: type (string), shown as [TYPE] label in offer card.
activeSkills array contains both generic and typed skills.
pickSkill handles both generic and typed skill IDs.

Electric and Fire typed skill effects are IMPLEMENTED (Prompt 2). Water and Steel typed skill effects are IMPLEMENTED (Prompt 3). Toxic typed skill effects are NOT YET IMPLEMENTED.

---

## Electric Typed Skills — Battle Effects

**New battleState fields:** `electricActivationCount:0`, `capacitorFired:false`, `thunderclapFired:false`, `dischargeCount:0`, `dischargeFired:false`, `overchargeCount:0`, `conductorBonuses:{}`.

**New helper functions** (added after HELPERS section):
- `isJolted(item)` — true if item.hasteMs > 0
- `countJoltedAllies()` — count non-broken jolted player items
- `countJoltedEnemies()` — count non-broken jolted enemy items
- `countBurnStacksOnBoard(b)` — sum burnStack across all non-broken items on board b
- `getItemColumn(item,b)` — returns `{row,col}` for item on board b
- `getItemBehind(item,b)` — returns non-broken back item in same column, or null

**applyBurn updated:** Signature is now `applyBurn(item, stacks, src, attackerSide='player')`. If `attackerSide==='player'` and `kindle` skill is active, `stacks+=1` before any other logic.

**applyHaste updated:** Signature is now `applyHaste(item, durationMs, src, sourceIsEnemy=false)`.
- Grounding Wire: if `sourceIsEnemy` and item is in `board.back`, block Jolt and deal 2 dmg to random enemy instead.
- Grounded check unchanged.
- Resonance: if `resonance` skill active and `item._side==='player'`, multiply `durationMs` by 1.25 before setting hasteMs.
- Node: after setting hasteMs, if `node` skill active and `!sourceIsEnemy` and player item, mirror Jolt to slot 1 ↔ slot 4 partner.
- Thunderclap: after setting hasteMs, if `thunderclap` skill active and `!sourceIsEnemy` and player item and not yet fired, check if `countJoltedAllies()>=3` → set `thunderclapFired=true`, deal 25 to all non-broken enemies with maxHp>0.

**dealDmg / dealDmgRandom / dealDmgTwoRandom / dealDmgWithStatus:** After target selection, before `applyDmgTo`, apply:
- `charged`: +2 dmg per Jolted enemy (when side==='enemy')
- `arc`: +8 dmg if target is Jolted (when side==='enemy')

**tickSide (activation block):** `battleState.activatingItem=item` is set before `item.effectFn(...)` and cleared to `null` immediately after. After effectFn and ITEM_ACTIVATED event, for `side==='player'`:
- `capacitor`: increment `electricActivationCount`; at 8 activations (once) Jolt all player items 2s.
- `feedback_loop`: if activating item is Jolted, deal 3 dmg to random enemy.
- `conductor`: if front-row item activates, the item directly behind (same column) permanently gets `baseActMs -= 300` (min 500ms); tracked per item via `conductorBonuses[behind.id]`.
- `overcharge`: increment `overchargeCount` for each Jolted activation; every 5th fires `effectFn` again.
- `infinite_loop`: 15% chance to Jolt activating item for 1s.

**checkAndBreak (after break effects):**
- `discharge`: increment `dischargeCount` on each player break; at count>=2 and `!dischargeFired`, Jolt all remaining player items 2s.

---

## Fire Typed Skills — Battle Effects

**New battleState fields:** `emberStormFired:false`, `hearthMs:0`, `activatingItem:null`.

**applyBurn Kindle:** See above — `attackerSide='player'` parameter adds +1 stack when kindle active.

**dealDmg / dealDmgRandom / dealDmgTwoRandom / dealDmgWithStatus:** After target selection, before `applyDmgTo`:
- `stoked`: +1 dmg per burn stack currently on the entire enemy board (when side==='enemy')
- `searing`: +8 dmg if target has burnStack>0 AND activatingItem.itemType==='weapon' (when side==='enemy')
- `meltdown`: ×2 dmg if target has 5+ burn stacks (when side==='enemy')

**tickSide burn tick (after burnStack--, Asbestos):**
- `backfire`: each enemy burn tick → `dealDmgRandom('enemy',1,'Backfire')`.
- `wildfire`: 20% chance per enemy burn tick → spread 1 burn to a random other non-broken enemy via `applyBurn(wt,1,'Wildfire','player')`.

**checkAndBreak (after break effects):**
- `flash_point`: enemy breaks with >=3 burnStack → `dealDmgRandom('enemy',20,'Flash Point')`.
- `conflagration`: enemy breaks while burning → `Math.ceil(burnStack/2)` stacks spread to all other non-broken enemies via `applyBurn(...,'player')`.
- `backdraft`: player breaks → deal `countBurnStacksOnBoard(enemyBoard)` dmg to random enemy.

**runBattle interval:**
- `ember_storm`: at battleMs>=20000 (once per battle), apply 3 burn to all non-broken enemies via `applyBurn(...,'player')`.
- `hearth`: accumulate `hearthMs`; every 8s, if any back row player items and any enemies alive, apply 1 burn to random enemy via `applyBurn(...,'player')`.

**Log function:** The battle log function is `log(type, msg)` where type is `'a'` for action, `'d'` for damage, `'h'` for heal, `'s'` for status, `'b'` for break, `'x'` for system. Use `log('a', ...)` for most typed skill notifications.

---

## Water Typed Skills — Battle Effects (IMPLEMENTED Prompt 3)

**New battleState fields:** `wetSecondsAccumulated:0`, `deepCurrentBonus:0`, `tidalForceActive:false`, `maelstromMs:0`, `tsunamiFired:false`, `floodGateFired:false`.

**New helper functions** (added after existing helpers):
- `countWetEnemies()` — count non-broken enemy items with `slowMs > 0`
- `isWet(item)` — true if `item.slowMs > 0`
- `getAdjacentInRow(item,b)` — returns non-broken items immediately left/right of item within its row on board b

**applySlow updated:** Signature is now `applySlow(item, durationMs, src, side='enemy')`.
- `saturation`: if `side==='enemy'` and saturation skill active, multiply `durationMs` by 1.5 before applying.
- After setting `slowMs`, also track `item._slowMaxMs = Math.max(item._slowMaxMs||0, item.slowMs)` for Ebb.

**getSkillMods additions (Water):**
- `current`: speedMult *= (1 - countWetEnemies() * 0.02) when any enemies are Wet.
- `deep_current`: speedMult *= (1 - battleState.deepCurrentBonus) when deepCurrentBonus > 0.
- `tidal_force`: speedMult *= 0.85 when `battleState.tidalForceActive`.

**dealDmg / dealDmgRandom:** After target selection, before `applyDmgTo`:
- `pressure`: +3 dmg if target `isWet(t)` (when side==='enemy').

**applyDmgTo:** Now accepts optional 5th parameter `attackerItem=null`. Pass `battleState.activatingItem` from `dealDmg` and `dealDmgRandom` calls.

**tickSide enemy activation block (after effectFn fires):**
- `drag`: if `isWet(item)`, `applyDmgTo(item, 3, 'Drag', 'enemy')`.
- `undertow`: if `isWet(item)`, spread 0.5s Wet to adjacent non-Wet enemies in same row via `getAdjacentInRow`.

**checkAndBreak (after existing break effects):**
- `ebb`: on player break, reset all Wet enemy `slowMs` to their `_slowMaxMs`.
- `flood_gate`: first player break (once per battle via `floodGateFired`), apply 3s Wet to all non-broken enemies.

**runBattle interval:**
- `maelstrom`: accumulate `maelstromMs`; every 3s, all Wet non-broken enemies take 4 dmg and lose 500ms slowMs.
- `deep_current`: each tick, if any enemies are Wet, accumulate `wetSecondsAccumulated += step/1000`. Set `deepCurrentBonus = Math.min(0.5, Math.floor(wetSecondsAccumulated) * 0.01)`.
- `tidal_force`: each tick, `battleState.tidalForceActive = countWetEnemies() >= 3`.
- `tsunami`: at `battleMs >= 25000` (once per battle via `tsunamiFired`), apply 5s Wet to all non-broken enemies and 3s Jolt to all non-broken allies.

---

## Steel Typed Skills — Battle Effects (IMPLEMENTED Prompt 3)

**New battleState fields:** `forgeTick:0`, `pressurePlateFired:false`, `platingAbsorbedTotal:0`, `ironFormationActive:false`, `impenetrableUsed:{}`, `bulwarkUsed:{}`.

**applyDmgTo updated (Steel):** After shield absorption, when `side==='player'` and absorbed > 0:
- Increment `battleState.platingAbsorbedTotal += absorbed`.
- `temper`: call `dealDmgRandom('enemy', 2, 'Temper')`.
- `cold_iron`: if `attackerItem` is set, permanently add 100ms to `attackerItem.baseActMs`.

`pressure_plate`: first time a player item drops to/below 50% HP (hp > 0), set `pressurePlateFired=true` and add +40 shield to that item.

`impenetrable`: when player item reaches 0 HP, check `battleState.impenetrableUsed[target.id]`; if not used, set to true, set `target.hp=1`, return (survives).

`bulwark`: when player item reaches 0 HP, check if `target.maxShield >= 30` and not yet used; if so, set `target.hp=1`, return (survives).

`iron_formation`: at start of battle (in `resetItemBattleState` after forEach), if all 3 front row slots are filled, set `battleState.ironFormationActive=true`. In `applyDmgTo` when `side==='player'`, apply `dmg = Math.round(dmg * 0.8)` reduction first.

**getSkillMods additions (Steel):**
- `iron_will`: `dmgBonus += Math.floor(platingAbsorbedTotal / 20)`.

**dealDmg / dealDmgRandom:** After target selection, before `applyDmgTo`:
- `alloy`: +4 dmg if `battleState.activatingItem` has `shield > 0` (when side==='enemy').

**tickSide player activation block (inside `if(side==='player')`):**
- `rampart`: every player activation, all other non-broken player items with `maxHp>0` gain +5 shield.
- `vanguard`: if activating item is in back row, all non-broken front row items with `maxHp>0` gain +10 shield.
- `plate_mail`: every player activation, all other non-broken player items with `maxHp>0` gain +3 shield.

**runBattle interval:**
- `forge`: accumulate `forgeTick`; every 10s, find the most wounded (lowest hp/maxHp ratio) non-broken player item with `maxHp>0` and add +25 shield.

---

## Toxic Typed Skills — Battle Effects (IMPLEMENTED Prompt 4)

**New battleState fields:** `virulenceCount:0`, `seepingHealMs:0`, `tippingPointFired:false`, `criticalMassFired:false`, `necrosisMs:0`, `pandemicActive:false`, `colonySide:false`, `lastKillWasPoison:false`.

**New helper functions:**
- `countTotalPoisonStacks(b)` — sum poisonStack across all non-broken items on board b
- `getHeavilyPoisonedEnemies(threshold)` — array of non-broken enemy items with poisonStack >= threshold
- `getMostPoisonedEnemy()` — non-broken enemy item with highest poisonStack, or null

**applyPoison updated (unified):** Handles Pandemic (double stacks before applying), Virulence (every 3rd application +1 extra stack), Outbreak (extra stack when triggered), poisonApplicationsToEnemy tracking. Order: Pandemic → apply → Virulence → Outbreak tracking → checkOutbreak → Outbreak extra stack → fireEvent → log.

**Festering:** In tickSide poison tick loop, `poisonInterval = 900` when festering active and `side==='enemy'`. Immunity (2000ms for player) takes priority over Festering. Applied to enemy-side poison ticking only.

**Seeping:** In tickSide poison tick loop for enemy items, after `applyDmgTo`, heal most wounded alive player ally 1 HP.

**Toxin:** In dealDmg/dealDmgRandom, `dmg += floor(countTotalPoisonStacks(enemyBoard) / 3)` when side==='enemy'.

**Infected:** In dealDmg/dealDmgRandom after target selected, `dmg += target.poisonStack` when side==='enemy'.

**Carrier:** In checkAndBreak enemy break block, transfer `min(2, poisonStack)` to random surviving enemy via applyPoison.

**Sepsis:** In checkAndBreak enemy break block, if `poisonStack >= 5`, deal 10 dmg to all other non-broken enemies via applyDmgTo.

**Tipping Point:** In checkAndBreak enemy break block, if `lastKillWasPoison && !tippingPointFired`, set tippingPointFired=true and apply 3 poison to all surviving enemies. `lastKillWasPoison` set in poison tick loop when `item.hp<=0`, reset after Tipping Point check.

**Colony:** In tickSide player activation block, if front row item activates and back row has alive items, apply 1 poison to random enemy via applyPoison.

**Pandemic:** In applyPoison, `stacks *= 2` before application when active.

**Necrosis:** In runBattle interval, accumulate `necrosisMs += step` (TICK * battleSpeed). At 1000ms, all enemies with 5+ poisonStack lose 3 maxHp (min 1). Reset necrosisMs to 0.
