# KingsCoup — Godot (GDScript) Build Plan, Phases 0–6

Alternative to `02-ImplementationSteps.md`: the same design (see `01-ClassStructure.md` and the memory file for the settled decisions), written for **Godot 4.7 + GDScript** instead of Unreal Blueprints. Nothing here has been built — this is a plan for "what would I have to code".

The goal is unchanged: get one player unit fighting one enemy through full turns (attack → damage → win/lose) before building anything else. Phases 0–6 here end at a **battle that runs in the console/debug output**; Phase 7 (UI) is not covered.

---

## 0. How the Unreal concepts map to Godot

| Unreal (current plan) | Godot equivalent |
|---|---|
| Struct (`F_CoreStats`) | `Resource` subclass (editable in the Inspector, savable, copyable with `duplicate()`) |
| Enum asset (`E_ActionSpeed`) | `enum` inside a `class_name Enums` script |
| Primary Data Asset | `Resource` subclass saved as a `.tres` file |
| Hard reference to an asset | `@export var ability: AbilityDefinition` (drag the `.tres` in) |
| Soft reference / ID (`CharacterID`) | a `StringName` id, looked up through a `Registry` autoload |
| GameInstance + manager Objects | **Autoloads** (global singleton nodes — trivial in Godot, no subsystem wall) |
| Actor Component | a child `Node` with a script (`StatsComponent`, …) |
| Actor (`BP_CombatUnitBase`) | `Node3D` scene (2.5D) with a script, plus a sprite child |
| Blueprint Interface | none — use a base class (`CombatUnit`) and duck typing (`has_method`) |
| Event Dispatcher | `signal` |
| Expose on Spawn | instantiate the scene, set properties, **then** `add_child()` (so `_ready()` already sees them) |
| GameMode / GameState | a script on the battle scene's root node (`BattleManager`) |
| Level | a `.tscn` scene |
| Cast To | `as` (`var u := node as CombatUnit`) |

Three Godot rules that matter below:
1. **Resources are shared by reference.** Two units using the same `.tres` share one object. Never mutate a definition at runtime; `duplicate(true)` first.
2. **Don't name a variable `str` or `int`** (they are built-in functions/types). Stats are named `strength`, `vitality`, `intelligence`, `spirit`, `speed`.
3. **An autoload cannot share a name with a `class_name`.** Autoload scripts below have no `class_name`.

---

## Phase 0 — Project & folder setup

1. Create the project (Godot 4.7). Renderer: Forward+ or Mobile (3D lighting for the 2.5D look). Project Settings → Rendering → Textures → Default Texture Filter = **Nearest** (crisp pixel art).
2. Folders:

```
res://
  data/            abilities/ status_effects/ characters/ enemies/ items/
                   skill_wheel/ multiclass/ guests/ bonds/ encounters/
  scripts/
    core/          enums.gd game_data.gd core_stats.gd stats_math.gd (+ state resources)
    data/          one script per definition resource
    autoload/      registry.gd party_manager.gd bond_manager.gd encounter_manager.gd
    combat/        components + unit scripts + turn_manager.gd + battle_manager.gd
  scenes/
    combat/        combat_unit.tscn player_unit.tscn enemy_unit.tscn
    battle/        battle.tscn
    exploration/   (later)
  ui/              (later)
```

3. `.gitignore`: `.godot/`, `*.translation`, `export_presets.cfg` (if it holds secrets), `.mono/`. Scenes and `.tres` files are text, so git diffs are readable.

---

## Phase 1 — Enums & value types

All in `scripts/core/`.

### 1.1 `enums.gd`
```gdscript
class_name Enums
extends RefCounted

enum ActionSpeed { QUICK, STANDARD, LONG }
enum AbilityCostType { NONE, MP, SPECIAL_GAUGE }
enum TargetType { SINGLE_ENEMY, ALL_ENEMIES, SINGLE_ALLY, ALL_ALLIES, SELF }
enum EffectType { PHYSICAL_DAMAGE, MAGIC_DAMAGE, HEAL, RESTORE_MP, APPLY_STATUS_EFFECT }
enum MulticlassOption { FIGHTER, RANGER, MAGE }
enum EquipmentSlot { WEAPON, HEAD, CHEST, GLOVES, ACCESSORY }
enum BondEventType { DIALOGUE, FETCH_ITEM, QUEST }   # fill in later
enum BattlePhase { INTRO, IN_PROGRESS, VICTORY, DEFEAT }
```
Used as `Enums.ActionSpeed.STANDARD`.

### 1.2 `game_data.gd` — base class for everything in `data/`
```gdscript
class_name GameData
extends Resource

@export var id: StringName = &""   # unique, e.g. &"basic_attack"; the Registry indexes on this
```

### 1.3 `core_stats.gd` — replaces `F_CoreStats` (used for values *and* deltas)
```gdscript
class_name CoreStats
extends Resource

@export var strength: float = 0.0
@export var vitality: float = 0.0
@export var intelligence: float = 0.0
@export var spirit: float = 0.0
@export var speed: float = 0.0
@export var max_hp: float = 0.0
@export var max_mp: float = 0.0

func plus(o: CoreStats) -> CoreStats:
    var r := CoreStats.new()
    r.strength = strength + o.strength
    r.vitality = vitality + o.vitality
    r.intelligence = intelligence + o.intelligence
    r.spirit = spirit + o.spirit
    r.speed = speed + o.speed
    r.max_hp = max_hp + o.max_hp
    r.max_mp = max_mp + o.max_mp
    return r
```

### 1.4 `stats_math.gd` — the confirmed spreadsheet formula
```gdscript
class_name StatsMath
extends RefCounted

# Stat(Level) = Base + (Base × Scale) × (Level − 1), clamped to [0, cap]
static func leveled_stat(base: float, scale: float, level: int, cap: float) -> float:
    return clampf(base + base * scale * (level - 1), 0.0, cap)
```

### 1.5 Remaining value types (all `extends Resource`, all `@export` fields)
| Class (file) | Fields |
|---|---|
| `GearCombatStats` | `atk`, `armor`, `magic_atk`, `magic_armor` (float) |
| `CharacterRuntimeState` | `character_id: StringName`, `current_hp: int`, `current_mp: int`, `level: int = 1` (cap 99), `current_xp: int`, `earned_bonus_stats: CoreStats`, `unlocked_skill_node_ids: Array[StringName]`, `equipped_ability_ids: Array[StringName]`, `skill_points_available: int` |
| `BondProgress` | `character_id`, `bond_level: int`, `consumed_bond_event_ids: Array[StringName]` |
| `PartyGroupState` | `group_id: StringName`, `active_member_ids: Array[StringName]`, `bench_member_ids: Array[StringName]` |
| `EquippedGearEntry` | `character_id`, `weapon`, `head`, `chest`, `gloves`, `accessory` (all `StringName` item ids) |

`CharacterRuntimeState.earned_bonus_stats` must default to `CoreStats.new()` (set in `_init()`), or it will be `null`.

IDs (`StringName`) replace the "soft object reference" fields from the Unreal plan: they are cheap to save and never create reference cycles.

---

## Phase 2 — Data definitions

All in `scripts/data/`, each `extends GameData` with a `class_name`. Create instances as `.tres` files in `res://data/<folder>/` via the FileSystem dock → New Resource. **Set `id` on every instance.**

| Class | Folder | Fields |
|---|---|---|
| `AbilityDefinition` | `abilities` | `display_name: String`, `action_speed: Enums.ActionSpeed = STANDARD`, `cost_type: Enums.AbilityCostType`, `cost_amount: int`, `effect_type: Enums.EffectType`, `effect_magnitude: float`, `target_type: Enums.TargetType`, `animation_tag: StringName`, `icon: Texture2D`, `status_effect_to_apply: StatusEffectDefinition` |
| `StatusEffectDefinition` | `status_effects` | `display_name`, `duration: int` (turns), `stat_modifiers: CoreStats`, `b_vulnerable_to_finisher: bool`, `b_prevents_action: bool` |
| `CharacterDefinition` | `characters` | `display_name`, `class_name_text: String` (display only), `base_stats: CoreStats`, `growth_scale: CoreStats`, `sprite_frames: SpriteFrames`, `starting_ability_ids: Array[StringName]` |
| `EnemyMoveEntry` | (embedded) | `ability: AbilityDefinition`, `weight: float`, `min_hp_percent: float`, `max_hp_percent: float` |
| `EnemyDefinition` | `enemies` | `display_name`, `base_stats: CoreStats`, `growth_scale: CoreStats` (**always zero** — enemies never scale; a "Lv10 Goblin" is its own `.tres`), `sprite_frames: SpriteFrames`, `move_pattern: Array[EnemyMoveEntry]`, `description: String` |
| `ClassGrowthProfile` | `multiclass` | `growth_scale: CoreStats` (Fighter / Ranger / Mage — Phase 10) |
| `MulticlassComboDefinition` | `multiclass` | `combo_key: String` (e.g. `"FFM"`), `tier3_ability_ids: Array[StringName]` |
| `SkillWheelNode` | `skill_wheel` | `node_id`, `connected_node_ids: Array[StringName]`, `position: Vector2`, `ap_cost: int`, `granted_ability_id: StringName`, `passive_bonus: CoreStats` |
| `EquipmentItem` | `items` | `slot: Enums.EquipmentSlot`, `display_name`, `icon`, `bonus_ability_id: StringName` |
| `WeaponItem` (extends `EquipmentItem`) | `items` | `gear_stats: GearCombatStats`, `direct_stat_modifiers: CoreStats` |
| `ArmorItem` (extends `EquipmentItem`) | `items` | `gear_stats: GearCombatStats`, `hidden_growth_contribution: CoreStats` |
| `GuestDefinition` | `guests` | `display_name`, `passive_ability_id`, `active_ability_id`, `bond_level_required: int`, `portrait: Texture2D` |
| `EncounterEntry` | (embedded) | `enemy: EnemyDefinition`, `spawn_slot_index: int`, `level: int` |
| `EncounterFormation` | `encounters` | `entries: Array[EncounterEntry]` |
| `BondEventDefinition` | `bonds` | `character_id`, scenes/quests list, requirements, reward |

Make at least two instances now: `data/abilities/basic_attack.tres` (id `basic_attack`, STANDARD, PHYSICAL_DAMAGE, magnitude 45, SINGLE_ENEMY, cost NONE) and one test character + one test enemy.

### 2.1 `registry.gd` — autoload `Registry` (id → resource lookup)
```gdscript
extends Node

var _by_id: Dictionary = {}   # StringName -> GameData

func _ready() -> void:
    _scan("res://data")

func _scan(path: String) -> void:
    for dir in DirAccess.get_directories_at(path):
        _scan(path.path_join(dir))
    for f in DirAccess.get_files_at(path):
        var file := f.trim_suffix(".remap")     # exported builds rename .tres → .tres.remap
        if file.ends_with(".tres"):
            var res := load(path.path_join(file))
            if res is GameData and res.id != &"":
                _by_id[res.id] = res

func get_data(id: StringName) -> GameData:
    return _by_id.get(id) as GameData
```
Register it in Project Settings → Globals → Autoload as `Registry`.

---

## Phase 3 — Autoload managers (replaces GameInstance + Object managers)

Each is a script with **no `class_name`**, registered under Project Settings → Globals → Autoload. Anywhere the Unreal plan says "Get Game Instance → cast → PartyManager", you just write `PartyManager.some_function()`.

### 3.1 `party_manager.gd` → autoload `PartyManager`
Variables:
```gdscript
extends Node

var roster_state: Array[CharacterRuntimeState] = []   # every unlocked character
var party_groups: Array[PartyGroupState] = []         # normally one: &"default"
var current_group_id: StringName = &"default"
var lead_character_id: StringName = &""               # who the exploration pawn represents
var main_character_id: StringName = &""               # fixed; only used by multiclass
var main_character_multiclass_picks: Array[Enums.MulticlassOption] = []   # max 3
var currency: int = 0
var inventory: Array[StringName] = []                 # item ids
var equipped_gear: Array[EquippedGearEntry] = []
```
Functions to write (signature → purpose):
```gdscript
func get_character_state(character_id: StringName) -> CharacterRuntimeState:
    for s in roster_state:
        if s.character_id == character_id:
            return s
    return null

func get_current_party_group() -> PartyGroupState:
    for g in party_groups:
        if g.group_id == current_group_id:
            return g
    return null
```
Stubs now (bodies come in later phases): `apply_combat_result(...)`, `spend_skill_point(character_id, node_id)`, `add_multiclass_pick(choice)`, `level_up_character(character_id)` (Phase 9), `respec_character(character_id)` (Phase 11), `unlock_guest_ability(guest_id)` (Phase 12), `set_lead_character(character_id)`, `switch_to_party_group(group_id)`, `create_party_group(group_id, active, bench)`, `merge_party_groups(source_id, target_id)`, `set_character_away(character_id)`, `set_character_available(character_id, group_id)`.

**For testing Phase 6**, add a `_ready()` that builds one `CharacterRuntimeState` for your test character (hp/mp from its stats) and one `PartyGroupState` with that id in `active_member_ids`.

### 3.2 `bond_manager.gd` → autoload `BondManager`
`var bond_progress: Array[BondProgress] = []` and `func trigger_bond_event(character_id: StringName, event_id: StringName) -> void: pass`.

### 3.3 `encounter_manager.gd` → autoload `EncounterManager`
```gdscript
extends Node

var pending_formation: EncounterFormation
var return_scene_path: String = ""
var return_spawn_point_id: StringName = &""

func set_pending_encounter(formation: EncounterFormation, return_scene: String, return_spawn: StringName) -> void:
    pending_formation = formation
    return_scene_path = return_scene
    return_spawn_point_id = return_spawn

func clear_pending_encounter() -> void:
    pending_formation = null
```

---

## Phase 4 — "Interfaces" (there are none; this is the contract instead)

Godot has no interfaces. The three Blueprint interfaces become **methods on `CombatUnit`** (Phase 5) plus two tiny classes:

| Unreal | Godot |
|---|---|
| `BPI_ActionSource.GetSpeed()` / `TakeTurn()` | `CombatUnit.get_speed()` / `take_turn()` |
| `BPI_Targetable.ApplyEffect()` / `IsEnemyUnit()` / `GetTargetType()` | `CombatUnit.apply_effect()` / `is_enemy_side` / read the ability's `target_type` |
| `BPI_EnemyBrain.SelectNextAction()` | `EnemyBrain.select_next_action(me)` — a base class bosses can subclass |

```gdscript
# scripts/combat/action_choice.gd — what SelectNextAction returned as two outputs
class_name ActionChoice
extends RefCounted

var ability: AbilityDefinition
var target: CombatUnit
```
Anything that only needs "something I can attack" can check `target.has_method("apply_effect")`.

---

## Phase 5 — Combat components & unit scenes

All in `scripts/combat/`. Components are plain `Node` children.

### 5.1 `stats_component.gd`
```gdscript
class_name StatsComponent
extends Node

var base_stats: CoreStats
var growth_scale: CoreStats
var level: int = 1
var earned_bonus: CoreStats = CoreStats.new()
var max_hp_cap: float = 9999.0      # players; enemies override to their own base value
var max_mp_cap: float = 999.0
var status: StatusEffectComponent   # set by the unit; supplies live buff/debuff modifiers

# Recomputed fresh every call — never cached (keeps growth/buff changes automatic).
func get_effective_stats() -> CoreStats:
    var s := CoreStats.new()
    s.strength     = StatsMath.leveled_stat(base_stats.strength,     growth_scale.strength,     level, 255.0)
    s.vitality     = StatsMath.leveled_stat(base_stats.vitality,     growth_scale.vitality,     level, 255.0)
    s.intelligence = StatsMath.leveled_stat(base_stats.intelligence, growth_scale.intelligence, level, 255.0)
    s.spirit       = StatsMath.leveled_stat(base_stats.spirit,       growth_scale.spirit,       level, 255.0)
    s.speed        = StatsMath.leveled_stat(base_stats.speed,        growth_scale.speed,        level, 255.0)
    s.max_hp       = StatsMath.leveled_stat(base_stats.max_hp,       growth_scale.max_hp,       level, max_hp_cap)
    s.max_mp       = StatsMath.leveled_stat(base_stats.max_mp,       growth_scale.max_mp,       level, max_mp_cap)
    s = s.plus(earned_bonus)
    if status != null:
        s = s.plus(status.get_active_modifiers())
    # Floor: everything except HP/MP is at least 1 (applied last, after modifiers).
    s.strength = maxf(s.strength, 1.0)
    s.vitality = maxf(s.vitality, 1.0)
    s.intelligence = maxf(s.intelligence, 1.0)
    s.spirit = maxf(s.spirit, 1.0)
    s.speed = maxf(s.speed, 1.0)
    return s

func get_gear_combat_stats() -> GearCombatStats:
    return GearCombatStats.new()     # deferred — design not settled
```

### 5.2 `resource_component.gd`
```gdscript
class_name ResourceComponent
extends Node

signal hp_changed(current: int)
signal mp_changed(current: int)

var special_gauge: float = 0.0
var current_hp: int = 0:
    set(v):
        current_hp = maxi(v, 0)
        hp_changed.emit(current_hp)
var current_mp: int = 0:
    set(v):
        current_mp = maxi(v, 0)
        mp_changed.emit(current_mp)

func take_damage(amount: int) -> void:
    current_hp -= amount
```

### 5.3 `status_effect_component.gd`
```gdscript
class_name StatusEffectComponent
extends Node

class ActiveStatus extends RefCounted:
    var definition: StatusEffectDefinition
    var turns_left: int = 0
    var skip_next_tick: bool = false    # set when cast on yourself this turn

var active_effects: Array[ActiveStatus] = []

func apply_status(def: StatusEffectDefinition, source: CombatUnit) -> bool:
    if def == null:
        return false
    var a := ActiveStatus.new()
    a.definition = def
    a.turns_left = def.duration
    a.skip_next_tick = (source == get_parent())     # self-cast skips the tick this turn
    active_effects.append(a)
    return true

# Called at the END of the owner's own turn (ResolveTurn).
func tick_durations() -> void:
    for i in range(active_effects.size() - 1, -1, -1):
        var a := active_effects[i]
        if a.skip_next_tick:
            a.skip_next_tick = false
            continue
        a.turns_left -= 1
        if a.turns_left <= 0:
            active_effects.remove_at(i)

func get_active_modifiers() -> CoreStats:
    var total := CoreStats.new()
    for a in active_effects:
        if a.definition.stat_modifiers != null:
            total = total.plus(a.definition.stat_modifiers)
    return total
```

### 5.4 `ability_loadout_component.gd`
```gdscript
class_name AbilityLoadoutComponent
extends Node

var equipped_abilities: Array[AbilityDefinition] = []

func execute_ability(ability: AbilityDefinition, target: CombatUnit) -> bool:
    if ability == null or target == null:
        return false
    target.apply_effect(ability, get_parent() as CombatUnit)
    return true
```

### 5.5 Empty placeholders
`multiclass_component.gd` (`class_name MulticlassComponent extends Node`) and `guest_assist_component.gd` — leave empty (Phase 10 / 12).

### 5.6 `enemy_brain.gd` (placeholder: everyone uses Basic Attack)
```gdscript
class_name EnemyBrain
extends Node

func select_next_action(me: CombatUnit) -> ActionChoice:
    var choice := ActionChoice.new()
    choice.ability = Registry.get_data(&"basic_attack") as AbilityDefinition
    var targets := me.turn_manager.get_opposing_units(me)
    if not targets.is_empty():
        choice.target = targets.pick_random()
    return choice
```
**Later (real version)**, same signature: read `(me as EnemyUnit).enemy_def.move_pattern`; compute HP% = `current_hp / effective.max_hp`; keep entries whose `min_hp_percent ≤ hp% ≤ max_hp_percent`; weighted-random pick among them (sum weights, `randf() * total`, walk the list); target still comes from `get_opposing_units`. Bosses subclass `EnemyBrain` and override `select_next_action`.

### 5.7 `combat_unit.gd` — the base (replaces `BP_CombatUnitBase`)
```gdscript
class_name CombatUnit
extends Node3D

@export var is_enemy_side: bool = false

var turn_meter: float = 0.0
var turn_manager: TurnManager       # set by TurnManager before take_turn()

@onready var stats: StatsComponent = $StatsComponent
@onready var resources: ResourceComponent = $ResourceComponent
@onready var ability_loadout: AbilityLoadoutComponent = $AbilityLoadoutComponent
@onready var status_effects: StatusEffectComponent = $StatusEffectComponent

func _ready() -> void:
    stats.status = status_effects

func is_alive() -> bool:
    return resources.current_hp > 0

func get_speed() -> float:
    return stats.get_effective_stats().speed

func take_turn() -> void:
    var choice := _choose_action()
    turn_manager.resolve_turn(self, choice.ability, choice.target)

func _choose_action() -> ActionChoice:      # overridden by subclasses
    return ActionChoice.new()

func apply_effect(ability: AbilityDefinition, instigator: CombatUnit) -> void:
    match ability.effect_type:
        Enums.EffectType.PHYSICAL_DAMAGE:
            var atk := instigator.stats.get_effective_stats().strength
            var vit := stats.get_effective_stats().vitality
            var dmg := maxi(int(ability.effect_magnitude + atk - vit), 1)   # placeholder formula, min 1
            resources.take_damage(dmg)
            print("%s hits %s for %d (HP left %d)" % [instigator.name, name, dmg, resources.current_hp])
        Enums.EffectType.APPLY_STATUS_EFFECT:
            status_effects.apply_status(ability.status_effect_to_apply, instigator)
        _:
            pass     # MAGIC_DAMAGE, HEAL, RESTORE_MP — later
```

### 5.8 `player_unit.gd`
```gdscript
class_name PlayerUnit
extends CombatUnit

var character_id: StringName            # set BEFORE add_child() — that is "expose on spawn"
var character_def: CharacterDefinition

func _ready() -> void:
    super()
    is_enemy_side = false
    character_def = Registry.get_data(character_id) as CharacterDefinition
    var state := PartyManager.get_character_state(character_id)
    stats.base_stats = character_def.base_stats
    stats.growth_scale = character_def.growth_scale
    stats.level = state.level
    stats.earned_bonus = state.earned_bonus_stats
    resources.current_hp = state.current_hp      # carries over between fights
    resources.current_mp = state.current_mp
    for aid in state.equipped_ability_ids:
        ability_loadout.equipped_abilities.append(Registry.get_data(aid) as AbilityDefinition)

# First playtest: players auto-attack. Phase 7 replaces this with the action menu.
func _choose_action() -> ActionChoice:
    var c := ActionChoice.new()
    c.ability = Registry.get_data(&"basic_attack") as AbilityDefinition
    var targets := turn_manager.get_opposing_units(self)
    if not targets.is_empty():
        c.target = targets.pick_random()
    return c
```

### 5.9 `enemy_unit.gd`
```gdscript
class_name EnemyUnit
extends CombatUnit

var enemy_id: StringName                # set before add_child()
var level: int = 1
var enemy_def: EnemyDefinition

@onready var brain: EnemyBrain = $EnemyBrain

func _ready() -> void:
    super()
    is_enemy_side = true
    enemy_def = Registry.get_data(enemy_id) as EnemyDefinition
    stats.base_stats = enemy_def.base_stats
    stats.growth_scale = enemy_def.growth_scale      # all zero — no scaling
    stats.level = level
    # Enemies may exceed the player 9999/999 ceiling: cap at their own authored value.
    stats.max_hp_cap = enemy_def.base_stats.max_hp
    stats.max_mp_cap = enemy_def.base_stats.max_mp
    var eff := stats.get_effective_stats()
    resources.current_hp = int(eff.max_hp)           # enemies always spawn at full health
    resources.current_mp = int(eff.max_mp)

func _choose_action() -> ActionChoice:
    return brain.select_next_action(self)
```

### 5.10 Scenes
- `combat_unit.tscn`: root `Node3D` (script `combat_unit.gd`) → children `StatsComponent`, `ResourceComponent`, `AbilityLoadoutComponent`, `StatusEffectComponent` (each a `Node` with its script), plus a visual child (`AnimatedSprite3D`, billboard on).
- `player_unit.tscn`: **New Inherited Scene** from `combat_unit.tscn`, swap the root script to `player_unit.gd`, add `MulticlassComponent` + `GuestAssistComponent` child nodes.
- `enemy_unit.tscn`: inherited scene, root script `enemy_unit.gd`, add an `EnemyBrain` child.

---

## Phase 6 — Turn manager & battle scene

### 6.1 `turn_manager.gd` (meter-based, no timer)
```gdscript
class_name TurnManager
extends Node

signal turn_started(unit: CombatUnit)
signal action_resolved(acting: CombatUnit, target: CombatUnit)
signal battle_ended(players_won: bool)

@export var action_threshold: float = 100.0
@export var start_jitter: float = 15.0          # one-time initiative roll; tune by feel
@export var revive_start_meter: float = 0.0     # unused until revive exists

var active_units: Array[CombatUnit] = []

func initialize_combat(units: Array[CombatUnit]) -> void:
    active_units.assign(units)
    for u in active_units:
        u.turn_manager = self
        u.turn_meter = randf_range(0.0, start_jitter)    # the ONLY random roll, at battle start
    take_next_turn()

func take_next_turn() -> void:
    var best: CombatUnit = null
    var best_time: float = INF
    var best_speed: float = 0.0
    for u in active_units:
        if not u.is_alive():
            continue
        var spd := u.get_speed()                       # always >= 1 (stat floor), so no divide-by-zero
        var t := maxf((action_threshold - u.turn_meter) / spd, 0.0)
        # smallest time wins; tie → higher speed; still tied → earlier in the array (players first)
        if t < best_time or (is_equal_approx(t, best_time) and spd > best_speed):
            best = u
            best_time = t
            best_speed = spd
    if best == null:
        return                                         # nobody alive
    for u in active_units:                             # jump everyone forward to that moment
        if u.is_alive():
            u.turn_meter += u.get_speed() * best_time
    best.turn_meter = action_threshold                 # land exactly — avoids float drift
    turn_started.emit(best)
    best.take_turn()

func resolve_turn(acting: CombatUnit, ability: AbilityDefinition, target: CombatUnit) -> void:
    acting.ability_loadout.execute_ability(ability, target)
    action_resolved.emit(acting, target)
    acting.turn_meter -= action_threshold              # leftover carries over
    acting.status_effects.tick_durations()             # durations count the unit's OWN turns
    if get_opposing_units(acting).is_empty():
        battle_ended.emit(not acting.is_enemy_side)
        return
    take_next_turn.call_deferred()                     # NOT a direct call (see note)

func get_opposing_units(requester: CombatUnit) -> Array[CombatUnit]:
    var result: Array[CombatUnit] = []
    for u in active_units:
        if u.is_alive() and u.is_enemy_side != requester.is_enemy_side:
            result.append(u)
    return result

func resolve_help_action(acting: CombatUnit, target_threat: CombatUnit) -> void:
    pass                                               # Help action — later
```
**Why `call_deferred` on the last line:** `take_next_turn → take_turn → resolve_turn → take_next_turn` would otherwise recurse once per turn, and GDScript has a fixed call-stack limit — a long fight would overflow it. Deferring ends the current call chain first. (Unreal has the same issue — it caps nested Blueprint calls at roughly 120 — so the Blueprint version may need a short Timer/Delay at the same point if a long all-AI fight hits that limit.) Player turns in Phase 7 will `await` the menu choice and then call `resolve_turn`, which fits the same shape.

Action categories (Quick/Standard/Long) and Haste/Slow are **not** in this pass: every action costs `action_threshold`. Later, `resolve_turn` subtracts a category-dependent amount instead (see the table in `02-ImplementationSteps.md`).

### 6.2 `battle_manager.gd` — replaces `GM_Combat` **and** `GS_Combat`
Attach to the root of `battle.tscn` (a `Node3D`). Children: a `TurnManager` node, `Marker3D` nodes in groups `player_spawn` / `enemy_spawn` (ordered), a `Camera3D`, light(s), a floor.
```gdscript
extends Node3D

signal phase_changed(phase: Enums.BattlePhase)

const PLAYER_UNIT_SCENE := preload("res://scenes/combat/player_unit.tscn")
const ENEMY_UNIT_SCENE := preload("res://scenes/combat/enemy_unit.tscn")

@export var test_enemy_id: StringName = &"test_slime"     # fallback when no formation is pending

@onready var turn_manager: TurnManager = $TurnManager

var phase: Enums.BattlePhase = Enums.BattlePhase.INTRO

func _ready() -> void:
    var units: Array[CombatUnit] = []                       # players FIRST so ties favour them
    var player_spawns := get_tree().get_nodes_in_group("player_spawn")
    var enemy_spawns := get_tree().get_nodes_in_group("enemy_spawn")

    var party := PartyManager.get_current_party_group()
    for i in party.active_member_ids.size():
        var p := PLAYER_UNIT_SCENE.instantiate() as PlayerUnit
        p.character_id = party.active_member_ids[i]         # set BEFORE add_child
        p.position = (player_spawns[i] as Node3D).position
        add_child(p)                                        # _ready() runs here, id already set
        units.append(p)

    var formation := EncounterManager.pending_formation
    var entries: Array[EncounterEntry] = []
    if formation != null:
        entries = formation.entries
    for i in entries.size():
        var e := ENEMY_UNIT_SCENE.instantiate() as EnemyUnit
        e.enemy_id = entries[i].enemy.id
        e.level = entries[i].level
        e.position = (enemy_spawns[entries[i].spawn_slot_index] as Node3D).position
        add_child(e)
        units.append(e)
    if entries.is_empty():                                  # test fallback: one enemy
        var e := ENEMY_UNIT_SCENE.instantiate() as EnemyUnit
        e.enemy_id = test_enemy_id
        e.position = (enemy_spawns[0] as Node3D).position
        add_child(e)
        units.append(e)

    turn_manager.battle_ended.connect(_on_battle_ended)
    _set_phase(Enums.BattlePhase.IN_PROGRESS)
    turn_manager.initialize_combat(units)

func _on_battle_ended(players_won: bool) -> void:
    _set_phase(Enums.BattlePhase.VICTORY if players_won else Enums.BattlePhase.DEFEAT)
    print("Battle over — ", "VICTORY" if players_won else "DEFEAT")

func _set_phase(p: Enums.BattlePhase) -> void:
    phase = p
    phase_changed.emit(p)
```
(Group membership: add the `Marker3D` nodes to groups via the Node dock → Groups tab. Phase 9 will also write results back with `PartyManager.apply_combat_result(...)`.)

### 6.3 Phase 6 done when
Run `battle.tscn` (F6): the console prints each attack and remaining HP, units act in Speed order with correct ratios, and it ends with "VICTORY" or "DEFEAT". Tune `start_jitter`, `effect_magnitude` and stats by watching that output.

---

## Differences worth knowing vs. the Unreal plan

- **Less plumbing:** no casting, no reroute nodes, no interface plumbing, no "Construct Object" for managers, no Expose-on-Spawn flags, no `list_properties`-style editor quirks.
- **Data is text:** `.tres`/`.tscn` diff and merge in git; no 100+ MB binary assets.
- **You lose:** Unreal's lighting/post-processing quality for an HD-2D look (Godot can do it, with more setup) and PaperZD (replace with `AnimatedSprite3D` / `AnimationPlayer`).
- **Risks to watch:** shared-Resource mutation (always `duplicate(true)` before editing a definition at runtime), the call-stack depth in the turn loop (handled with `call_deferred`), and `Registry` scanning (the `.remap` handling is needed once you export).
- **Order of work:** 0 → 1 → 2 (+ `Registry`, one ability/character/enemy `.tres`) → 3 → 5 (components, then unit scenes) → 6. Phase 4 is just `ActionChoice`.
