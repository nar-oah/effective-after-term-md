# 09 内容数据规范

## 数据驱动原则

所有事件、提案、政党和来访者均使用 Godot `Resource` 保存，避免把内容写死在脚本中。

## ProposalData

```gdscript
class_name ProposalData
extends Resource

@export var id: StringName
@export var title: String
@export_multiline var description: String
@export var proposal_type: StringName # government / interest / party
@export var source_id: StringName

@export var cd_months: int
@export var effects: Dictionary        # stat_name: int
@export var tags: Array[StringName]

@export var lobby_votes: int = 0
@export var donation: int = 0
@export var opposition_votes: int = 0
@export var party_id: StringName

@export var negotiation_options: Array[Resource]
@export var synergy_ids: Array[StringName]
```

## NegotiationOptionData

用于利益集团主动来访时修改提案。

```gdscript
class_name NegotiationOptionData
extends Resource

@export var option_group: StringName
@export var display_name: String
@export var effect_changes: Dictionary
@export var cd_change: int
@export var lobby_vote_change: int
@export var donation_change: int
@export var add_tags: Array[StringName]
@export var remove_tags: Array[StringName]
```

同一 `option_group` 只能选择一个选项，例如“减税幅度”。

## RuntimeProposal

运行时对象，不保存为静态资源。

```gdscript
class_name RuntimeProposal
extends RefCounted

var base_data: ProposalData
var unique_version_id: String
var selected_options: Dictionary
var remaining_cd: int
var is_active: bool
```

`unique_version_id` 用于判断新旧法案中的提案是否完全相同。

## EventData

```gdscript
class_name EventData
extends Resource

@export var id: StringName
@export var title: String
@export var newspaper_title: String
@export_multiline var description: String

@export var initial_months: int
@export var collapse_damage: int
@export var max_delays: int = 2
@export var damage_growth_per_delay: int = 1

@export var solve_conditions: Array[Resource]
@export var delay_tiers: Array[Resource]

@export var predictor_ids: Array[StringName]
@export var truth_chance: float = 1.0
@export var timing_error_range: Vector2i
```

## ConditionData

```gdscript
class_name ConditionData
extends Resource

@export var stat_name: StringName
@export var operator: StringName # gte / lte / tag_gte
@export var target_value: int
```

一组条件默认使用 OR。需要 AND 时，在事件数据中增加 `condition_mode`。

## DelayTierData

```gdscript
class_name DelayTierData
extends Resource

@export var priority: int
@export var conditions: Array[ConditionData]
@export var reset_months: int
```

按 `priority` 从高到低检查。

## PartyData

```gdscript
class_name PartyData
extends Resource

@export var id: StringName
@export var display_name: String
@export var initial_seats: int
@export var annual_proposal_quota: int
@export var term_proposal_target: int
@export var proposal_pool: Array[ProposalData]
```

## VisitorData

```gdscript
class_name VisitorData
extends Resource

@export var id: StringName
@export var display_name: String
@export var visitor_type: StringName
@export var bias_tags: Array[StringName]
@export var dialogue_resource: Resource
@export var possible_event_ids: Array[StringName]
@export var possible_proposal_ids: Array[StringName]
```

## SynergyData

```gdscript
class_name SynergyData
extends Resource

@export var id: StringName
@export var title: String
@export var required_tags: Dictionary
@export var required_proposal_ids: Array[StringName]
@export var effect_changes: Dictionary
@export var opposition_vote_change: int
@export var conversion_rule: Resource
```

## 内容命名

建议 ID 使用统一前缀：

- `evt_`：事件；
- `gov_`：政府政策；
- `int_`：利益集团提案；
- `pty_`：政党提案；
- `syn_`：羁绊；
- `vis_`：来访者；
- `party_`：政党。

示例：

```text
evt_milk_surplus
gov_raise_wage
int_agri_storage
pty_green_quota
syn_wage_consumption
```

## 比赛版内容量

| 类型 | 建议数量 |
|---|---:|
| 政府政策 | 12 |
| 利益集团提案 | 16 |
| 政党提案 | 8 |
| 事件 | 8 |
| 来访者 | 10 |
| 羁绊 | 12 |
| 政党 | 4 |
| 利益集团 | 4 |
