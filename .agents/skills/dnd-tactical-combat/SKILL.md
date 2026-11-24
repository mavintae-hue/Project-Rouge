---
name: dnd-tactical-combat
description: >
  Bridges D&D 5.5E (2024) character builds and rules into Project-Rouge's deterministic
  turn-based roguelike combat engine: 5-ft grid scale, cover, Opportunity Attacks, Initiative,
  Action/Bonus Action/Reaction/one-spell-slot TurnBudget, D20 Advantage/Crit resolution,
  damage/Resistance/TempHP/Concentration pipeline, 2024 Weapon Masteries, Conditions, and
  Short/Long Rests. Load before writing or editing combat, spell, item, or AI turn logic.
---

# D&D Tactical Combat Engine (5.5E / 2024 Roguelike)

Mechanical specification for Project-Rouge's turn-based combat simulation. Pairs with [.agents/skills/dnd-character-builder/SKILL.md](../dnd-character-builder/SKILL.md) (2024 + 3PP builds/options), [.agents/skills/isometric-diamond-grid/SKILL.md](../isometric-diamond-grid/SKILL.md) (grid & distance), and [.agents/skills/roguelike/SKILL.md](../roguelike/SKILL.md) (run structure & seeds).

---

## 1. Pure Deterministic Simulation Architecture

- **Engine-Agnostic Rules Layer:** Keep all combat, dice, status, and AI rules in pure TypeScript under `src/game/rules/` (or `src/engine/combat/`). **Zero imports** from `pixi.js`, `react`, `next`, or browser DOM APIs.
- **Dedicated Seeded RNG Streams:** Consume an explicit `combatRng` stream (derived from `sub_seed(runSeed, "combat")` per `roguelike` skill §2) — never `Math.random()`, and never share RNG state with floor generation (`mapRng`) or loot spawning (`spawnRng`).
- **Event-Driven Output:** Every action resolves synchronously into pure state transitions + an ordered `CombatEvent[]` log (e.g., `TurnStarted`, `ActorMoved`, `OpportunityAttackTriggered`, `D20Tested`, `DamageApplied`, `ConditionChanged`, `ConcentrationChecked`, `ActorDefeated`). PixiJS animations and React combat logs consume these events without mutating rules state.

---

## 2. Grid Scale & Spatial Combat (D&D 5.5E ↔ Isometric Grid)

- **Tile Scale:** `1 tile = 5 ft`.
  - Standard walking speed `30 ft = 6 tiles` (Difficult Terrain costs `2×` movement: `10 ft` / 2 tiles of budget per tile entered).
  - Standard melee reach = `1 tile` (`5 ft`); Reach weapons = `2 tiles` (`10 ft`).
  - Distance metric: Chebyshev `Math.max(Math.abs(dx), Math.abs(dy))` for tile reach and movement; Euclidean circular radius for spherical spells/auras when appropriate.
- **Cover Rules (Line-of-Sight Ray Check):**
  - **Half Cover:** `+2 AC` and `+2` to DEX saving throws (low obstacles, pillars, intervening creatures).
  - **Three-Quarters Cover:** `+5 AC` and `+5` to DEX saving throws (arrow slits, heavy corners blocking >50% of diamond edges).
  - **Full Cover:** Cannot be directly targeted by attacks or single-target spells requiring a clear path.
- **Opportunity Attacks (OA):**
  - Moving out of a hostile creature's melee reach (`dist <= reach` → `dist > reach`) **without Disengage** (or teleportation/forced movement like `Push`) triggers an OA **Reaction** *before* the leaving step completes.
  - Resolving OA before completing movement ensures effects like Sentinel or drop-to-0-HP stop the actor on the departure tile.

---

## 3. Initiative & 2024 Turn Budget

### Exploration vs. Combat Mode Transition

- **Exploration Mode:** Player moves freely on the diamond grid; traps, hazards, and FOV still evaluate per step.
- **Combat Mode Trigger:** When a hostile actor enters active visibility/aggro range (or an ambush/attack opens combat), roll Initiative for all combatants (`1d20 + DEX mod + Initiative bonuses`).
- **Tie-Breaking:** Sort descending by Initiative total → descending by base DEX score → deterministic actor ID order. Return to Exploration Mode automatically once no active hostiles remain in encounter range.

### `TurnBudget` (Per-Turn Action Economy)

Reset at the **start of the actor's own turn** (note: `reactionUsed` resets at the start of the actor's turn, allowing 1 Reaction per round until their next turn):

```ts
export interface TurnBudget {
  movementRemainingFt: number;       // resets to effective speed (e.g. 30, halved if prone stand-up)
  actionUsed: boolean;               // Attack, Cast a Spell, Dash, Disengage, Dodge, Help, Hide, Ready, Search, Study, Utilize
  bonusActionUsed: boolean;          // BA spells, class BA features, off-hand/Nick mastery rules
  reactionUsed: boolean;             // Opportunity Attack, Shield, Counterspell, class reactions
  freeObjectInteractionUsed: boolean; // 1 free draw/stow/door interaction per turn
  leveledSpellSlotCastThisTurn: boolean; // 2024 rule: at most ONE spell using a spell slot per turn
}
```

- **Strict 2024 Spell-Slot Rule:** In D&D 5.5E (2024), a character may cast **at most one spell using a spell slot per turn** (`leveledSpellSlotCastThisTurn`). Free cantrips and non-slot feature castings (e.g., racial/item casts without expending a slot) do not violate this rule unless another rule restricts them.

---

## 4. D20 Test & Damage Resolution Engine

### Core Derived Modifiers

- **Ability Modifier:** `Math.floor((score - 10) / 2)`
- **Proficiency Bonus (PB):** `Math.ceil(level / 4) + 1`
- **Spell Save DC:** `8 + PB + spellcastingMod` (plus magic item bonuses)
- **Attack Roll Bonus:** `PB (if proficient) + abilityMod + weaponMagicBonus + miscBonus`

### D20 Roll Rules (Advantage, Disadvantage, Crits)

- **Non-Stacking & Mutual Cancellation:** Multiple sources of Advantage still roll only `2d20` take highest; multiple sources of Disadvantage still roll `2d20` take lowest. If **at least one Advantage AND at least one Disadvantage** apply, **they cancel out completely** to a straight `1d20` roll.
- **Attack Natural 1 / Natural 20:**
  - Natural `1` on an attack roll = **automatic miss** regardless of modifiers.
  - Natural `20` on an attack roll = **automatic hit & Critical Hit**: roll all damage dice **twice** (weapon dice, Sneak Attack, Smite, elemental bonus dice), then add flat modifiers (`+STR/DEX`, `+weaponBonus`, etc.) **once**.

### Damage & Concentration Pipeline (Exact Order)

Apply damage in this strict sequence:
1. **Immunity check:** If target is immune to `damageType` → final damage = `0` (skip steps 2–5).
2. **Resistance / Vulnerability:**
   - Resistance: `Math.floor(damage / 2)` (applied once, non-stacking).
   - Vulnerability: `damage * 2` (applied once, non-stacking).
   - *(Apply flat reductions like Heavy Armor Master before Resistance/Vulnerability if specified by 2024 rules, otherwise standard order: modifiers → resistance/vulnerability.)*
3. **Temporary HP absorption:** Subtract from `tempHp` first; overflow carries to `hp`.
4. **HP & Defeat:** Clamp `hp` to `>= 0`. At `0 HP`, trigger Defeated/Dying state (player Death Saves or roguelike Second Wind/Defeat according to run mode).
5. **Concentration Check:** If the damaged actor is concentrating and took `damage > 0` to `hp` or `tempHp`, roll a **CON Saving Throw** with `DC = Math.max(10, Math.floor(damage / 2))`. Drop Concentration immediately on failure (or if incapacitated/defeated).

---

## 5. 2024 Weapon Masteries & Conditions

### Weapon Masteries (PHB 2024 + Harmonized 3PP Weapons)

Support the 8 core 2024 Weapon Masteries as structured tags on weapons (including Steinhardt Trick Weapons, Valda's Firearms, and Grim Hollow Advanced Weapons per `dnd-character-builder`):

| Mastery | Mechanical Hook in Combat Engine |
| :--- | :--- |
| **Cleave** | Once per turn on melee hit, make 1 extra melee attack against a second creature within `1 tile` of the first and within your reach (weapon damage dice only, no ability mod unless negative). |
| **Graze** | On a missed attack roll, still deal damage equal to the ability modifier used for the attack (`>= 0`). |
| **Nick** | Moves the Light-property extra attack from your Bonus Action into the **Attack Action** (once per turn), freeing your Bonus Action. |
| **Push** | On hit (Large or smaller target), push target up to `2 tiles` (`10 ft`) straight away (stop at walls/blocking terrain; no Opportunity Attack triggered). |
| **Sap** | On hit, target has **Disadvantage** on its next attack roll before the start of your next turn. |
| **Slow** | On hit and damage dealt, reduce target's speed by `2 tiles` (`10 ft`) until the start of your next turn (non-stacking). |
| **Topple** | On hit, target makes a **CON save** (`DC = 8 + PB + attackAbilityMod`) or falls **Prone**. |
| **Vex** | On hit and damage dealt, you gain **Advantage** on your next attack roll against that target before the end of your next turn. |

### Conditions & Expiry Hooks

Represent active conditions as `{ id, sourceId, expiresAt, dc? }` with explicit deterministic cleanup hooks:
- `start_of_turn` (e.g., `Sap`, `Slow`, `Dodge`, `Disengage`, `Guiding` self-timers)
- `end_of_turn` (e.g., `Frightened`/`Poisoned` save-ends, `Vex` window)
- `on_hit` / `on_attack` (e.g., `Sap` consumed on next attack, `Vex` consumed on next attack, `Invisibility` broken on attack/cast)
- `concentration` (auto-removed when caster's concentration drops or breaks)

Core conditions to support from the start: `Blinded`, `Charmed`, `Frightened`, `Grappled`, `Incapacitated`, `Invisible`, `Paralyzed` (auto-crit within `1 tile`), `Poisoned`, `Prone` (melee within `1 tile` has Advantage, ranged/beyond `1 tile` has Disadvantage; standing costs half speed), `Restrained`, `Stunned`, `Unconscious`.

---

## 6. Roguelike Run Integration & Data Registries

- **Rest Economy:**
  - **Short Rests** (shrines / campfires between floors or rooms): spend Hit Dice (`hitDie + CON mod`, min `1`) and recharge Short-Rest class resources (Action Surge, Warlock Pact Slots, Monk Focus, Second Wind, Channel Divinity).
  - **Long Rests** (between acts / boss clears): restore full HP, half spent Hit Dice, and Long-Rest resources/spell slots.
- **Enemy Intent & Telegraphing:** Per `roguelike` fairness rules, expose enemy intent on the isometric grid (hovered/targeted tile, heavy-attack windup marker, spell AoE warning diamond) so deaths never feel unpreventable.
- **Data-Driven Registries:** Store classes, subclasses, species, backgrounds/origin feats, spells, weapons, magic items, and monster stat blocks as typed data schemas (`src/game/data/`) rather than hardcoding stats inside combat resolvers.

---

## 7. Verification Checklist (Required Unit Tests)

Before finalizing any combat rule changes, run automated unit tests verifying:
1. **Deterministic Seeded Replay:** Identical initial combat state + action sequence + `runSeed` produces identical `CombatEvent[]` and final HP/positions across runs.
2. **Advantage / Disadvantage Cancellation:** `2 Adv + 1 Disadv` resolves as a single straight `1d20` roll.
3. **Nat 1 & Nat 20 Crits:** Nat 1 misses even against `AC 1`; Nat 20 hits `AC 99` and doubles rolled dice without doubling flat `+STR/+DEX` bonuses.
4. **Damage Pipeline Order:** Resistance rounds down (`Math.floor(dmg / 2)`), Temp HP absorbs before HP, and Concentration DC equals `Math.max(10, Math.floor(dmg / 2))`.
5. **2024 One-Spell-Slot Rule:** Casting a leveled spell slot as a Bonus Action rejects a second leveled spell slot as an Action on the same turn while allowing a Cantrip.
6. **Opportunity Attack Budget:** Leaving melee reach triggers an OA consuming the enemy's `reactionUsed = true`; a second departure in the same round triggers no OA; `Disengage` or `Push` triggers no OA.
