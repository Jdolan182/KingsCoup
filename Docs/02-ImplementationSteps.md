# KingsCoup — Step-by-Step Build Plan

Companion doc: see `01-ClassStructure.md` for the full class list and reasoning.

**Recommended approach:** don't build every class in full before testing anything. Work through Phases 0-7 to get one player unit fighting one enemy unit through a complete turn (attack, take damage, win/lose) — a minimal vertical slice — *before* populating full roster/enemy/ability content or building the exploration side, multiclass, skill wheel, or guest/bond systems (Phases 8-13). It's much cheaper to find a broken assumption in the turn-order or data-asset design when there's one ability and one enemy than after you've built ten.

All classes below are pure Blueprint — no C++ required. `Primary Data Asset` doesn't show up in the default "Pick Parent Class" list; tick **Show All Classes** (or just type into the search box) to find it. Note: an earlier version of this doc assumed `Game Instance Subsystem` was Blueprint-creatable the same way — it isn't (that class isn't marked `Blueprintable` in stock Unreal). Phase 3 below uses a corrected approach: plain `Object`-parented "manager" Blueprints, manually constructed by the GameInstance instead.

---

## Phase 0 — Project & folder setup

1. In the Content Browser, create top-level folders: `Data`, `Blueprints`, `Characters`, `UI`, `Levels`.
2. Under `Data`, create subfolders: `Abilities`, `Characters`, `Enemies`, `Items`, `SkillWheel`, `Multiclass`, `Guests`, `Bonds`, `Encounters`, `StatusEffects`.
3. Under `Blueprints`, create subfolders: `Core`, `Combat`, `Exploration`, `UI`.
4. Under `Levels`, create `Exploration` and `Battle` subfolders (or one folder per exploration zone once you have more than one).

## Phase 1 — Shared enums & structs

Create these first since almost everything else references them (Content Browser → Add → Blueprints → Structure / Enumeration). Put all of them in `Content/Blueprints/Core` — they're foundational types shared across Data, Combat, Exploration, and UI, not specific to any one of those folders.

1. `E_ActionSpeed` — Quick, Standard, Long
2. `E_AbilityCostType` — MP, SpecialGauge, None
3. `E_TargetType` — SingleEnemy, AllEnemies, SingleAlly, AllAllies, Self
4. `E_MulticlassOption` — Fighter, Ranger, Mage
5. `F_CoreStats` (struct) — Str, Vit, Int, Spr, Spd, MaxHP, MaxMP (all float, 7 fields). HP/MP scale with the exact same Base×Scale formula as the other 5 (confirmed from your spreadsheet), so they belong in this struct rather than as separate fields — that also means equipment/Skill Wheel/status-effect modifiers (all expressed as an `F_CoreStats` delta) can affect Max HP/MP automatically, with no extra plumbing. Reused everywhere a stat *value* or a stat *delta* is needed: base stats, per-level growth scale, equipment modifiers, Skill Wheel passive bonuses, status effect modifiers, earned bonus stats — same 7 fields each time, meaning differs by which variable it's assigned to. (Name the last two fields `MaxHP` and `MaxMP` for consistency — not just `MP`.)
6. `F_GearCombatStats` (struct) — Atk, Armor, MagicAtk, MagicArmor (all float). The separate "gear layer" combat stats from your spreadsheet's "From Gear" section — distinct from `F_CoreStats`. Weapons and armor contribute this; damage math combines it with a unit's `F_CoreStats`.
7. `F_CharacterRuntimeState` (struct) — CharacterID, CurrentHP, CurrentMP, Level (int, capped 99), CurrentXP, EarnedBonusStats (`F_CoreStats` — the permanent, hidden, armor-driven bonus; see Phase 5 and Phase 9), UnlockedSkillNodeIDs (array), EquippedAbilityIDs (array), SkillPointsAvailable. **No MulticlassPicks field here** — multiclassing only ever applies to the MC, so it doesn't belong on a struct meant to represent any party member (see Phase 3).
8. `F_BondProgress` (struct) — CharacterID, BondLevel, ConsumedBondEventIDs (array)
9. `F_PartyGroupState` (struct) — GroupID (Name), ActiveMemberIDs (array, Object Ref Soft → `DA_CharacterDefinition`), BenchMemberIDs (array, same type). Supports sequential split-party story beats — see Phase 3.
10. `F_EquippedGearEntry` (struct) — CharacterID (Object Ref Soft → `DA_CharacterDefinition`), Weapon (Object Ref Soft → `DA_WeaponItem`), Head/Chest/Gloves/Accessory (each Object Ref Soft → `DA_ArmorItem`) — one field per `E_EquipmentSlot` value; Chest/Gloves/Accessory/Head all share the same `DA_ArmorItem` class, distinguished by that instance's own `Slot` field

## Phase 2 — Data Asset classes

For each: Content Browser → Add → Blueprint Class → Pick Parent Class → search "Primary Data Asset" (Show All Classes). Create the class Blueprint itself (and any test instances made from it) directly in the folder noted below — these are the `Data/*` subfolders from Phase 0, no separate split between "class" and "instance" locations needed at this project size.

1. `DA_AbilityDefinition` — `Content/Data/Abilities`. Name, E_ActionSpeed, E_AbilityCostType + Amount, Effect (start simple: an enum EffectType + float Magnitude), E_TargetType, animation tag/name, icon.
2. `DA_StatusEffectDefinition` — `Content/Data/StatusEffects`. Name, Duration, StatModifiers (`F_CoreStats` delta), special flags (e.g. `bVulnerableToFinisher` for the coup-de-grâce case).
3. `DA_CharacterDefinition` — `Content/Data/Characters`. DisplayName, ClassName (display-only text, e.g. "Cleric" — nothing reads it programmatically), BaseStats (`F_CoreStats`, level-1 values — now includes MaxHP/MaxMP, no separate fields needed), GrowthScale (`F_CoreStats` — the per-stat Scale coefficient; growth per level = Base×Scale, and this is where each fixed character's class identity actually lives mechanically), PaperZD flipbook/animation references, starting ability IDs.
4. `DA_EnemyDefinition` — `Content/Data/Enemies`. DisplayName, same stat fields as `DA_CharacterDefinition` (if enemies scale by tier/level), sprite/animation refs, move-pattern list (array of structs: `DA_AbilityDefinition` ref + trigger condition + weight).
5. `DA_ClassGrowthProfile` — `Content/Data/Multiclass`. One instance each for Fighter, Ranger, Mage: GrowthScale (`F_CoreStats`). Used only by the MC's `MulticlassComponent` (Phase 10) — the 8 fixed characters use their own `DA_CharacterDefinition.GrowthScale` instead and never reference this.
6. `DA_MulticlassComboDefinition` — `Content/Data/Multiclass`. ComboKey (e.g. "FFM", built from a sorted combination so order doesn't matter), array of unique tier-3 `DA_AbilityDefinition` refs.
7. `DA_SkillWheelNode` — `Content/Data/SkillWheel`. NodeID, tree position/connections (array of connected NodeIDs), AP cost, granted ability OR a passive bonus expressed as an `F_CoreStats` delta. Deliberately no class-prerequisite field.
8. `DA_EquipmentItem` (parent for the next two — common fields: Slot, DisplayName, Icon, optional BonusAbility ref for a temporary while-equipped-only ability) and its two subclasses `DA_WeaponItem`, `DA_ArmorItem` — all three in `Content/Data/Items`.
   - `DA_WeaponItem` (extends `DA_EquipmentItem`) — GearStats (`F_GearCombatStats`, typically Atk or MagicAtk), optional DirectStatModifiers (`F_CoreStats` delta — some weapons also grant flat Str/Mag while equipped; this is a live modifier, not banked).
   - `DA_ArmorItem` (extends `DA_EquipmentItem`) — GearStats (`F_GearCombatStats`, typically Armor/MagicArmor), HiddenGrowthContribution (`F_CoreStats` — small values, never shown in UI; see Phase 9 for how this becomes a permanent bonus).
9. `DA_GuestDefinition` — `Content/Data/Guests`. DisplayName, passive ability ref, active limit-break ability ref, bond-level requirement to unlock, portrait.
10. `DA_EncounterFormation` — `Content/Data/Encounters`. Array of (DA_EnemyDefinition ref + spawn position).
11. `DA_BondEventDefinition` — `Content/Data/Bonds`. CharacterID, ordered list of bond scenes/quests, requirements, reward (Skill Wheel bonus / guest unlock / stat bonus).

Once the classes exist, right-click each in the Content Browser → Create Data Asset, and make one or two real test instances (e.g. `DA_Ability_Attack`, `DA_Enemy_TestSlime`) — you'll need them for the Phase 7 playtest.

## Phase 3 — GameInstance & Managers

All four classes go in `Content/Blueprints/Core`.

1. Create `BP_KingsCoupGameInstance` (parent: Game Instance). Set it in Project Settings → Maps & Modes → Game Instance Class.
2. Create `BP_PartyManager` (parent: **Object**). Add:
   - `RosterState` — Array of Struct → `F_CharacterRuntimeState` (every unlocked character, regardless of current party status)
   - `PartyGroups` — Array of Struct → `F_PartyGroupState`. Normally one entry (`GroupID = "Default"`). During a sequential split-party story beat, a second entry temporarily exists alongside it, each with its own independent Active/Bench lineup — this is what lets the game cut away to a different group and later cut back with that group's exact setup preserved, rather than just tracking one flat "current party."
   - `CurrentGroupID` — Name — whichever group the player is actively playing right now
   - `LeadCharacterID` — Object Reference (Soft) → `DA_CharacterDefinition` — whoever `BP_PlayerExplorationPawn` currently represents. **Not the same thing as `MainCharacterID` below** — this can be anyone and changes whenever the story hands control to a different character; `MainCharacterID` never changes and only matters for the multiclass system.
   - `Currency` — Integer
   - `Inventory` — Array of Object Reference (Soft) → `DA_EquipmentItem`
   - `EquippedGear` — Array of Struct → `F_EquippedGearEntry` (needed by `LevelUpCharacter` below)
   - `MainCharacterID` — Object Reference (Soft) → `DA_CharacterDefinition` — standalone, not inside `RosterState`, since exactly one character ever uses it
   - `MainCharacterMulticlassPicks` — Array of Enum → `E_MulticlassOption`, max 3 — standalone, same reasoning

   Add functions: `GetCharacterState(CharacterID)`, `ApplyCombatResult(...)`, `SpendSkillPoint(CharacterID, NodeID)`, `AddMulticlassPick(Choice)`, `LevelUpCharacter(CharacterID)` (see Phase 9), `RespecCharacter(CharacterID)` (see Phase 11), `UnlockGuestAbility(GuestID)` (see Phase 12), `SetLeadCharacter(CharacterID)`, and the party-group set:
   - `GetCurrentPartyGroup()` — returns the `F_PartyGroupState` matching `CurrentGroupID`. Anything that used to read a flat "active party" (like `GM_Combat` spawning the battle party in Phase 6) goes through this instead.
   - `SwitchToPartyGroup(GroupID)` — changes `CurrentGroupID`. Cutting back to a group already in `PartyGroups` restores its exact prior setup automatically, since nothing about it was touched while it was inactive.
   - `CreatePartyGroup(GroupID, InitialActiveMembers, InitialBenchMembers)` — called at a story split point to spin up the second group.
   - `MergePartyGroups(SourceGroupID, TargetGroupID)` — called at reunion; folds the source group's members into the target and removes the source's entry from `PartyGroups`.
   - `SetCharacterAway(CharacterID)` / `SetCharacterAvailable(CharacterID, GroupID)` — remove a character from whichever group currently holds them, or add them to a specific group's bench. `RosterState` is untouched either way — a character who's "away" still has all their saved stats/progress, they're just not selectable in any group right now.
3. Create `BP_BondManager` (parent: Object). Add: array of `F_BondProgress`, function `TriggerBondEvent(CharacterID, EventID)`.
4. Create `BP_EncounterManager` (parent: Object). Add: `PendingFormation` (DA_EncounterFormation ref), `ReturnLevelName`, `ReturnSpawnPointID`, functions `SetPendingEncounter(...)` / `ClearPendingEncounter()`.
5. On `BP_KingsCoupGameInstance`, add three Object Reference variables: `PartyManager` → `BP_PartyManager`, `BondManager` → `BP_BondManager`, `EncounterManager` → `BP_EncounterManager`.
6. Override **Event Init** on `BP_KingsCoupGameInstance` (right-click in its event graph, search "Init" — it's a built-in GameInstance event). Use **Construct Object from Class** three times, once per manager, and store each result into its matching variable from step 5.

**Access pattern for every later phase**: anywhere below that says something like `BP_PartyManager.SpendSkillPoint(...)`, that means: **Get Game Instance** → cast to `BP_KingsCoupGameInstance` → read the `PartyManager` variable → call the function on it. There's no "Get Subsystem" shortcut node for these — they're just object references you fetch through the GameInstance.

## Phase 4 — Interfaces

Content Browser → Add → Blueprints → Blueprint Interface.

1. `BPI_ActionSource` — `GetSpeed()`, `TakeTurn()`
2. `BPI_Targetable` — `ApplyEffect(DA_AbilityDefinition, Instigator)`, `GetTargetType()`
3. `BPI_EnemyBrain` — `SelectNextAction()`

## Phase 5 — Combat components & unit base classes

All classes in this phase go in `Content/Blueprints/Combat`.

1. Create Actor Components: `BP_StatsComponent`, `BP_ResourceComponent`, `BP_AbilityLoadoutComponent`, `BP_StatusEffectComponent`, `BP_EnemyBrainComponent` (implements `BPI_EnemyBrain`, holds default pattern-list logic reading its owner's `DA_EnemyDefinition`).
   - `BP_ResourceComponent` also needs `CurrentHP`/`CurrentMP` (Integer, matching `F_CharacterRuntimeState`'s types) alongside `SpecialGauge` — during a battle these are the live, depleting working copies; `F_CharacterRuntimeState` only holds the persistent value read at spawn and written back at battle-end.
   - In `BP_StatsComponent`, add a pure function `CalculateLeveledStat(Base, Scale, Level, Max) -> float` = `Clamp(Base + (Base × Scale) × (Level - 1), 0, Max)` — this is your confirmed spreadsheet formula, verified exact against your Level 3/50/99 sample rows. Call it once per field of `DA_CharacterDefinition.BaseStats`/`GrowthScale` (now 7 fields: Str/Vit/Int/Spr/Spd with Max 255, plus MaxHP with Max 9999 and MaxMP with Max 999).
   - Add `GetEffectiveStats() -> F_CoreStats` that recomputes fresh every call: `CalculateLeveledStat(...) per stat + EarnedBonusStats + equipped-weapon DirectStatModifiers + Skill Wheel passives + status effects`. Don't cache/accumulate this — it's what makes MC growth changes (Phase 10) apply retroactively for free.
   - Add `GetGearCombatStats() -> F_GearCombatStats` — sums the equipped weapon's and armor's `GearStats` (Atk/Armor/MagicAtk/MagicArmor). Kept separate from `GetEffectiveStats()` since it's a distinct stat layer per your design.
   - Also create `BP_MulticlassComponent` and `BP_GuestAssistComponent` here (parent: Actor Component) — leave both **empty** for now. Their real logic doesn't land until Phase 10 (multiclass blending) and Phase 12 (guest moves) respectively; they just need to exist so `BP_PlayerCombatUnit` has somewhere to attach them in step 3.
2. Create `BP_CombatUnitBase` (parent: Actor). Add the four non-brain components (`BP_StatsComponent`, `BP_ResourceComponent`, `BP_AbilityLoadoutComponent`, `BP_StatusEffectComponent`). Implement `BPI_ActionSource` and `BPI_Targetable`. Add a `bIsEnemySide` (Boolean) variable that `IsEnemyUnit` just returns — subclasses set its default differently rather than overriding the function.
3. Create `BP_PlayerCombatUnit` (parent: CombatUnitBase). Add `CharacterID` (Object Ref, Soft → `DA_CharacterDefinition`), plus the `BP_MulticlassComponent` and `BP_GuestAssistComponent` components from step 1. Set `bIsEnemySide` default to `false`. On BeginPlay, pull this character's `F_CharacterRuntimeState` from `BP_PartyManager.RosterState` (matched by `CharacterID`) and initialize the components — `CurrentHP`/`CurrentMP` carry over from last time.
4. Create `BP_EnemyCombatUnit` (parent: CombatUnitBase). Add `EnemyDefinition` (Object Ref, Soft → `DA_EnemyDefinition`), `Level` (Integer), plus `BP_EnemyBrainComponent`. Set `bIsEnemySide` default to `true`. On BeginPlay, initialize `StatsComponent` from `EnemyDefinition.BaseStats`/`GrowthScale` + `Level`, and set `CurrentHP`/`CurrentMP` to the freshly-computed Max values (enemies always spawn full health, no persistent record to carry over).
   - **Enemies never use stat scaling.** `GrowthScale` stays 0 on every `DA_EnemyDefinition` — a higher-level variant of an enemy (e.g. "Goblin Lv5" vs "Goblin Lv10") is its own separate Data Asset with `BaseStats` already authored at the intended final values, not a computed scale-up. `Level` is still set on the enemy's `StatsComponent` (used for turn-order/UI display and any future level-gated logic), it just has no effect on the stat math since Scale = 0.
   - `StatsComponent.MaxHPCap`/`MaxMPCap` default to 9999/999 (same ceiling as players). Before calling `GetEffectiveStats()`, `BP_EnemyCombatUnit.BeginPlay` overrides both to `EnemyDefinition.BaseStats.MaxHP`/`MaxMP` (via `SetBaseStats`'s `Output_Get` pass-through → `BreakFCoreStats` → `SetMaxHPCap`/`SetMaxMPCap`). Since `GrowthScale` is 0, the leveled result always equals `Base` anyway, so this cap is a no-op clamp in practice — its real purpose is just to stop the player's flat 9999/999 default from silently capping an enemy authored above that (e.g. a boss with `BaseStats.MaxHP` of 50000).

## Phase 6 — Turn Manager & Combat GameMode

1. Create `BP_TurnManager` (parent: Actor, `Content/Blueprints/Combat`, spawned by GM_Combat). This is a **meter-based scheduler with no Timer and no fixed ticks** — Speed affects both who goes first *and* how often. (The earlier timer/`AdvanceSchedule` design was dropped: fixed steps cap high-Speed units at one turn per step and break turn ratios.)
   - **On `BP_CombatUnitBase`:** `TurnMeter` (Float, 0) and `TurnManager` (`BP_TurnManager`, hard ref — set by `TakeNextTurn` before `TakeTurn`).
   - **On `BP_TurnManager`:** `ActiveUnits` (Array of `BP_CombatUnitBase`, hard), `ActionThreshold` (Float, 100), `StartJitter` (Float, 15 — tunable), `ReviveStartMeter` (Float, 0 — tunable, unused until revive exists), dispatchers `OnTurnStart(ActingUnit)` and `OnActionResolved(ActingUnit, Target)` (both `BP_CombatUnitBase`, hard).
   - `InitializeCombat(Units: Array<BP_CombatUnitBase>)`: clear `ActiveUnits`, copy `Units` in, set each unit's `TurnMeter = Random Float in Range(0, StartJitter)` — the **one-time** initiative roll (D&D feel), never repeated. Then call `TakeNextTurn`.
   - `IsAlive(Unit) -> Boolean` (pure): `Unit.GetBP_ResourceComponent.GetCurrentHP > 0`.
   - `TakeNextTurn()`: for each living unit, `Spd = GetSpeed()` and `ThisTime = Max((ActionThreshold − TurnMeter) / Spd, 0)`. The smallest `ThisTime` acts next (tie → higher Spd; still tied → first in the array, so `GM_Combat` adds players before enemies). If nobody is alive, stop. Otherwise add `GetSpeed × BestTime` to every living unit's `TurnMeter`, set the actor's `TurnMeter` to exactly `ActionThreshold` (avoids float drift), set `TurnManager = Self`, broadcast `OnTurnStart`, then call `TakeTurn()`. Speed is read fresh every time, so buffs apply live with no extra code.
   - `ResolveTurn(ActingUnit, Ability, Target)` (`Ability` is a hard `DA_AbilityDefinition`; units are hard `BP_CombatUnitBase`): `ExecuteAbility` → `OnActionResolved` → `ActingUnit.TurnMeter −= ActionThreshold` (leftover carries over) → `StatusEffectComponent.TickDurations` on the acting unit → if `GetOpposingUnits(ActingUnit)` is empty the battle is over (set `GS_Combat.BattlePhase`), otherwise `TakeNextTurn`.
   - `GetOpposingUnits(RequestingUnit) -> Array<BP_CombatUnitBase>`: living units in `ActiveUnits` whose `IsEnemyUnit` differs from the requester's. This is also what finally fills `SelectNextAction`'s target.
   - `ResolveHelpAction(ActingUnit, TargetThreat)`: empty stub for now.
   - **Stat floor:** `GetEffectiveStats` clamps Str/Vit/Int/Spr/Spd to a minimum of 1 (applied after future modifiers); MaxHP/MaxMP may be 0. This also means Speed can never be 0, so the time division is safe.
   - **Dead units:** skipped by `IsAlive`. Revive (not built yet) will set the meter to `ReviveStartMeter` so how it feels can be tuned later.
   - **Status durations:** a status with Duration N ticks down at the end of the *affected unit's own* turns, and an effect a unit casts on itself skips the tick for the turn it was cast.
   - **Action categories (Quick/Standard/Long) and Haste/Slow are NOT in the first pass** — every action costs 100. The settled rules are in the table below and will change how much `ResolveTurn` subtracts. Haste/Slow are category-shifts only; plain Speed buffs/debuffs are ordinary stat modifiers (exact Haste behaviour to be tuned by feel later).
   - Quick's exact meter cost is still open: "immediate bonus turn" (subtract ~0, capped at one chain) vs subtracting ~50.

   | Base category | Normal | Hasted | Slowed |
   |---|---|---|---|
   | Quick | 1 bonus chain (max) | unchanged | acts like Standard |
   | Standard | normal | acts like Quick | acts like Long |
   | Long | resolves at start of next turn, +1 normal turn | acts like Standard | unchanged |
2. Create `GM_Combat` (parent: GameModeBase, `Content/Blueprints/Combat`). Variables: `TurnManager` (`BP_TurnManager`, hard), `SpawnedUnits` (Array of `BP_CombatUnitBase`, hard). On BeginPlay: get the game instance's `PartyManager` and `EncounterManager`, read `GetCurrentPartyGroup().ActiveMemberIDs` (Array of soft `DA_CharacterDefinition`) and the pending formation's entries (`F_EncounterEntry`: `Enemy` soft `DA_EnemyDefinition`, `SpawnSlotIndex` Integer, `Level` Integer). Spawn **players first**, then enemies, adding each to `SpawnedUnits`. On the unit classes tick Instance Editable + Expose on Spawn for `CharacterID`, `EnemyID` and `Level`, so `Spawn Actor from Class` sets them before `BeginPlay`. Then spawn `BP_TurnManager`, set `TurnManager`, and call `InitializeCombat(SpawnedUnits)` (which starts the first turn). If the formation is empty, fall back to one test player and one test enemy. Spawn points: Target Points tagged `PlayerSpawn` / `EnemySpawn`.
3. Create `GS_Combat` (parent: **GameStateBase**, to pair with GameModeBase). Add `E_BattlePhase` (Intro/InProgress/Victory/Defeat, in `Content/Blueprints/Core`) and a `BattlePhase` variable.
   - **First playtest scope:** player units auto-use `DA_Ability_BasicAttack` on a random living enemy (the Phase 7 menu replaces this); placeholder damage is `EffectMagnitude + Str − target Vit`, minimum 1, with a `Print String` of damage and remaining HP for tuning.
4. Create your Battle level, set its World Settings → GameMode Override to `GM_Combat`.

## Phase 7 — Combat UI + first playtest

1. Create `WBP_BattleHUD` — per-unit HP/MP/Gauge bars, bound to `BP_StatsComponent`/`BP_ResourceComponent` via event dispatchers (`OnStatsChanged`).
2. Create `WBP_ActionMenu` — buttons (Attack/Ability/Item/Guard/Help/Guest) that read the acting unit's `BP_AbilityLoadoutComponent` and submit the chosen action to `BP_TurnManager`.
3. Create `WBP_TurnOrderBar` — do this **last** in Phase 7, once turns work. It needs a `BP_TurnManager` function that simulates the next few turns from copies of the current meters (without changing the real ones); for the first playtest just print the acting unit's name.
4. Add the HUD widget to viewport from `GM_Combat` or its PlayerController's BeginPlay.
5. **Playtest checkpoint:** place one `BP_PlayerCombatUnit` and one `BP_EnemyCombatUnit` (using your Phase 2 test data assets) directly in the Battle level, PIE, and verify a full turn resolves correctly — action selection, damage, HP updates, and win/lose triggers `GS_Combat`'s phase change. Fix anything broken here before moving on.

## Phase 8 — Exploration mode

1. Create `GM_Exploration` (parent: GameModeBase).
2. Create `BP_PlayerExplorationPawn` (parent: a PaperZD character class) with free-roam movement (pair with the `unreal-enhanced-input` skill for the Input Action/Mapping Context setup).
3. Create `BP_FieldEnemyActor` — sphere collision for player-proximity detection, `DA_EncounterFormation` reference, `bIsAvoidable` flag. On player overlap: call `BP_EncounterManager.SetPendingEncounter(...)`, then `Open Level` to the Battle level.
4. Create `BP_NPC_Base` — interact trigger, dialogue hook, queries `BP_BondManager` for available bond scenes at this NPC.
5. Build a small test Exploration level with your player pawn and one `BP_FieldEnemyActor`; verify walking into it correctly opens the Battle level from Phase 7.

## Phase 9 — Return-trip wiring

1. In `GM_Combat`, on Victory/Defeat: write updated HP/MP (and any consumed items) back to `BP_PartyManager` via `ApplyCombatResult`. On Victory, also grant XP split across `GetCurrentPartyGroup().ActiveMemberIDs`.
2. Implement `BP_PartyManager.LevelUpCharacter(CharacterID)`, called whenever `CurrentXP` crosses the threshold for the next level (`XP(Level) = 1000 + 800 × (Level - 1)` per your confirmed Exp curve): increment `Level`, then — this is the hidden EV mechanic — look up that character's currently-equipped `DA_ArmorItem`s in `EquippedGear`, sum their `HiddenGrowthContribution`, and add the total into `EarnedBonusStats`. This is the one value in the whole stat system that's genuinely written and kept rather than recomputed — it depends on what was equipped *at this moment*, not on current state, so it can't be derived later. No UI reads this yet.
3. Call `BP_EncounterManager.ClearPendingEncounter()`, then `Open Level` back to `ReturnLevelName`, spawning the player at `ReturnSpawnPointID` instead of the level's default start.
4. Playtest the full loop: walk into an enemy → fight → win → gain XP/level if applicable → land back in the exploration level in the right spot with updated HP.

## Phase 10 — MC multiclass

1. Build `WBP_MulticlassPicker` — shown at each of the 3 story milestones, calls `BP_PartyManager.AddMulticlassPick(Choice)`.
2. In `MulticlassComponent`, after the 3rd pick: sort the 3 choices from `BP_PartyManager.MainCharacterMulticlassPicks` into a canonical order, build a lookup key (e.g. "FFM"), find the matching `DA_MulticlassComboDefinition` (a DataTable or Map keyed by that string works well), and grant its unique tier-3 abilities into the MC's `EquippedAbilityIDs`/unlocked-ability list (the normal per-character struct field — only the picks themselves live outside it).
3. On picks 1 and 2, grant that pick's generic per-class abilities immediately instead of waiting for the combo.
4. After **every** pick (not just the 3rd), have `MulticlassComponent` recompute the MC's current `GrowthScale` as the average of the `DA_ClassGrowthProfile.GrowthScale` for each class picked so far. Store this on the component (or directly overwrite a `CurrentGrowthScale` field the MC's `StatsComponent` reads instead of a `DA_CharacterDefinition.GrowthScale`, since the MC doesn't have one fixed value). Because `GetEffectiveStats()` (Phase 5) recomputes from `Level` and `GrowthScale` fresh every time rather than accumulating past level-ups, this blend automatically "backdates" — the very next stat query after a pick reflects the new growth rate across every level the MC already has, with no extra logic needed.
5. For every other character, `DA_CharacterDefinition.ClassName` is display-only flavor text — nothing in code branches on it. Their kit comes entirely from their starting abilities plus the shared, unrestricted Skill Wheel, and their `GrowthScale` is fixed for the whole game.

## Phase 11 — Skill Wheel

1. Build `WBP_SkillWheel` — visual node-tree UI reading all `DA_SkillWheelNode` assets, showing locked/available/unlocked state per character based on `F_CharacterRuntimeState.UnlockedSkillNodeIDs` and `SkillPointsAvailable`.
2. Implement `BP_PartyManager.SpendSkillPoint(CharacterID, NodeID)` — validate the node is adjacent to an already-unlocked node and AP is available, then add it and deduct the cost.
3. Implement `BP_PartyManager.RespecCharacter(CharacterID)` — refund all spent points (minus a cost if you land on one being non-free) and clear that character's `UnlockedSkillNodeIDs`.
4. Hook `GrantSkillPoints(CharacterID, Amount)` up to your story-milestone trigger system (however cutscenes/story flags are being fired).

## Phase 12 — Guests & Bonds

1. Wire `BP_NPC_Base` interactions (or dialogue system) to call `BP_BondManager.TriggerBondEvent(CharacterID, EventID)`.
2. In `TriggerBondEvent`, look up the matching `DA_BondEventDefinition`, check prerequisites (previous events consumed), raise `BondLevel`, and apply the reward.
3. When a guest's bond-level requirement is met, call `BP_PartyManager.UnlockGuestAbility(GuestID)` — this makes both the passive and active ability permanently selectable in `BP_AbilityLoadoutComponent`'s pool for any character, flagged `bIsGuestAbility` for UI display.
4. For the currently-active guest in a given battle, have `GM_Combat` pass a lightweight guest reference (not a full CombatUnit) so `GuestAssistComponent` knows which guest move the shared Special Gauge can currently trigger.

## Phase 13 — Content pass & beyond

With the loop validated end-to-end, this is where breadth work happens:

1. Populate the full roster: 8-9 `DA_CharacterDefinition` assets, their starting/skill-wheel abilities, and portraits/sprites.
2. Build out the enemy roster and `DA_EncounterFormation` variety across your exploration levels.
3. Flesh out the `DA_AbilityDefinition` library (attacks, spells, guest limit breaks, equipment bonus abilities).
4. Populate the Skill Wheel (`DA_SkillWheelNode` tree, ~8-10 unlockable points' worth) and the 10 `DA_MulticlassComboDefinition` entries.
5. Populate `DA_BondEventDefinition` per character and the guest roster.
6. Only after that: save/load (serialize `BP_PartyManager` + `BP_BondManager`), active-3 + bench party swapping, equipment UI, and any platforming/traversal scope decision for exploration (parked per the design doc).

## Parked for later (not scheduled into a phase yet)

- **A visible UI for `EarnedBonusStats`** — the hidden armor-driven EV mechanic (Phase 9) is already in the core build, just with no UI exposing it. Revisit if it turns out to be worth surfacing as a real min-max mechanic rather than invisible variety.
- **The actual damage formula** — how `EffectiveStat` (Str/Vit/Int/Spr) combines with `GearCombatStats` (Atk/Armor/MagicAtk/MagicArmor) and an ability's Power value into a final damage number. Your spreadsheet has an open "?" here too — worth a dedicated pass once combat is otherwise working end-to-end.
