# KingsCoup — Proposed Blueprint Class Structure

Companion doc: see `02-ImplementationSteps.md` for a step-by-step build order.

For the game design decisions this structure is based on (combat rules, progression systems, party/guest/bond design), see the project memory file `kingscoup_game_design.md`.

## Core architecture principle

Combat happens in a **separate Level** from exploration, so anything that needs to survive a level load (current HP/MP, unlocked Skill Wheel nodes, the MC's multiclass picks, bond levels, equipped loadouts) can't live on an Actor in either level — Actors get destroyed on level transition. The design splits cleanly into:

- **Persistent progression data** — lives on three `Object`-parented "manager" Blueprints, manually constructed once by `BP_KingsCoupGameInstance` on Event Init and referenced from it for the rest of the session (see below — `GameInstanceSubsystem` isn't actually Blueprint-creatable in stock Unreal, so this manual-construction pattern is the correction to an earlier assumption in this doc). This is the single source of truth, and what gets saved/loaded.
- **Runtime combat state** — lives on Actors spawned fresh into the Battle level for the duration of one fight, initialized from the persistent data and (for things like current HP) written back when the fight ends.

## A. Foundational framework

```
BP_KingsCoupGameInstance (parent: Game Instance)
  — Event Init constructs one instance of each manager below (Construct Object from Class) and
    stores it in a matching variable. Anything elsewhere that needs this data does:
    Get Game Instance → cast to BP_KingsCoupGameInstance → read the manager variable.
├─ BP_PartyManager (parent: Object)     — per-character runtime state (current HP/MP, Level,
│                                          CurrentXP, unlocked Skill Wheel nodes, equip loadouts),
│                                          inventory, currency; party membership lives in
│                                          PartyGroups (array of F_PartyGroupState — GroupID +
│                                          its own Active/Bench lists), so a sequential
│                                          split-party story beat can spin up a second group
│                                          without losing either group's exact lineup while
│                                          it's off-screen; CurrentGroupID says which one is
│                                          being played. LeadCharacterID tracks who the
│                                          Exploration Pawn currently represents — independent
│                                          of MainCharacterID, which is a standalone field that
│                                          only matters for the multiclass system and never
│                                          changes (NOT part of the per-character struct — it
│                                          only ever applies to one character)
├─ BP_BondManager (parent: Object)      — BondLevel per party member/guest, consumed
│                                          BondEvent flags, unlocked guest abilities
└─ BP_EncounterManager (parent: Object) — "pending encounter" data set by the field trigger
                                           before OpenLevel(BattleLevel), read by GM_Combat
```

## B. Exploration mode

```
GM_Exploration (GameModeBase)
BP_PlayerExplorationPawn (PaperZD Character)      — free-roam movement, no combat logic
BP_FieldEnemyActor                                 — DA_EncounterFormation ref, proximity detection,
                                                       reacts to player, writes to EncounterManager
                                                       + OpenLevel on contact; flag for avoidable vs.
                                                       blocking
BP_NPC_Base                                        — dialogue trigger, checks BondManager for
                                                       available bond scenes at this NPC
```

## C. Combat mode

```
GM_Combat (GameModeBase)                           — reads EncounterManager + PartyManager's
                                                       current party group (GetCurrentPartyGroup)
                                                       on BeginPlay, spawns party + enemy units
GS_Combat (GameState)                               — current battle phase, exposed for UI binding

BP_TurnManager (spawned by GM_Combat)                — meter-based scheduler, no Timer / fixed ticks
  — each unit has a TurnMeter (float) on BP_CombatUnitBase; the manager works out (100 - meter) / Spd
    for every living unit, the smallest acts next, and everyone's meter jumps forward by that time
    (exact turn ratios at any Speed); the actor then subtracts 100 and the leftover carries over
  — one-time random starting meter (StartJitter) at battle start only; Speed is read fresh each time,
    so buffs apply live
  — action economy (later) is the amount subtracted: Quick → less, Standard → 100, Long → deferred
  — evaluates "Help" as a standard action targeting a unit/threat expected to act soon
  — fires BP events for UI/animation: OnTurnStart, OnActionQueued, OnActionResolved

BPI_ActionSource (interface)                       — anything that can take a turn implements this
BPI_Targetable (interface)
BPI_EnemyBrain (interface)                          — SelectNextAction(); default impl = pattern list,
                                                       swappable for a Behavior-Tree-backed impl later

BP_CombatUnitBase (Actor)
├─ StatsComponent           — recomputes GetEffectiveStats() fresh on every query: LeveledCoreStat
│                              (Base + Base×Scale×(Level-1), clamped) + EarnedBonusStats (the one
│                              permanent/banked exception — see Section E) + equipped-weapon direct
│                              modifiers + unlocked Skill Wheel passive bonuses + active StatusEffect
│                              modifiers, all as F_CoreStats — which now includes MaxHP/MaxMP as two
│                              of its 7 fields, so they go through the exact same formula and the
│                              same modifier pipeline as Str/Vit/Int/Spr/Spd. CurrentHP/CurrentMP
│                              (the depleting pool, distinct from the computed Max) come from
│                              PartyManager runtime values, not derived each query.
├─ ResourceComponent         — MP pool + the shared Special/Limit Gauge (builds per-battle;
│                              exposes SpendOnSelf() / SpendOnGuest())
├─ AbilityLoadoutComponent   — currently equipped abilities (pulled from PartyManager loadout),
│                              resolves ability execution against BPI_Targetable
└─ StatusEffectComponent     — active buffs/debuffs (Haste, Sleep, Knockdown, etc. from
                               DA_StatusEffectDefinition)

BP_PlayerCombatUnit (extends CombatUnitBase)
  — reads this character's PartyManager snapshot on spawn
  — MC only: MulticlassComponent (reads PartyManager's standalone multiclass-picks field —
    not a per-character struct field — and resolves which of the 10 combo tier-3 abilities
    are unlocked; also blends the picked classes' DA_ClassGrowthProfile entries into the MC's
    current GrowthScale — see Section E for why this makes retroactive backdating free)
  — optional: GuestAssistComponent (exposes the currently-active guest's move as spendable via
    this unit's Special Gauge)

BP_EnemyCombatUnit (extends CombatUnitBase)
  └─ EnemyBrainComponent — implements BPI_EnemyBrain; default implementation evaluates
                            DA_EnemyDefinition's weighted/conditional pattern list.

WBP_BattleHUD, WBP_ActionMenu (Attack/Ability/Item/Guard/Help/Guest), WBP_TurnOrderBar
```

## D. Data-driven layer (Primary Data Assets)

```
DA_CharacterDefinition       — BaseStats : F_CoreStats (level-1 values, now 7 fields including
                               MaxHP/MaxMP), GrowthScale : F_CoreStats (the per-stat Scale
                               coefficient — growth per level = Base×Scale; this is where
                               "Fighter" vs "Mage" identity actually lives mechanically for the 8
                               fixed characters), ClassName (display-only flavor text, e.g.
                               "Cleric" — nothing reads this to drive logic), PaperZD flipbook refs
                               (per fixed roster member)
DA_EnemyDefinition           — same stat fields as DA_CharacterDefinition (if enemies scale by an
                               internal level/tier), sprite, move-pattern list (conditions →
                               DA_AbilityDefinition)
DA_AbilityDefinition         — action speed (Quick/Standard/Long), cost (MP or Gauge), effect,
                               animation trigger, targeting type
DA_ClassGrowthProfile        — one instance each for Fighter/Ranger/Mage: GrowthScale : F_CoreStats.
                               Used only by MulticlassComponent to blend the MC's current growth
                               from whichever of the 3 have been picked so far — the 8 fixed
                               characters don't reference this, their growth is just their own
                               DA_CharacterDefinition.GrowthScale
DA_MulticlassComboDefinition — 10 entries (FFF, FFR, FFM, RRR, RRF, RRM, MMM, MMF, MMR, MFR),
                               each listing its unique tier-3 abilities
DA_SkillWheelNode            — id, tree position/connections, AP cost, granted ability or a passive
                               bonus expressed as an F_CoreStats delta — no class-prerequisite
                               field (any character can path anywhere)
DA_GuestDefinition            — passive ability ref, active limit-break ability ref, bond
                               requirements to unlock, portrait
DA_EquipmentItem (base)      — Slot, DisplayName, Icon, optional BonusAbility (temporary, while-
                               equipped only)
├─ DA_WeaponItem              — GearStats : F_GearCombatStats (Atk or MagicAtk), optional
│                                DirectStatModifiers : F_CoreStats delta (a weapon that also grants
│                                flat Str/Mag — a live modifier, gone the instant it's unequipped)
└─ DA_ArmorItem                — GearStats : F_GearCombatStats (Armor/MagicArmor), plus
                                 HiddenGrowthContribution : F_CoreStats — small, NOT shown in any
                                 UI, added into the wearer's permanent EarnedBonusStats at the
                                 moment they level up while it's equipped (see Section E)
DA_StatusEffectDefinition    — Haste, Sleep, Knockdown, etc. — duration, StatModifiers :
                               F_CoreStats delta, special flags
DA_EncounterFormation        — which enemies + layout spawn for a given field trigger
DA_BondEventDefinition       — per character: ordered, bankable list of bond scenes/quests,
                               requirements, and rewards (Skill Wheel bonus, guest unlock, etc.)
```

## E. Stats & leveling

- **`F_CoreStats`** (struct, `Content/Blueprints/Core`) — 7 fields: Str (physical dmg), Vit (physical def), Int (magic dmg), Spr (magic def), Spd (turn order *and* turn frequency in the CTB scheduler), **MaxHP**, **MaxMP**. HP/MP were originally kept as separate fields outside this struct, but that meant equipment/Skill Wheel/status effect modifiers — all expressed as an `F_CoreStats` delta — had no way to affect Max HP/MP. Folding them in fixes that for free: a "+50 Max HP" accessory, a Vitality-boosting Skill Wheel node, or a temporary Max-HP-boosting status effect all just work through the same modifier pipeline as every other stat, no separate plumbing needed. One struct type reused for base values, per-level growth *scale*, and every stat modifier (equipment, Skill Wheel passives, status effects, earned bonus) — all just an `F_CoreStats` added or subtracted.
- **`F_GearCombatStats`** (struct) — Atk, Armor, MagicAtk, MagicArmor. A separate "gear layer," distinct from the character's own `F_CoreStats` — matches your sheet's split between innate stats and equipment-only combat stats. Weapons and armor contribute here; damage math (not yet designed) will combine a character's `EffectiveStat` with their `GearCombatStats`.
- **Confirmed scaling formula, reverse-engineered from your spreadsheet's Base/Scale columns**: `LeveledCoreStat = clamp( Base + (Base × Scale) × (Level - 1), Max )`. Verified exactly against your Level 3/50/99 sample rows for every stat (Str, Vit, Mag, Spr, Spd, HP, MP) and the Exp curve. It's genuinely linear — growth-per-level is just `Base × Scale`. Caps (enforced per-field when clamping `F_CoreStats`, not by the struct itself): 255 for Str/Vit/Int/Spr/Spd, 9999 for MaxHP, 999 for MaxMP.
- **HP and MP scale the same way as every other stat** — they're leveled stats, not fixed pools, and now live as two more fields inside `F_CoreStats` itself rather than a separate Base/Scale pair. `CurrentHP`/`CurrentMP` (the depleting-in-battle value) stay on `F_CharacterRuntimeState`, distinct from the computed Max — those two are fundamentally different in kind (consumed/restored by gameplay events, clamped between 0 and the current `EffectiveStats.MaxHP`) from anything else in this section.
- **Level is capped at 99** and lives on `F_CharacterRuntimeState` alongside `CurrentXP`. Assuming standard XP-per-victory leveling (`GM_Combat` splits XP across `GetCurrentPartyGroup().ActiveMemberIDs` on Victory) — flag if level-ups should be story-gated instead, matching how Skill Points work.
- **Stats are recomputed from scratch on every query, never accumulated** — with one deliberate exception (below). `GetEffectiveStats()` always calculates `LeveledCoreStat + EarnedBonusStats + equipped-weapon direct modifiers + Skill Wheel passives + status effects` from current inputs, rather than incrementing a stored value on level-up.
- **This recompute-not-accumulate approach solves the MC's "backdating" requirement for free.** When a new multiclass pick changes `MulticlassComponent`'s blended `GrowthScale`, nothing needs to "retroactively re-apply" past level-ups — the next `GetEffectiveStats()` call just recalculates using the new growth rate against the MC's current `Level`, automatically reflecting the change across every level already gained.
- **`EarnedBonusStats : F_CoreStats`** (new field on `F_CharacterRuntimeState`) — the one deliberate exception to "recompute, never accumulate." This is the hidden, permanent, FF9-style EV mechanic: small values, kept out of any UI for now. When `BP_PartyManager.LevelUpCharacter(CharacterID)` runs, after incrementing `Level` it sums `HiddenGrowthContribution` across that character's currently-equipped `DA_ArmorItem`s and adds the total into `EarnedBonusStats`, permanently — it does not depend on what's equipped later, only on what was equipped *at that level-up moment*. Unlike everything else in this section, this value genuinely must be stored and never re-derived, because it represents banked history, not current state.

## Why this shape

- **Separate manager objects (Party, Bond) instead of one god-object** — bonds/relationships are conceptually independent of combat stats and get touched by exploration/dialogue systems that never need to know about turn order.
- **`AbilityDefinition` is one data type feeding four unlock paths** (multiclass tier-3, Skill Wheel node, equipment bonus, guest reward) — so it's not four separate "ability" systems, just four different *unlock sources* pointing at the same data asset.
- **`EnemyBrainComponent` behind an interface** — ship the simple pattern-list now, swap in a Behavior Tree per-boss once comfortable with them, no rework required elsewhere.
- **Everything persistent lives off-Actor** — this is the one piece that's easy to get wrong later (defaulting to storing progress on the Pawn) and expensive to retrofit once save/load exists, so it's worth locking in before building a single fight.
- **Stats recomputed fresh, never accumulated — except `EarnedBonusStats`, deliberately.** Makes the MC's growth-blend backdating automatic and keeps equipment/buffs from ever going stale, while still allowing the one value that's genuinely permanent history (armor-driven EVs) to actually persist rather than being silently lost every time it's queried.
