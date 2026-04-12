# ZORP — Technical Brief for Claude Code Sessions

Single self-contained HTML file game. All game logic, CSS, and HTML live in `zorp_v5.html`. No build step, no dependencies, no external files. Open in browser to play.

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

After every trainer win (wins 1–9 only, not win 10):

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

Win 10 → game over. `continueAfterResult` calls `resetRun()` when `G.wins >= 10`.

`advanceDay()` only increments the day and fires `activateCurrentNode()`. It does NOT trigger type select — that happens entirely in the post-win chain before `advanceDay` is called.

---

## Node / Skill Tree

- HEAD node is pre-filled with Potential at init and on every reset. Never leave HEAD empty.
- Typeless players: HEAD has Potential skill. All 10 nodes fillable across 9 wins (HEAD filled at start = win 0).
- Typed players: HEAD has Type Awakening. The win they pick their type gives no node pick. They can fill 8 nodes maximum across 9 wins; at least one node permanently stays locked.
- NECK is NOT skipped by typed players. It becomes unlockable on the next win after they type.
- `getAvailableNodes()` returns all nodes where parent is filled and self is empty (excluding HEAD).

---

## Key State Objects

```js
G          // global game state: gold, honor, wins, day, type, rivalDefeated, dayStep
board      // {front: [5 items], back: [5 items]} — player board
enemyBoard // same shape — enemy board
treeState  // {nodeId: skill object or null} — skill tree state
activeSkills // array of skill objects currently active on player
bannedTypes  // array of type id strings banned from future offerings
```

---

## TYPES Array

Five types (no typeless entry in the array — typeless is handled separately in showTypeSelect):
`electric`, `fire`, `water`, `steel`, `toxic`

Ice does not exist. If you see `id: 'ice'` anywhere, that is a regression — replace with water.

---

## What NOT to Change Without Discussion

- The `effectFn` three-argument signature
- The `side` convention in damage functions (the attackerMods check)
- `applyShield` targeting random (not most wounded — that's `healItem`'s job)
- HEAD node pre-fill in init and resetRun
- Post-win flow sequence — it is intentionally structured and fragile to reordering
- `dealDmgWithStatus` applying to the same target that was damaged (not a separate random target)
