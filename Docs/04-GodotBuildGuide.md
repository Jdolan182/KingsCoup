# KingsCoup — Godot Build Guide (you write the code), Phases 0–6

A "what to make and what logic to write" guide for building the design in **Godot 4.7 / GDScript** yourself. There is **no implementation code** here: each item lists the file, its fields/types, the functions (name, inputs, output) and the logic as plain steps, plus Godot features to look up.

If you get stuck, `03-GodotImplementationSteps.md` has a full worked version of everything below (same file and class names) — try each piece first, then compare. The design rules come from `01-ClassStructure.md` / `02-ImplementationSteps.md` and the memory file.

**Conventions used below**
- `Type` after a name = the GDScript static type. `Array[X]` = typed array.
- "Export" = `@export` so it shows in the Inspector. "Look up:" = the Godot class/method names to read in the docs.
- Don't name variables `str` or `int` (built-ins). Stats are `strength`, `vitality`, `intelligence`, `spirit`, `speed`.
- Resources are shared by reference — never edit a loaded definition at runtime without `duplicate(true)`.

---

## Phase 0 — Project setup

Make:
1. A new Godot 4.7 project. Default texture filter = **Nearest** (pixel art).
2. The folder tree: `data/` (abilities, status_effects, characters, enemies, items, skill_wheel, multiclass, guests, bonds, encounters), `scripts/` (core, data, autoload, combat), `scenes/` (combat, battle, exploration), `ui/`.
3. A `.gitignore` containing `.godot/`.

Done when: project opens, folders exist, first commit made.

---

## Phase 1 — Enums and value types (`scripts/core/`)

### 1.1 `enums.gd` — one script holding every enum
Make a script with a `class_name` and no logic, only these enums:
- `ActionSpeed`: Quick, Standard, Long
- `AbilityCostType`: None, MP, SpecialGauge
- `TargetType`: SingleEnemy, AllEnemies, SingleAlly, AllAllies, Self
- `EffectType`: PhysicalDamage, MagicDamage, Heal, RestoreMP, ApplyStatusEffect
- `MulticlassOption`: Fighter, Ranger, Mage
- `EquipmentSlot`: Weapon, Head, Chest, Gloves, Accessory
- `BattlePhase`: Intro, InProgress, Victory, Defeat

Look up: GDScript `enum`, and how enums on a `class_name` script are read from other scripts.

### 1.2 `game_data.gd` — base for every definition
A `Resource` subclass with one exported field: `id: StringName`. Every definition (Phase 2) extends this so the Registry can index them.

### 1.3 `core_stats.gd` — the stat bundle
A `Resource` with seven exported floats: `strength, vitality, intelligence, spirit, speed, max_hp, max_mp`. It is used for stat **values** and stat **deltas** (modifiers) alike.
Function to write: `plus(other: CoreStats) -> CoreStats` — returns a **new** CoreStats with every field added (don't mutate either input).

### 1.4 `stats_math.gd` — the formula
A script with one static function: `leveled_stat(base: float, scale: float, level: int, cap: float) -> float`.
Logic: `base + base × scale × (level − 1)`, then clamp to `[0, cap]`.
Look up: `static func`, `clampf`.

### 1.5 Remaining value types (each a `Resource`, exported fields)
- `GearCombatStats`: `atk, armor, magic_atk, magic_armor` (float).
- `CharacterRuntimeState`: `character_id: StringName`, `current_hp: int`, `current_mp: int`, `level: int = 1`, `current_xp: int`, `earned_bonus_stats: CoreStats`, `unlocked_skill_node_ids: Array[StringName]`, `equipped_ability_ids: Array[StringName]`, `skill_points_available: int`. **Trap:** `earned_bonus_stats` is `null` unless you create it in `_init()`.
- `BondProgress`: `character_id`, `bond_level: int`, `consumed_bond_event_ids: Array[StringName]`.
- `PartyGroupState`: `group_id: StringName`, `active_member_ids: Array[StringName]`, `bench_member_ids: Array[StringName]`.
- `EquippedGearEntry`: `character_id`, plus `weapon, head, chest, gloves, accessory` (item ids as `StringName`).

Done when: you can create each as a `.tres` in the Inspector and see its fields.

---

## Phase 2 — Data definitions (`scripts/data/`, instances in `data/…`)

Each extends `GameData` and has a `class_name`. Fields (all exported):

| Class | Fields |
|---|---|
| `AbilityDefinition` | `display_name: String`, `action_speed`, `cost_type`, `cost_amount: int`, `effect_type`, `effect_magnitude: float`, `target_type`, `animation_tag: StringName`, `icon: Texture2D`, `status_effect_to_apply: StatusEffectDefinition` |
| `StatusEffectDefinition` | `display_name`, `duration: int` (turns), `stat_modifiers: CoreStats`, `vulnerable_to_finisher: bool`, `prevents_action: bool` |
| `CharacterDefinition` | `display_name`, `class_name_text: String` (display only), `base_stats: CoreStats`, `growth_scale: CoreStats`, `sprite_frames: SpriteFrames`, `starting_ability_ids: Array[StringName]` |
| `EnemyMoveEntry` | `ability: AbilityDefinition`, `weight: float`, `min_hp_percent: float`, `max_hp_percent: float` |
| `EnemyDefinition` | `display_name`, `base_stats`, `growth_scale` (**always all zero** — enemies never scale), `sprite_frames`, `move_pattern: Array[EnemyMoveEntry]`, `description: String` |
| `EncounterEntry` | `enemy: EnemyDefinition`, `spawn_slot_index: int`, `level: int` |
| `EncounterFormation` | `entries: Array[EncounterEntry]` |
| `ClassGrowthProfile` | `growth_scale: CoreStats` (Phase 10) |
| `MulticlassComboDefinition` | `combo_key: String`, `tier3_ability_ids: Array[StringName]` (Phase 10) |
| `SkillWheelNode` | `node_id`, `connected_node_ids`, `position: Vector2`, `ap_cost`, `granted_ability_id`, `passive_bonus: CoreStats` (Phase 11) |
| `EquipmentItem` / `WeaponItem` / `ArmorItem` | `slot`, `display_name`, `icon`, `bonus_ability_id`; weapon adds `gear_stats`, `direct_stat_modifiers`; armor adds `gear_stats`, `hidden_growth_contribution` |
| `GuestDefinition`, `BondEventDefinition` | per `02-ImplementationSteps.md` (Phases 12–13) — can wait |

Make these instances now (you need them for Phase 6): `basic_attack` (Standard, PhysicalDamage, magnitude 45, SingleEnemy, no cost), one test character, one test enemy with all-zero growth. **Set `id` on each.**

### `registry.gd` — autoload `Registry`
Job: find every definition and let others look them up by id.
Logic:
1. On `_ready`, walk `res://data` recursively.
2. For every `.tres`, load it; if it is a `GameData` with a non-empty id, store it in a dictionary keyed by id.
3. `get_data(id: StringName) -> GameData`: return the entry or null.
Look up: `DirAccess.get_directories_at/get_files_at`, `load`, `is` operator. Note exported builds rename files to `.tres.remap` — strip that suffix.
Register it in Project Settings → Globals → Autoload.

Done when: printing `Registry.get_data(&"basic_attack")` in a test script returns your resource.

---

## Phase 3 — Autoload managers (`scripts/autoload/`)

Autoload scripts have **no `class_name`** (it would clash with the autoload name). Register each under Globals → Autoload.

### `party_manager.gd` → `PartyManager`
Variables: `roster_state: Array[CharacterRuntimeState]`, `party_groups: Array[PartyGroupState]`, `current_group_id: StringName` (default `&"default"`), `lead_character_id`, `main_character_id`, `main_character_multiclass_picks: Array[Enums.MulticlassOption]` (max 3), `currency: int`, `inventory: Array[StringName]`, `equipped_gear: Array[EquippedGearEntry]`.

Write now:
- `get_character_state(character_id) -> CharacterRuntimeState` — loop the roster, return the one whose id matches, else null.
- `get_current_party_group() -> PartyGroupState` — loop the groups, return the one whose `group_id` equals `current_group_id`.

Stub (empty body, fill later): `apply_combat_result`, `spend_skill_point`, `add_multiclass_pick`, `level_up_character`, `respec_character`, `unlock_guest_ability`, `set_lead_character`, `switch_to_party_group`, `create_party_group`, `merge_party_groups`, `set_character_away`, `set_character_available`.

For testing: in `_ready`, build one `CharacterRuntimeState` for your test character (hp/mp from its base stats) and one `PartyGroupState` containing that id as active.

### `bond_manager.gd` → `BondManager`
`bond_progress: Array[BondProgress]` and an empty `trigger_bond_event(character_id, event_id)`.

### `encounter_manager.gd` → `EncounterManager`
Variables: `pending_formation: EncounterFormation`, `return_scene_path: String`, `return_spawn_point_id: StringName`.
Functions: `set_pending_encounter(formation, return_scene, return_spawn)` stores all three; `clear_pending_encounter()` nulls the formation.

---

## Phase 4 — The "interface" contract (no interfaces in Godot)

Decide and write down (a comment block in `combat_unit.gd` is fine) that anything in a fight must offer: `get_speed() -> float`, `take_turn()`, `apply_effect(ability, instigator)`, `is_alive() -> bool`, `is_enemy_side: bool`. Anything that needs to be attackable can be checked with `has_method`.

Make one small script, `action_choice.gd` (`RefCounted`, `class_name ActionChoice`) with two plain fields: `ability: AbilityDefinition`, `target: CombatUnit`. It replaces the "two output values" that `SelectNextAction` returned.

---

## Phase 5 — Components and unit scenes (`scripts/combat/`)

All components are `Node` scripts with a `class_name`, added as child nodes of a unit.

### `stats_component.gd`
Variables: `base_stats`, `growth_scale` (CoreStats), `level: int = 1`, `earned_bonus: CoreStats` (new instance), `max_hp_cap: float = 9999`, `max_mp_cap: float = 999`, `status: StatusEffectComponent`.
Function `get_effective_stats() -> CoreStats` — recomputes **every call**, never caches:
1. Create a new CoreStats.
2. For each of the 5 core stats: `StatsMath.leveled_stat(base, growth, level, 255)`.
3. For max_hp / max_mp: same, but the cap is `max_hp_cap` / `max_mp_cap`.
4. Add `earned_bonus` (`plus`).
5. If `status` exists, add `status.get_active_modifiers()`.
6. **Last**, floor the five core stats at 1 (`maxf`); HP/MP may be 0. Doing it after the modifiers is deliberate — buffs/debuffs can never push Speed to 0.
Also stub `get_gear_combat_stats() -> GearCombatStats` (return an empty one; design not settled).

### `resource_component.gd`
Signals: `hp_changed(current: int)`, `mp_changed(current: int)`. Variables: `current_hp: int`, `current_mp: int`, `special_gauge: float`.
Make the HP/MP setters clamp at 0 and emit their signal. Function `take_damage(amount: int)`: subtract from HP.
Look up: property setters (`var x: int: set(v): …`).

### `status_effect_component.gd`
Needs a small inner record for one active effect: `definition`, `turns_left: int`, `skip_next_tick: bool`.
- `apply_status(def, source) -> bool` — null def returns false; otherwise add a record with `turns_left = def.duration`, and `skip_next_tick = true` if the **source is this component's own unit** (a self-cast skips the tick on the turn it was cast).
- `tick_durations()` — called at the end of the unit's own turn. Loop **backwards** (you remove during the loop): if `skip_next_tick`, clear it and move on; otherwise decrement `turns_left`, and remove the record at 0.
- `get_active_modifiers() -> CoreStats` — sum of every active effect's `stat_modifiers` (use `plus`).

### `ability_loadout_component.gd`
Variable `equipped_abilities: Array[AbilityDefinition]`.
`execute_ability(ability, target) -> bool` — null-check both; call `target.apply_effect(ability, <the unit this component belongs to>)`; return true. (Owning unit = `get_parent()`.)

### Placeholders
`multiclass_component.gd` and `guest_assist_component.gd`: empty `Node` scripts for now.

### `enemy_brain.gd`
Function `select_next_action(me: CombatUnit) -> ActionChoice`. For now: ability = the `basic_attack` definition from the Registry; target = a random entry from `me.turn_manager.get_opposing_units(me)` (guard the empty case).
Later version (don't build yet): filter `move_pattern` by the unit's current HP percentage, weighted-random pick among the survivors, load that entry's ability. Boss brains will subclass this and override the function.

### `combat_unit.gd` (extends `Node3D`, `class_name CombatUnit`)
Exports/variables: `is_enemy_side: bool`, `turn_meter: float`, `turn_manager: TurnManager` (assigned by the manager).
`@onready` references to the four component child nodes.
Functions:
- `_ready()` — give `stats` its `status` reference.
- `is_alive() -> bool` — `resources.current_hp > 0`.
- `get_speed() -> float` — effective stats' speed.
- `take_turn()` — get an `ActionChoice` from `_choose_action()`, then tell the turn manager: `resolve_turn(self, choice.ability, choice.target)`.
- `_choose_action() -> ActionChoice` — base returns an empty choice; subclasses override.
- `apply_effect(ability, instigator)` — `match` on `ability.effect_type`:
  - PhysicalDamage: `damage = effect_magnitude + attacker strength − defender vitality`, minimum 1 → `resources.take_damage`, and `print` the damage and remaining HP (this is your tuning output).
  - ApplyStatusEffect: `status_effects.apply_status(ability.status_effect_to_apply, instigator)`.
  - Other types: leave empty for now.

### `player_unit.gd` / `enemy_unit.gd` (extend `CombatUnit`)
**Player**: variables `character_id`, `character_def`. In `_ready` (call the parent's first): `is_enemy_side = false`; look up the definition; fetch the runtime state from `PartyManager`; copy base/growth stats, `level`, `earned_bonus_stats` into `stats`; set `current_hp/mp` from the state (HP carries over between fights); fill the loadout from `equipped_ability_ids` via the Registry. `_choose_action` (first playtest only): auto Basic Attack on a random opposing unit — Phase 7's menu replaces this.
**Enemy**: variables `enemy_id`, `level`, `enemy_def`, `@onready brain`. In `_ready`: `is_enemy_side = true`; look up the definition; copy base/growth; `stats.level = level`; set **`max_hp_cap` / `max_mp_cap` to the enemy's own base max_hp / max_mp** (so a big boss isn't clamped to the player 9999/999); compute effective stats; set current HP/MP to their max (enemies always start full). `_choose_action` returns `brain.select_next_action(self)`.
**Key rule:** both `character_id` / `enemy_id` are set by whoever spawns the unit **before** `add_child()` — that is how the ids are already there when `_ready` runs.

### Scenes
- `combat_unit.tscn`: root `Node3D` with the `combat_unit` script; children `StatsComponent`, `ResourceComponent`, `AbilityLoadoutComponent`, `StatusEffectComponent`, and an `AnimatedSprite3D` (billboard on).
- `player_unit.tscn`: **New Inherited Scene** of the above; change the root script; add the Multiclass/Guest placeholder nodes.
- `enemy_unit.tscn`: inherited; change the root script; add an `EnemyBrain` child.
Node names must match the `@onready` paths exactly.

Done when: instancing a player and an enemy scene in a throwaway test scene prints sane `get_effective_stats()` values (check the Level 1 numbers against your spreadsheet).

---

## Phase 6 — Turn manager and battle scene

### `turn_manager.gd` (`Node`, `class_name TurnManager`)
Signals: `turn_started(unit)`, `action_resolved(acting, target)`, `battle_ended(players_won: bool)`.
Exports: `action_threshold: float = 100`, `start_jitter: float = 15`, `revive_start_meter: float = 0` (unused until revive exists).
Variable: `active_units: Array[CombatUnit]`.

**`initialize_combat(units)`**
1. Store the units in `active_units` (look up: `Array.assign` for typed arrays).
2. For each: give it a reference to this manager, and set `turn_meter` to a random value in `[0, start_jitter]`. This is the **only** random roll in the whole system — do not re-roll later.
3. Call `take_next_turn()`.

**`take_next_turn()`** — the core. For every living unit compute the time until it reaches the threshold: `max((action_threshold − turn_meter) / speed, 0)`.
1. Track the best (smallest time) unit. Tie-breaks: if times are (nearly) equal, the **higher speed** wins; if still tied, the one earlier in the array wins (so the spawner adds players before enemies). Look up: `is_equal_approx`, `INF`.
2. If nobody is alive, return.
3. Advance **every living unit's** `turn_meter` by `speed × best_time`.
4. Set the chosen unit's `turn_meter` to exactly `action_threshold` (stops floating-point drift).
5. Emit `turn_started`, then call the unit's `take_turn()`.
Speed is read fresh each time, so buffs apply instantly. Speed can't be 0 (the stat floor), so the division is safe.

**`resolve_turn(acting, ability, target)`**
1. `acting.ability_loadout.execute_ability(ability, target)`.
2. Emit `action_resolved`.
3. `acting.turn_meter -= action_threshold` (leftover carries over).
4. `acting.status_effects.tick_durations()`.
5. If `get_opposing_units(acting)` is empty, the battle is over: emit `battle_ended(not acting.is_enemy_side)` and return.
6. Otherwise schedule `take_next_turn` with **`call_deferred`**, not a direct call. A direct call recurses once per turn and a long fight overflows GDScript's call stack.

**`get_opposing_units(requester) -> Array[CombatUnit]`** — living units in `active_units` whose `is_enemy_side` differs from the requester's.
**`resolve_help_action(acting, threat)`** — empty stub.

Not in this pass: action categories (Quick/Standard/Long) and Haste/Slow. Every action costs `action_threshold`. Later, `resolve_turn` subtracts a category-dependent amount instead.

### `battle_manager.gd` — root script of `battle.tscn` (replaces GameMode + GameState)
Scene: a `Node3D` root with a `TurnManager` child, ordered `Marker3D` spawn points in groups `player_spawn` and `enemy_spawn`, a `Camera3D`, a light and a floor.
Variables: `phase: Enums.BattlePhase`; signal `phase_changed(phase)`; `preload`ed player and enemy scenes; an exported fallback enemy id for testing.
`_ready` logic:
1. Get the current party group from `PartyManager`.
2. For each active member: instantiate the player scene, set its `character_id`, set its position from the matching spawn marker, **then** `add_child` (this is the order that makes the id visible to `_ready`), append to a `units` list. **Players go in the list first** (tie-break rule).
3. For each entry in the pending encounter formation: instantiate an enemy, set `enemy_id` and `level`, position from `spawn_slot_index`, `add_child`, append.
4. If there is no pending formation, spawn one fallback test enemy instead.
5. Connect `battle_ended` to a handler that sets the phase to Victory/Defeat and prints the result.
6. Set phase InProgress, then `turn_manager.initialize_combat(units)` (which starts the first turn itself).
Look up: `PackedScene.instantiate`, `as` casts, `get_tree().get_nodes_in_group`.

### Phase 6 is done when
Running `battle.tscn` (F6) prints each attack and remaining HP, units act in Speed order with sensible ratios (a unit with double the Speed acts about twice as often), and the fight ends with a Victory or Defeat line. Tune `start_jitter`, `effect_magnitude` and stats from that output.

---

## Suggested order and checkpoints
0 → 1 → 2 (+ `Registry`, three `.tres`) → 3 → 5 components (check `get_effective_stats` against your spreadsheet) → unit scenes → 6. Commit at the end of each phase. For the Unreal-vs-Godot comparison, keep notes on: time to build each phase, bugs hit, and how fast you can test a change.
