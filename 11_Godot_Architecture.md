# 11 Godot 工程结构

## 推荐版本

Godot 4.x，使用 GDScript。

## 场景结构

```text
Main.tscn
├── GameState
├── MonthManager
├── EventManager
├── ProposalManager
├── BillManager
├── ParliamentManager
├── ElectionManager
├── SaveManager
└── UI
    ├── NewspaperPanel
    ├── TimelinePanel
    ├── VisitorPanel
    ├── NegotiationPanel
    ├── BillBuilderPanel
    ├── ParliamentPanel
    ├── TopStatusBar
    └── ResultPanel
```

## Autoload

建议仅使用两个：

- `DataRegistry`：加载所有 Resource；
- `MetaSave`：保存局外升级。

本局状态保存在 `Main` 场景中，减少全局耦合。

## 关键运行时状态

```gdscript
class_name RunState
extends RefCounted

var month: int = 1
var term_month: int = 1
var years_completed: int = 0
var collapse_value: int = 0
var months_without_collapse_gain: int = 0

var party_seats: Dictionary
var party_year_progress: Dictionary
var party_term_progress: Dictionary

var donation: int = 0
var current_bill: Array[RuntimeProposal]
var proposal_library: Array[RuntimeProposal]
var active_events: Array[RuntimeEvent]
var recorded_event_ids: Array[StringName]
```

## 月份推进

```gdscript
func advance_month() -> void:
    tick_proposal_cds()
    tick_event_countdowns()
    evaluate_due_events()
    update_collapse_recovery_counter()

    run_state.month += 1
    run_state.term_month += 1

    if run_state.month % 12 == 1:
        resolve_year_end()

    if run_state.term_month > 48:
        resolve_election()

    generate_newspaper()
    spawn_monthly_visitor()
```

注意：具体是先结算事件还是先让本月 CD 归零，必须固定。推荐“政策先归零并生效，再检查事件”，让最后一个月赶上的政策有效。

## 法案替换

```gdscript
func enact_bill(new_bill: Array[RuntimeProposal]) -> void:
    var old_by_version := {}
    for proposal in run_state.current_bill:
        old_by_version[proposal.unique_version_id] = proposal

    var enacted: Array[RuntimeProposal] = []

    for proposal in new_bill:
        if old_by_version.has(proposal.unique_version_id):
            enacted.append(old_by_version[proposal.unique_version_id])
        else:
            proposal.remaining_cd = proposal.base_data.cd_months
            proposal.is_active = proposal.remaining_cd <= 0
            enacted.append(proposal)

    run_state.current_bill = enacted
```

旧法案中未被保留的提案自然丢失，其 CD 不保存。

## 投票计算

```gdscript
func calculate_votes(draft: Array[RuntimeProposal], bribe_votes: int) -> int:
    var votes := run_state.party_seats["player"]
    votes += get_single_party_bloc_votes(draft)
    votes += get_lobby_votes(draft)
    votes += bribe_votes
    votes -= get_opposition_votes(draft)
    return clampi(votes, 0, get_total_seats())
```

## 法案数值计算

```gdscript
func calculate_bill_stats() -> Dictionary:
    var stats := get_base_country_stats()
    var active := run_state.current_bill.filter(func(p): return p.is_active)

    apply_base_effects(active, stats)
    var tag_counts := count_tags(active)
    apply_synergies(active, tag_counts, stats)
    apply_conversions(active, tag_counts, stats)
    clamp_stats(stats)

    return stats
```

## 事件结算

```gdscript
func resolve_event(event: RuntimeEvent, stats: Dictionary) -> void:
    if conditions_met(event.data.solve_conditions, stats):
        event.resolve_as_solved()
        return

    for tier in event.data.delay_tiers_sorted:
        if conditions_met(tier.conditions, stats) and event.can_delay():
            event.apply_delay(tier.reset_months)
            return

    add_collapse(event.current_damage)
    event.resolve_as_failed()
```

## 信号

推荐信号：

```gdscript
signal month_advanced(month: int)
signal proposal_cd_changed(proposal_id, remaining: int)
signal proposal_activated(proposal_id)
signal event_confirmed(event_id)
signal event_delayed(event_id, months: int)
signal event_resolved(event_id)
signal collapse_changed(value: int)
signal bill_enacted
signal vote_finished(yes_votes: int, passed: bool)
signal election_resolved(reelected: bool)
```

## 保存

比赛版只需保存：

- 局外执政点；
- 已购买改革；
- 设置项。

本局中途保存属于可选项。时间有限时可只提供每月自动存档。

## 调试面板

开发阶段必须能：

- 跳过月份；
- 修改经济数值；
- 修改党派席位；
- 增减献金；
- 强制生成事件；
- 强制生成来访者；
- 将某提案 CD 设为 0；
- 查看所有隐藏事件及真伪。
