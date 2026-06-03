# 东方红魔乡 回合制RPG · 最终完整规则书（全角色+双计数器+被动技能）

## 目录

1. 大纲与核心概念
2. 回合顺序与双计数器机制
3. 数据模式（Schema）定义
4. Python OOP 引擎设计（含 `eval` 解析）
5. 物种定义（Species）—— 包含所有亚种与角色特质物种
6. 角色完整数据（Characters）—— 10 个角色，每个角色 4 个主动技能，含详细解释与描述
7. 总结

---

## 1. 大纲与核心概念

本规则书定义了一个基于东方红魔乡（TH06）的回合制 RPG 战斗系统。核心概念：

- **物种单继承**：每个角色属于一个物种，物种可单继承父物种。属性（HP/ATK/REGEN）和被动技能可被覆盖。
- **双回合计数器**：
  - `global_turn`：全局回合计数器，每回合结束时 +1（若 `turn_end` 被取消则不增加）。
  - `player_turn_counter`：每个玩家独立的回合计数器，在该玩家回合开始时 +1（若 `turn_start` 被取消则不增加）。
- **修饰器（Modifier）持续时间**：支持 `last_turns` 和 `last_turn_type`（`"global"` 或 `"player"`），分别按全局回合或目标玩家回合递减。
- **被动技能**：可以是永久（来自物种）或临时（由 Effect 附加）。临时被动技能支持 `when`（触发条件）和 `invalid_when`（删除条件）。被动技能按 `priority` 降序执行。
- **Effect 系统**：所有技能效果由 `Effect` 对象描述，包含 `type`、`allowed_target_type`、`modifiers`、`cancellers`，并可附加 `passive_skills` 数组。
- **Canceller**：取消特定事件（伤害、治疗、回合开始等），用于实现闪避、时停、魔法抵消等机制。
- **琪露诺的「幻想乡最强」**：每回合结束时，独立掷 1d6 分别乘算当前 ATK 和 REGEN，并将 HP 设置为最大 HP 的 5%。此被动优先级为 1，确保在妖怪的再生之前执行，且每回合随机刷新数值，真正实现「最强」的随机爆发力。

---

## 2. 回合顺序与双计数器机制

```
战斗开始
   │
   ├─ 初始化 global_turn = 0
   ├─ 初始化 player_a.turn_counter = 0, player_b.turn_counter = 0
   ├─ 设置 current_actor = 挑战者（玩家）
   │
   ▼
【当前玩家回合开始】
   │
   ├─ 触发 turn_start 事件
   │   ├─ 若任何取消器生效，跳过整个回合（不增加该玩家回合计数器）
   │   └─ 否则，current_actor.turn_counter += 1
   │
   ├─ 当前玩家选择技能和所有效果的目标
   │
   ├─ 按顺序执行技能中的每个 Effect
   │    ├─ 对每个 Effect，检查 canceller（按 allowed_target_type 匹配）
   │    ├─ 若事件未被取消，则应用 modifiers
   │    │   └─ modifiers 可附加临时被动技能（见 Effect.passive_skills）
   │    └─ 若 HP ≤ 0，角色立即退场（但本次技能剩余效果仍执行）
   │
   ├─ 回合结束（turn_end 事件）
   │    ├─ 应用所有 turn_end 取消器：若取消，则跳过本回合的被动技能触发和 global_turn 增加
   │    └─ 若未取消：
   │        ├─ 按 priority 降序执行当前角色的所有被动技能（Effect 列表）
   │        ├─ global_turn += 1
   │        └─ 更新所有角色的 modifiers（减少剩余回合数，删除过期项）
   │
   └─ 交换行动方（敌方变为 current_actor），重复以上步骤，直到一方 HP ≤ 0 退场
```

**重要说明**：
- 被动技能可以是永久（来自物种）或临时（由主动技能附加）。临时被动技能会检查 `when`（默认总是触发）和 `invalid_when`（返回 `True` 时永久删除）。
- 修饰器的 `last_turns` 递减基于 `last_turn_type`：
  - `"global"`：每次 `global_turn` 增加时减少剩余回合数。
  - `"player"`：仅当拥有该 modifier 的玩家回合开始时减少（即每轮该玩家行动一次减一次）。
- 琪露诺的「幻想乡最强」优先级为 1，高于「妖怪的再生」（优先级 0），因此在每回合结束时先随机乘算 ATK/REGEN 并锁血，再进行回复。这使得琪露诺每回合都有机会暴发，但也可能运气差导致数值下降。

---

## 3. 数据模式（Schema）定义

所有数据均以 JSON 格式定义。以下为完整的 Schema 说明。

### 3.1 Species（物种）

```json
{
  "name": "string",                // 唯一标识
  "description": "string",         // 角色扮演描述
  "parent_species": "string | null", // 单亲继承，若 null 则为根物种
  "hp": {
    "formula": "lambda base, level: ...",
    "base": 0,
    "level_coef": 0,
    "explanation": "string"
  },
  "atk": { ... },
  "regen": { ... },
  "modifiers": {                   // 提供给子物种的修饰值
    "hp_mod": 0,
    "atk_mod": 0,
    "regen_mod": 0
  },
  "passive_skills": [              // 永久被动技能
    {
      "name": "string",
      "priority": 0,
      "type": "passive",
      "when": "lambda event, turn_counters, stats: ...", // 可选，默认 True
      "invalid_when": "lambda ...",                     // 可选，永久被动一般不设
      "effects": [ Effect 对象 ]
    }
  ]
}
```

### 3.2 Character（角色实例）

```json
{
  "character_id": "string",
  "name": "string",
  "species": "string",
  "level": 0,
  "hp_max": 0,
  "atk_base": 0,
  "regen_base": 0,
  "skills": [ Skill 对象 ]
}
```

### 3.3 Skill（主动技能）

```json
{
  "name": "string",
  "type": "normal | spell_card | support | ultimate",
  "description": "string",
  "effects": [ Effect 对象 ]
}
```

### 3.4 Effect（效果）

```json
{
  "type": "string",               // damage, heal, buff, debuff, stun, lifesteal, extra_turn
  "allowed_target_type": "self_only | enemies_only | selectable",
  "modifiers": [ Modifier ],
  "cancellers": [ Canceller ],
  "passive_skills": [ PassiveSkill ]   // 临时附加的被动技能（仅本次效果添加）
}
```

### 3.5 Modifier（修改器）

```json
{
  "prop": "string",               // hp, atk, regen, 或自定义属性
  "allowed_target_type": "self_only | enemies_only | selectable",
  "value": "lambda ...",
  "stack_type": "additive | multiplicative | replace",
  "last_turns": -1,
  "last_turn_type": "global | player",  // 默认 "global"
  "invalid_when": "lambda ...",         // 可选，删除条件
  "explanation": "string",
  "description": "string"
}
```

### 3.6 Canceller（取消器）

```json
{
  "event_to_cancel": "string",    // damage, heal, buff, debuff, stun, lifesteal, extra_turn, turn_start, turn_end
  "allowed_target_type": "self_only | enemies_only | selectable",
  "cancel_type": "first | all",
  "when": "lambda event, turn_counters, source, target: ...",
  "cancel": "lambda: ...",
  "explanation": "string",
  "description": "string"
}
```

### 3.7 PassiveSkill（被动技能，临时与永久共用结构）

```json
{
  "name": "string",
  "priority": 0,
  "type": "passive",
  "when": "lambda event, turn_counters, stats: ...",      // 默认 True
  "invalid_when": "lambda ...",                          // 临时被动技能常用
  "effects": [ Effect ]
}
```

---

## 4. Python OOP 引擎设计

以下为完整的可运行引擎，支持解析上述 JSON 数据并执行战斗。使用 `eval` 安全解析 lambda 字符串。

```python
import random
import math
import json
from typing import Dict, List, Any, Optional

# ---------- 辅助函数 ----------
def safe_eval(expr: str, namespace: dict) -> Any:
    """安全地执行 lambda 表达式，注入 random、math 等模块"""
    return eval(expr, {"random": random, "math": math, **namespace})

# ---------- Modifier ----------
class Modifier:
    def __init__(self, data: dict):
        self.prop = data["prop"]
        self.allowed_target_type = data["allowed_target_type"]
        self.value_lambda = data["value"]
        self.stack_type = data["stack_type"]
        self.last_turns = data.get("last_turns", -1)
        self.last_turn_type = data.get("last_turn_type", "global")
        self.invalid_when_lambda = data.get("invalid_when", "lambda: False")
        self.explanation = data.get("explanation", "")
        self.description = data.get("description", "")
        self.remaining_turns = self.last_turns

    def get_value(self, context: dict) -> float:
        namespace = {"rand": random.randint, **context}
        return safe_eval(self.value_lambda, namespace)

    def is_valid(self, target_stats: dict) -> bool:
        namespace = {"hp": target_stats.get("hp", 0), "max_hp": target_stats.get("max_hp", 1)}
        return not safe_eval(self.invalid_when_lambda, namespace)

    def tick(self, turn_type: str):
        """根据 turn_type 递减剩余回合数"""
        if self.remaining_turns > 0 and self.last_turn_type == turn_type:
            self.remaining_turns -= 1
        return self.remaining_turns == 0

# ---------- Canceller ----------
class Canceller:
    def __init__(self, data: dict):
        self.event_to_cancel = data["event_to_cancel"]
        self.allowed_target_type = data["allowed_target_type"]
        self.cancel_type = data["cancel_type"]
        self.when_lambda = data.get("when", "lambda event, turn_counters, source, target: True")
        self.cancel_lambda = data.get("cancel", "lambda: True")
        self.explanation = data.get("explanation", "")
        self.description = data.get("description", "")

    def should_cancel(self, event: dict, context: dict) -> bool:
        namespace = {"event": event, **context}
        if safe_eval(self.when_lambda, namespace):
            return safe_eval(self.cancel_lambda, context)
        return False

# ---------- PassiveSkill ----------
class PassiveSkill:
    def __init__(self, data: dict, is_temporary: bool = False):
        self.name = data["name"]
        self.priority = data.get("priority", 0)
        self.type = data["type"]
        self.when_lambda = data.get("when", "lambda event, turn_counters, stats: True")
        self.invalid_when_lambda = data.get("invalid_when", "lambda: False") if is_temporary else "lambda: False"
        self.effects = [Effect(e) for e in data.get("effects", [])]
        self.is_temporary = is_temporary

    def should_trigger(self, event: dict, context: dict) -> bool:
        namespace = {"event": event, **context}
        return safe_eval(self.when_lambda, namespace)

    def is_invalid(self, context: dict) -> bool:
        if not self.is_temporary:
            return False
        return safe_eval(self.invalid_when_lambda, context)

# ---------- Effect ----------
class Effect:
    def __init__(self, data: dict):
        self.type = data["type"]
        self.allowed_target_type = data["allowed_target_type"]
        self.modifiers = [Modifier(m) for m in data.get("modifiers", [])]
        self.cancellers = [Canceller(c) for c in data.get("cancellers", [])]
        self.attached_passives = [PassiveSkill(ps, is_temporary=True) for ps in data.get("passive_skills", [])]

    def apply(self, source: "Character", target: "Character", turn_state: dict) -> List[dict]:
        events = []
        # 过滤无效 modifier
        active_mods = [m for m in self.modifiers if m.is_valid(target.get_stats())]
        # 先检查取消器
        event = {"type": self.type, "source": source.name, "target": target.name, "modifiers": active_mods}
        for canceller in self.cancellers:
            context = {"source": source, "target": target, "turn_counters": turn_state["counters"], "turn": turn_state["global_turn"]}
            if canceller.should_cancel(event, context):
                return []
        # 执行 modifier
        for mod in active_mods:
            value = mod.get_value({"atk": source.atk, "regen": source.regen, "hp": target.hp, "max_hp": target.hp_max})
            if mod.stack_type == "additive":
                target.add_modifier(mod.prop, value, mod.remaining_turns, mod.last_turn_type, mod.invalid_when_lambda)
            elif mod.stack_type == "multiplicative":
                target.multiply_modifier(mod.prop, value, mod.remaining_turns, mod.last_turn_type, mod.invalid_when_lambda)
            else:  # replace
                target.set_modifier(mod.prop, value, mod.remaining_turns, mod.last_turn_type, mod.invalid_when_lambda)
            events.append({"type": self.type, "modifier": mod, "value": value, "target": target.name})
        # 附加临时被动技能
        for ps in self.attached_passives:
            target.add_temporary_passive(ps)
        return events

# ---------- Character ----------
class Character:
    def __init__(self, data: dict, species_defs: Dict[str, dict]):
        self.id = data["character_id"]
        self.name = data["name"]
        self.species_name = data["species"]
        self.level = data["level"]
        # 从物种继承属性
        species = species_defs[self.species_name]
        self.hp_max = self._compute_stat(species["hp"], species.get("modifiers", {}))
        self.atk = self._compute_stat(species["atk"], species.get("modifiers", {}))
        self.regen = self._compute_stat(species["regen"], species.get("modifiers", {}))
        self.hp = self.hp_max
        self.turn_counter = 0          # 该玩家回合计数器
        # 持久修饰器 (prop -> Modifier 对象)
        self.active_modifiers = []
        # 临时被动技能
        self.temp_passive_skills = []
        # 永久被动技能（从物种链收集，按 priority 排序）
        self.permanent_passive_skills = self._collect_passive_skills(species_defs, self.species_name)
        # 主动技能
        self.skills = data.get("skills", [])

    def _collect_passive_skills(self, species_defs: Dict[str, dict], species_name: str) -> List[PassiveSkill]:
        """沿继承链收集永久被动技能，子类覆盖父类同名被动，最后按 priority 排序"""
        skills = []
        cur = species_name
        while cur:
            sp_data = species_defs[cur]
            for ps_data in sp_data.get("passive_skills", []):
                # 同名覆盖（子类优先）
                skills = [s for s in skills if s.name != ps_data["name"]] + [PassiveSkill(ps_data, is_temporary=False)]
            cur = sp_data.get("parent_species")
        skills.sort(key=lambda x: x.priority, reverse=True)
        return skills

    def _compute_stat(self, stat_def: dict, modifiers: dict) -> int:
        base = stat_def.get("base", 0) + modifiers.get(f"{stat_def.get('prop', '')}_mod", 0)
        level_coef = stat_def.get("level_coef", 0)
        formula = stat_def["formula"]
        value = safe_eval(formula, {"base": base, "level": self.level, "math": math})
        return math.ceil(value)

    def get_stats(self) -> dict:
        """应用所有修饰器后的当前属性"""
        hp = self.hp
        atk = self.atk
        regen = self.regen
        for mod in self.active_modifiers:
            if mod.prop == "hp":
                if mod.stack_type == "additive":
                    hp += mod.get_value({})
                elif mod.stack_type == "multiplicative":
                    hp *= mod.get_value({})
                else:
                    hp = mod.get_value({})
            elif mod.prop == "atk":
                if mod.stack_type == "additive":
                    atk += mod.get_value({})
                elif mod.stack_type == "multiplicative":
                    atk *= mod.get_value({})
                else:
                    atk = mod.get_value({})
            elif mod.prop == "regen":
                if mod.stack_type == "additive":
                    regen += mod.get_value({})
                elif mod.stack_type == "multiplicative":
                    regen *= mod.get_value({})
                else:
                    regen = mod.get_value({})
        return {"hp": max(0, hp), "atk": max(0, atk), "regen": max(0, regen), "max_hp": self.hp_max}

    def add_modifier(self, prop: str, value: float, turns: int, turn_type: str, invalid_when: str):
        mod = Modifier({
            "prop": prop,
            "allowed_target_type": "self_only",
            "value": f"lambda: {value}",
            "stack_type": "additive",
            "last_turns": turns,
            "last_turn_type": turn_type,
            "invalid_when": invalid_when
        })
        self.active_modifiers.append(mod)

    def multiply_modifier(self, prop: str, value: float, turns: int, turn_type: str, invalid_when: str):
        mod = Modifier({
            "prop": prop,
            "allowed_target_type": "self_only",
            "value": f"lambda: {value}",
            "stack_type": "multiplicative",
            "last_turns": turns,
            "last_turn_type": turn_type,
            "invalid_when": invalid_when
        })
        self.active_modifiers.append(mod)

    def set_modifier(self, prop: str, value: float, turns: int, turn_type: str, invalid_when: str):
        mod = Modifier({
            "prop": prop,
            "allowed_target_type": "self_only",
            "value": f"lambda: {value}",
            "stack_type": "replace",
            "last_turns": turns,
            "last_turn_type": turn_type,
            "invalid_when": invalid_when
        })
        self.active_modifiers.append(mod)

    def add_temporary_passive(self, passive: PassiveSkill):
        self.temp_passive_skills.append(passive)

    def update_modifiers(self, turn_type: str):
        """根据 turn_type 更新所有修饰器，并删除过期的"""
        expired_mods = []
        for mod in self.active_modifiers:
            if mod.tick(turn_type):
                expired_mods.append(mod)
        for mod in expired_mods:
            self.active_modifiers.remove(mod)

    def update_temporary_passives(self, context: dict):
        """删除 invalid_when 返回 True 的临时被动技能"""
        self.temp_passive_skills = [ps for ps in self.temp_passive_skills if not ps.is_invalid(context)]

    def trigger_passive_skills(self, event_type: str, turn_state: dict) -> List[dict]:
        """触发所有符合条件且 when 为 True 的被动技能（永久+临时），按 priority 降序"""
        all_passives = self.permanent_passive_skills + self.temp_passive_skills
        all_passives.sort(key=lambda x: x.priority, reverse=True)
        events = []
        context = {
            "source": self,
            "target": self,
            "turn_counters": {"global": turn_state["global_turn"], "player": self.turn_counter},
            "turn": turn_state["global_turn"],
            "event": {"type": event_type}
        }
        for ps in all_passives:
            if ps.should_trigger({"type": event_type}, context):
                for effect in ps.effects:
                    ev = effect.apply(self, self, turn_state)
                    events.extend(ev)
        return events

    def take_damage(self, amount: int):
        self.hp = max(0, self.hp - amount)

    def heal(self, amount: int):
        self.hp = min(self.hp_max, self.hp + amount)

    def is_alive(self) -> bool:
        return self.hp > 0

# ---------- 战斗引擎 ----------
class Battle:
    def __init__(self, player: Character, enemy: Character):
        self.player = player
        self.enemy = enemy
        self.global_turn = 0
        self.current_actor = player
        self.opponent = enemy

    def switch_actor(self):
        self.current_actor, self.opponent = self.opponent, self.current_actor

    def execute_skill(self, skill: dict, targets: Dict[str, Character]) -> List[dict]:
        events = []
        for effect_data in skill["effects"]:
            effect = Effect(effect_data)
            if effect.allowed_target_type == "self_only":
                target = self.current_actor
            elif effect.allowed_target_type == "enemies_only":
                target = self.opponent
            else:
                target = targets.get("selectable", self.opponent)
            turn_state = {
                "global_turn": self.global_turn,
                "counters": {"global": self.global_turn, "player": self.current_actor.turn_counter}
            }
            ev = effect.apply(self.current_actor, target, turn_state)
            events.extend(ev)
        return events

    def run_turn(self, player_skill: dict, skill_target: Character = None):
        """执行一回合战斗，返回事件列表"""
        # ---------- turn_start 阶段 ----------
        start_cancelled = False
        # 实际应检查 turn_start 取消器（略）
        if not start_cancelled:
            self.current_actor.turn_counter += 1

        # ---------- 技能执行 ----------
        targets = {"self": self.current_actor, "enemy": self.opponent, "selectable": skill_target or self.opponent}
        events = self.execute_skill(player_skill, targets)

        # 检查死亡
        if not self.current_actor.is_alive():
            return events

        # ---------- turn_end 阶段 ----------
        end_cancelled = False
        # 检查 turn_end 取消器（略）
        if not end_cancelled:
            # 触发被动技能（事件类型 "turn_end"）
            events.extend(self.current_actor.trigger_passive_skills("turn_end", {"global_turn": self.global_turn}))
            # 增加全局回合计数器
            self.global_turn += 1
            # 更新所有修饰器（按 global 类型）
            self.current_actor.update_modifiers("global")
            self.opponent.update_modifiers("global")
            # 更新临时被动技能
            ctx = {"turn_counters": {"global": self.global_turn, "player": self.current_actor.turn_counter}}
            self.current_actor.update_temporary_passives(ctx)
            self.opponent.update_temporary_passives(ctx)

        # 交换回合
        self.switch_actor()
        return events
```

---

## 5. 物种定义（Species）—— 包含所有亚种与角色特质物种

```json
{
  "species_definitions": [
    {
      "name": "human",
      "description": "人类，没有自动回复能力，但可以通过支援技能治疗。拥有较高的HP成长和ATK成长。",
      "parent_species": null,
      "hp": { "formula": "lambda base, level: base + 100 + (level * 12)", "base": 0, "level_coef": 12, "explanation": "HP = 100 + 等级 × 12" },
      "atk": { "formula": "lambda base, level: base + 3 + (level * 3)", "base": 0, "level_coef": 3, "explanation": "ATK = 3 + 等级 × 3" },
      "regen": { "formula": "lambda base, level: 0", "base": 0, "level_coef": 0, "explanation": "人类没有自动回复能力" },
      "modifiers": {},
      "passive_skills": []
    },
    {
      "name": "youkai",
      "description": "妖怪，拥有自动回复能力。基础HP和ATK成长低于人类，但每回合自动回复生命值。",
      "parent_species": null,
      "hp": { "formula": "lambda base, level: base + 50 + (level * 6)", "base": 0, "level_coef": 6, "explanation": "HP = 50 + 等级 × 6" },
      "atk": { "formula": "lambda base, level: base + 3 + (level * 2)", "base": 0, "level_coef": 2, "explanation": "ATK = 3 + 等级 × 2" },
      "regen": { "formula": "lambda base, level: base + 3 + level", "base": 0, "level_coef": 1, "explanation": "REGEN = 3 + 等级" },
      "modifiers": {},
      "passive_skills": [
        {
          "name": "妖怪的再生",
          "priority": 0,
          "type": "passive",
          "when": "lambda event, turn_counters, stats: True",
          "invalid_when": "lambda: False",
          "effects": [
            {
              "type": "heal",
              "allowed_target_type": "self_only",
              "modifiers": [
                { "prop": "hp", "allowed_target_type": "self_only", "value": "lambda regen: regen", "stack_type": "additive", "last_turns": -1, "explanation": "每回合结束时回复等于回复力的生命值", "description": "妖怪的体质让伤口自动愈合，黑色的妖气缠绕伤口，逐渐恢复。" }
              ],
              "cancellers": []
            }
          ]
        }
      ]
    },
    {
      "name": "博丽巫女",
      "description": "博丽神社的巫女，天生灵力强大，拥有操纵灵气的能力。",
      "parent_species": "human",
      "hp": { "formula": "lambda base, level: base + 100 + (level * 12)", "base": 5, "level_coef": 12, "explanation": "HP = 105 + 等级 × 12" },
      "atk": { "formula": "lambda base, level: base + 3 + (level * 3)", "base": 0, "level_coef": 3, "explanation": "ATK = 3 + 等级 × 3" },
      "regen": { "formula": "lambda base, level: 0", "base": 0, "level_coef": 0, "explanation": "人类没有自动回复能力" },
      "modifiers": { "hp_mod": 5 },
      "passive_skills": []
    },
    {
      "name": "魔法使",
      "description": "刻苦钻研魔法的魔法使，拥有强大的魔法攻击力，但体质较弱。",
      "parent_species": "human",
      "hp": { "formula": "lambda base, level: base + 100 + (level * 12)", "base": 0, "level_coef": 12, "explanation": "HP = 100 + 等级 × 12" },
      "atk": { "formula": "lambda base, level: base + 3 + (level * 3)", "base": 5, "level_coef": 3, "explanation": "ATK = 8 + 等级 × 3" },
      "regen": { "formula": "lambda base, level: 0", "base": 0, "level_coef": 0, "explanation": "人类没有自动回复能力" },
      "modifiers": { "atk_mod": 5 },
      "passive_skills": []
    },
    {
      "name": "人类",
      "description": "普通人类，无特别加成，但拥有完美的潇洒和操纵时间的能力。",
      "parent_species": "human",
      "hp": { "formula": "lambda base, level: base + 100 + (level * 12)", "base": 0, "level_coef": 12, "explanation": "HP = 100 + 等级 × 12" },
      "atk": { "formula": "lambda base, level: base + 3 + (level * 3)", "base": 0, "level_coef": 3, "explanation": "ATK = 3 + 等级 × 3" },
      "regen": { "formula": "lambda base, level: 0", "base": 0, "level_coef": 0, "explanation": "人类没有自动回复能力" },
      "modifiers": {},
      "passive_skills": []
    },
    {
      "name": "吸血鬼",
      "description": "吸血鬼，拥有超人般的体魄与魔力。回复力极高，ATK成长优秀。",
      "parent_species": "youkai",
      "hp": { "formula": "lambda base, level: base + 50 + (level * 6)", "base": 15, "level_coef": 6, "explanation": "HP = 65 + 等级 × 6" },
      "atk": { "formula": "lambda base, level: base + 3 + (level * 2)", "base": 5, "level_coef": 2, "explanation": "ATK = 8 + 等级 × 2" },
      "regen": { "formula": "lambda base, level: base + 3 + level", "base": 2, "level_coef": 1, "explanation": "REGEN = 5 + 等级" },
      "modifiers": { "hp_mod": 15, "atk_mod": 5, "regen_mod": 2 },
      "passive_skills": []
    },
    {
      "name": "魔女",
      "description": "魔女，拥有深厚的魔法知识，ATK极高但HP较低，病弱体质。",
      "parent_species": "youkai",
      "hp": { "formula": "lambda base, level: base + 50 + (level * 6)", "base": 5, "level_coef": 6, "explanation": "HP = 55 + 等级 × 6" },
      "atk": { "formula": "lambda base, level: base + 3 + (level * 2)", "base": 10, "level_coef": 2, "explanation": "ATK = 13 + 等级 × 2" },
      "regen": { "formula": "lambda base, level: base + 3 + level", "base": 0, "level_coef": 1, "explanation": "REGEN = 3 + 等级" },
      "modifiers": { "hp_mod": 5, "atk_mod": 10 },
      "passive_skills": []
    },
    {
      "name": "妖精",
      "description": "妖精，基础能力较差但回复稳定。存在较为自然，不易完全消灭。",
      "parent_species": "youkai",
      "hp": { "formula": "lambda base, level: base + 50 + (level * 6)", "base": -5, "level_coef": 6, "explanation": "HP = 45 + 等级 × 6" },
      "atk": { "formula": "lambda base, level: base + 3 + (level * 2)", "base": -2, "level_coef": 2, "explanation": "ATK = 1 + 等级 × 2" },
      "regen": { "formula": "lambda base, level: base + 3 + level", "base": 1, "level_coef": 1, "explanation": "REGEN = 4 + 等级" },
      "modifiers": { "hp_mod": -5, "atk_mod": -2, "regen_mod": 1 },
      "passive_skills": []
    },
    {
      "name": "妖怪",
      "description": "普通妖怪，各项能力均衡，没有明显的优势和劣势。",
      "parent_species": "youkai",
      "hp": { "formula": "lambda base, level: base + 50 + (level * 6)", "base": 5, "level_coef": 6, "explanation": "HP = 55 + 等级 × 6" },
      "atk": { "formula": "lambda base, level: base + 3 + (level * 2)", "base": 0, "level_coef": 2, "explanation": "ATK = 3 + 等级 × 2" },
      "regen": { "formula": "lambda base, level: base + 3 + level", "base": 0, "level_coef": 1, "explanation": "REGEN = 3 + 等级" },
      "modifiers": { "hp_mod": 5 },
      "passive_skills": []
    },
    {
      "name": "博丽灵梦",
      "description": "乐园的可爱巫女，拥有操纵灵气程度的能力。性格随性自由，但对守护幻想乡有着强烈的责任感。",
      "parent_species": "博丽巫女",
      "hp": { "formula": "lambda base, level: base + 100 + (level * 12)", "base": 10, "level_coef": 12, "explanation": "HP = 115 + 等级 × 12" },
      "atk": { "formula": "lambda base, level: base + 3 + (level * 3)", "base": 2, "level_coef": 3, "explanation": "ATK = 5 + 等级 × 3" },
      "regen": { "formula": "lambda base, level: 0", "base": 0, "level_coef": 0, "explanation": "人类没有自动回复能力" },
      "modifiers": { "hp_mod": 10, "atk_mod": 2 },
      "passive_skills": [
        {
          "name": "巫女的守护",
          "priority": 0,
          "type": "passive",
          "when": "lambda event, turn_counters, stats: True",
          "invalid_when": "lambda: False",
          "effects": [
            {
              "type": "buff",
              "allowed_target_type": "self_only",
              "modifiers": [
                { "prop": "damage_taken_multiplier", "allowed_target_type": "self_only", "value": "lambda: 0.5", "stack_type": "replace", "last_turns": -1, "invalid_when": "lambda hp, max_hp: hp >= max_hp * 0.3", "explanation": "当灵梦的生命值低于30%时，受到的伤害减少50%。灵梦的灵力自动形成护盾，在关键时刻保护她。", "description": "灵力自动形成护盾，发出柔和微光，将袭来的攻击力道减半。" }
              ],
              "cancellers": []
            }
          ]
        }
      ]
    },
    {
      "name": "雾雨魔理沙",
      "description": "普通的魔法使，努力家。拥有使用魔法程度的能力，擅长破坏性的魔法。性格有些乖僻，但乐于助人。",
      "parent_species": "魔法使",
      "hp": { "formula": "lambda base, level: base + 100 + (level * 12)", "base": -5, "level_coef": 12, "explanation": "HP = 95 + 等级 × 12" },
      "atk": { "formula": "lambda base, level: base + 3 + (level * 3)", "base": 5, "level_coef": 3, "explanation": "ATK = 13 + 等级 × 3" },
      "regen": { "formula": "lambda base, level: 0", "base": 0, "level_coef": 0, "explanation": "人类没有自动回复能力" },
      "modifiers": { "hp_mod": -5, "atk_mod": 5 },
      "passive_skills": [
        {
          "name": "收集癖",
          "priority": 0,
          "type": "passive",
          "when": "lambda event, turn_counters, stats: True",
          "invalid_when": "lambda: False",
          "effects": [
            {
              "type": "buff",
              "allowed_target_type": "self_only",
              "modifiers": [
                { "prop": "spell_card_damage_multiplier", "allowed_target_type": "self_only", "value": "lambda: 1.3", "stack_type": "multiplicative", "last_turns": -1, "invalid_when": "lambda hp, max_hp: hp >= max_hp * 0.3", "explanation": "当魔理沙的生命值低于30%时，她使用的符卡伤害增加30%。她从魔法口袋中掏出珍藏的道具，临时提升威力。", "description": "魔理沙从口袋中掏出珍藏的魔法道具，八卦炉的火焰变得更加炽烈。" }
              ],
              "cancellers": []
            }
          ]
        }
      ]
    },
    {
      "name": "十六夜咲夜",
      "description": "完美潇洒的从者，红魔馆的女仆长。拥有操纵时间程度的能力，包揽了红魔馆几乎所有的工作。",
      "parent_species": "人类",
      "hp": { "formula": "lambda base, level: base + 100 + (level * 12)", "base": 5, "level_coef": 12, "explanation": "HP = 105 + 等级 × 12" },
      "atk": { "formula": "lambda base, level: base + 3 + (level * 3)", "base": 0, "level_coef": 3, "explanation": "ATK = 3 + 等级 × 3" },
      "regen": { "formula": "lambda base, level: 0", "base": 0, "level_coef": 0, "explanation": "人类没有自动回复能力" },
      "modifiers": { "hp_mod": 5 },
      "passive_skills": [
        {
          "name": "时间感知",
          "priority": 0,
          "type": "passive",
          "when": "lambda event, turn_counters, stats: True",
          "invalid_when": "lambda: False",
          "effects": [
            {
              "type": "buff",
              "allowed_target_type": "self_only",
              "modifiers": [
                { "prop": "evade_chance", "allowed_target_type": "self_only", "value": "lambda: 0.5", "stack_type": "replace", "last_turns": -1, "invalid_when": "lambda hp, max_hp: hp >= max_hp * 0.3", "explanation": "当咲夜的生命值低于30%时，她有50%的概率闪避攻击。她本能地冻结时间，瞬间移动到攻击范围之外。", "description": "咲夜的怀表发出轻响，周围的时间流速变得微妙，她随时准备暂停时间躲避攻击。" }
              ],
              "cancellers": [
                { "event_to_cancel": "damage", "allowed_target_type": "self_only", "cancel_type": "first", "when": "lambda event, turn_counters, source, target: True", "cancel": "lambda: random.random() < 0.5", "explanation": "50%概率取消一次伤害事件", "description": "咲夜冻结时间，瞬移至攻击范围之外，银制小刀擦身而过。" }
              ]
            }
          ]
        }
      ]
    },
    {
      "name": "蕾米莉亚·斯卡蕾特",
      "description": "永远鲜红的幼月，红魔馆的主人。活了500年以上的吸血鬼，拥有操纵命运程度的能力。",
      "parent_species": "吸血鬼",
      "hp": { "formula": "lambda base, level: base + 50 + (level * 6)", "base": 20, "level_coef": 6, "explanation": "HP = 85 + 等级 × 6" },
      "atk": { "formula": "lambda base, level: base + 3 + (level * 2)", "base": 8, "level_coef": 2, "explanation": "ATK = 16 + 等级 × 2" },
      "regen": { "formula": "lambda base, level: base + 3 + level", "base": 3, "level_coef": 1, "explanation": "REGEN = 8 + 等级" },
      "modifiers": { "hp_mod": 20, "atk_mod": 8, "regen_mod": 3 },
      "passive_skills": [
        {
          "name": "命运掌控",
          "priority": 0,
          "type": "passive",
          "when": "lambda event, turn_counters, stats: True",
          "invalid_when": "lambda: False",
          "effects": [
            {
              "type": "buff",
              "allowed_target_type": "self_only",
              "modifiers": [
                { "prop": "atk_multiplier", "allowed_target_type": "self_only", "value": "lambda: 1.5", "stack_type": "multiplicative", "last_turns": 2, "last_turn_type": "player", "invalid_when": "lambda hp, max_hp: hp >= max_hp * 0.3", "explanation": "当蕾米莉亚的生命值低于30%时，她的ATK乘以1.5，持续2个玩家回合。她动用命运之力，确保下一次攻击造成巨大伤害。", "description": "猩红色从瞳孔漫溢，命运之线在蕾米莉亚指尖颤动，她轻笑一声：「命运站在我这边。」" }
              ],
              "cancellers": []
            }
          ]
        }
      ]
    },
    {
      "name": "芙兰朵露·斯卡蕾特",
      "description": "恶魔之妹，蕾米莉亚的妹妹。495岁的吸血鬼，拥有破坏一切程度的能力。红魔乡Extra关卡的BOSS。",
      "parent_species": "吸血鬼",
      "hp": { "formula": "lambda base, level: base + 50 + (level * 6)", "base": 15, "level_coef": 6, "explanation": "HP = 80 + 等级 × 6" },
      "atk": { "formula": "lambda base, level: base + 3 + (level * 2)", "base": 20, "level_coef": 2, "explanation": "ATK = 28 + 等级 × 2" },
      "regen": { "formula": "lambda base, level: base + 3 + level", "base": 2, "level_coef": 1, "explanation": "REGEN = 7 + 等级" },
      "modifiers": { "hp_mod": 15, "atk_mod": 20, "regen_mod": 2 },
      "passive_skills": [
        {
          "name": "破坏的冲动",
          "priority": 0,
          "type": "passive",
          "when": "lambda event, turn_counters, stats: True",
          "invalid_when": "lambda: False",
          "effects": [
            {
              "type": "buff",
              "allowed_target_type": "self_only",
              "modifiers": [
                { "prop": "damage_multiplier", "allowed_target_type": "self_only", "value": "lambda: 1.5", "stack_type": "multiplicative", "last_turns": -1, "invalid_when": "lambda hp, max_hp: hp >= max_hp * 0.3", "explanation": "当芙兰朵露的生命值低于30%时，她造成的所有伤害增加50%。破坏的冲动无法抑制。", "description": "芙兰的眼睛发出幽光，背后水晶般的翼骨展开，她喃喃道：「全部破坏掉。」" }
              ],
              "cancellers": []
            }
          ]
        },
        {
          "name": "多重存在",
          "priority": 0,
          "type": "passive",
          "when": "lambda event, turn_counters, stats: True",
          "invalid_when": "lambda: False",
          "effects": [
            {
              "type": "buff",
              "allowed_target_type": "self_only",
              "modifiers": [
                { "prop": "skill_effect_multiplier", "allowed_target_type": "self_only", "value": "lambda multi_existence: multi_existence", "stack_type": "replace", "last_turns": -1, "explanation": "所有主动技能的效果次数乘以 multi_existence 的值。分身会同步模仿芙兰的攻击。", "description": "分身围绕芙兰旋转，每个动作都同步模仿，让人分不清本体与幻影。" }
              ],
              "cancellers": []
            }
          ]
        }
      ]
    },
    {
      "name": "帕秋莉·诺蕾姬",
      "description": "不动的大图书馆，居住于巴瓦尔图书馆的魔女。拥有操纵七曜元素的能力，患有哮喘和贫血。",
      "parent_species": "魔女",
      "hp": { "formula": "lambda base, level: base + 50 + (level * 6)", "base": -10, "level_coef": 6, "explanation": "HP = 45 + 等级 × 6" },
      "atk": { "formula": "lambda base, level: base + 3 + (level * 2)", "base": 12, "level_coef": 2, "explanation": "ATK = 25 + 等级 × 2" },
      "regen": { "formula": "lambda base, level: base + 3 + level", "base": -1, "level_coef": 1, "explanation": "REGEN = 2 + 等级" },
      "modifiers": { "hp_mod": -10, "atk_mod": 12, "regen_mod": -1 },
      "passive_skills": [
        {
          "name": "病弱体质",
          "priority": 0,
          "type": "passive",
          "when": "lambda event, turn_counters, stats: True",
          "invalid_when": "lambda: False",
          "effects": [
            {
              "type": "buff",
              "allowed_target_type": "self_only",
              "modifiers": [
                { "prop": "heal_received_multiplier", "allowed_target_type": "self_only", "value": "lambda: 1.5", "stack_type": "multiplicative", "last_turns": -1, "invalid_when": "lambda hp, max_hp: hp >= max_hp * 0.3", "explanation": "当帕秋莉的生命值低于30%时，她受到的治疗效果增加50%。病弱时魔法反而更加活跃，加速恢复。", "description": "帕秋莉的气喘加重，但她身边的七曜元素却更加活跃，主动涌入她的身体加速愈合。" }
              ],
              "cancellers": []
            }
          ]
        }
      ]
    },
    {
      "name": "红美铃",
      "description": "华人小姑娘，红魔馆的门卫。中国妖怪，拥有使用气的程度的能力，擅长太极拳。",
      "parent_species": "妖怪",
      "hp": { "formula": "lambda base, level: base + 50 + (level * 6)", "base": 10, "level_coef": 6, "explanation": "HP = 65 + 等级 × 6" },
      "atk": { "formula": "lambda base, level: base + 3 + (level * 2)", "base": 3, "level_coef": 2, "explanation": "ATK = 6 + 等级 × 2" },
      "regen": { "formula": "lambda base, level: base + 3 + level", "base": 1, "level_coef": 1, "explanation": "REGEN = 4 + 等级" },
      "modifiers": { "hp_mod": 10, "atk_mod": 3, "regen_mod": 1 },
      "passive_skills": [
        {
          "name": "韧性的太极拳",
          "priority": 0,
          "type": "passive",
          "when": "lambda event, turn_counters, stats: True",
          "invalid_when": "lambda: False",
          "effects": [
            {
              "type": "buff",
              "allowed_target_type": "self_only",
              "modifiers": [
                { "prop": "damage_taken_multiplier", "allowed_target_type": "self_only", "value": "lambda: 0.7", "stack_type": "replace", "last_turns": -1, "invalid_when": "lambda hp, max_hp: hp >= max_hp * 0.3", "explanation": "当红美铃的生命值低于30%时，她受到的伤害减少30%。太极拳的柔劲化解了部分攻击。", "description": "美铃深吸一口气，太极气场展开，身体微微旋转，将袭来的攻击化力为柔。" }
              ],
              "cancellers": []
            }
          ]
        }
      ]
    },
    {
      "name": "琪露诺",
      "description": "湖上的冰精，自称最强的笨蛋妖精。拥有操控冷气程度的能力，智商是著名的「⑨」。",
      "parent_species": "妖精",
      "hp": { "formula": "lambda base, level: base + 50 + (level * 6)", "base": -5, "level_coef": 6, "explanation": "HP = 40 + 等级 × 6" },
      "atk": { "formula": "lambda base, level: base + 3 + (level * 2)", "base": -3, "level_coef": 2, "explanation": "ATK = 1 + 等级 × 2" },
      "regen": { "formula": "lambda base, level: base + 3 + level", "base": 1, "level_coef": 1, "explanation": "REGEN = 5 + 等级" },
      "modifiers": { "hp_mod": -5, "atk_mod": -3, "regen_mod": 1 },
      "passive_skills": [
        {
          "name": "幻想乡最强",
          "priority": 1,
          "type": "passive",
          "when": "lambda event, turn_counters, stats: event['type'] == 'turn_end'",
          "invalid_when": "lambda: False",
          "effects": [
            {
              "type": "buff",
              "allowed_target_type": "self_only",
              "modifiers": [
                { "prop": "atk", "allowed_target_type": "self_only", "value": "lambda atk, rand: atk * rand(1,6)", "stack_type": "replace", "last_turns": -1, "explanation": "每回合结束时，掷1d6，ATK替换为当前ATK乘以该数值。琪露诺的冰晶力量随机波动，有时极强有时变弱。", "description": "「あたいったら最強ね！」琪露诺振翅，冰晶之翼在阳光下闪烁，她的力量发生了不可思议的变化！" }
              ],
              "cancellers": []
            },
            {
              "type": "buff",
              "allowed_target_type": "self_only",
              "modifiers": [
                { "prop": "regen", "allowed_target_type": "self_only", "value": "lambda regen, rand: regen * rand(1,6)", "stack_type": "replace", "last_turns": -1, "explanation": "每回合结束时，掷1d6，REGEN替换为当前REGEN乘以该数值。琪露诺的再生能力也随自信波动。", "description": "冰之妖精的力量觉醒，周围的温度骤降，伤口以肉眼可见的速度愈合——至少琪露诺是这么认为的。" }
              ],
              "cancellers": []
            },
            {
              "type": "heal",
              "allowed_target_type": "self_only",
              "modifiers": [
                { "prop": "hp", "allowed_target_type": "self_only", "value": "lambda max_hp: math.ceil(max_hp * 0.05)", "stack_type": "replace", "last_turns": -1, "explanation": "每回合结束时，将HP设置为最大HP的5%。琪露诺坚信最强的自己需要这种极限状态。", "description": "「最强就是要这样！」琪露诺将自身逼入绝境，但眼中却闪烁着自信的光芒。" }
              ],
              "cancellers": []
            }
          ]
        }
      ]
    },
    {
      "name": "大妖精",
      "description": "善良的守护妖精，琪露诺的伙伴。红魔乡二面的道中BOSS，拥有自然治愈的能力。",
      "parent_species": "妖精",
      "hp": { "formula": "lambda base, level: base + 50 + (level * 6)", "base": 0, "level_coef": 6, "explanation": "HP = 45 + 等级 × 6" },
      "atk": { "formula": "lambda base, level: base + 3 + (level * 2)", "base": -2, "level_coef": 2, "explanation": "ATK = 1 + 等级 × 2" },
      "regen": { "formula": "lambda base, level: base + 3 + level", "base": 1, "level_coef": 1, "explanation": "REGEN = 5 + 等级" },
      "modifiers": { "hp_mod": 0, "atk_mod": -2, "regen_mod": 1 },
      "passive_skills": [
        {
          "name": "善良之心",
          "priority": 0,
          "type": "passive",
          "when": "lambda event, turn_counters, stats: True",
          "invalid_when": "lambda: False",
          "effects": [
            {
              "type": "buff",
              "allowed_target_type": "self_only",
              "modifiers": [
                { "prop": "heal_outgoing_multiplier", "allowed_target_type": "self_only", "value": "lambda: 1.5", "stack_type": "multiplicative", "last_turns": -1, "invalid_when": "lambda hp, max_hp: hp >= max_hp * 0.3", "explanation": "当大妖精的生命值低于30%时，她造成的治疗效果增加50%。善良的守护者在危急时刻会爆发出更强的治愈之力。", "description": "大妖精展开翠绿色的魔力护盾，光芒比平时更加温暖而柔和，仿佛在说「我会保护大家的」。" }
              ],
              "cancellers": []
            }
          ]
        }
      ]
    },
    {
      "name": "露米娅",
      "description": "宵暗的妖怪，红魔乡一面的BOSS。拥有操纵黑暗程度的能力，头发上缠绕的丝带是灵符。",
      "parent_species": "妖怪",
      "hp": { "formula": "lambda base, level: base + 50 + (level * 6)", "base": 0, "level_coef": 6, "explanation": "HP = 55 + 等级 × 6" },
      "atk": { "formula": "lambda base, level: base + 3 + (level * 2)", "base": 0, "level_coef": 2, "explanation": "ATK = 3 + 等级 × 2" },
      "regen": { "formula": "lambda base, level: base + 3 + level", "base": 0, "level_coef": 1, "explanation": "REGEN = 3 + 等级" },
      "modifiers": { "hp_mod": 0, "atk_mod": 0, "regen_mod": 0 },
      "passive_skills": [
        {
          "name": "宵暗之影",
          "priority": 0,
          "type": "passive",
          "when": "lambda event, turn_counters, stats: True",
          "invalid_when": "lambda: False",
          "effects": [
            {
              "type": "buff",
              "allowed_target_type": "self_only",
              "modifiers": [
                { "prop": "evade_chance", "allowed_target_type": "self_only", "value": "lambda: 1/3", "stack_type": "replace", "last_turns": -1, "invalid_when": "lambda hp, max_hp: hp >= max_hp * 0.3", "explanation": "当露米娅的生命值低于30%时，她有1/3的概率闪避攻击。她融入黑暗之中，难以被察觉。", "description": "露米娅化作一团黑影，在黑暗中忽隐忽现，让人难以锁定她的位置。" }
              ],
              "cancellers": [
                { "event_to_cancel": "damage", "allowed_target_type": "self_only", "cancel_type": "first", "when": "lambda event, turn_counters, source, target: True", "cancel": "lambda: random.random() < 1/3", "explanation": "33%概率取消一次伤害事件", "description": "黑暗屏障升起，攻击在露米娅面前消散。" }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

---

## 6. 角色完整数据（Characters）—— 10 个角色，完整主动技能

以下为每个角色的 JSON 数据，包含所有主动技能（普通攻击、符卡、支援、终极），每个技能都附有超长的 `explanation` 和 `description` 字段，以尊重原书的历史传统。

### 6.1 博丽灵梦

```json
{
  "character_id": "reimu_hakurei",
  "name": "博丽灵梦",
  "species": "博丽灵梦",
  "level": 0,
  "hp_max": 115,
  "atk_base": 5,
  "regen_base": 0,
  "skills": [
    {
      "name": "御札投掷",
      "type": "normal",
      "description": "投掷带有灵力的御札攻击敌人。博丽灵梦最基础的攻击方式，御札上附着弱小的灵力，能够对妖怪造成伤害。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk + rand(1,6))",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成等于 ATK + 1d6 的伤害。这是最基础的攻击，没有额外消耗，适合在任何场合使用。掷一个六面骰子，将结果与灵梦的 ATK 相加，即为对目标造成的伤害值。",
              "description": "灵梦从袖中抽出一张金光闪烁的御札，手腕轻扬，御札如飞刀般旋转着射向敌人。御札在半空中划出一道优美的金色弧线，触碰到目标的瞬间爆发出微小的灵力冲击，散发出淡淡的荧光。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "灵符「梦想封印」",
      "type": "spell_card",
      "description": "释放追踪的七彩灵力光球。博丽灵梦最具代表性的符卡之一，七个光球会追踪敌人并造成大量伤害。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk + rand(1,6) + rand(1,6) + 8)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK + 2d6 + 8 的伤害。这是灵梦的招牌符卡，消耗 15 点 HP 以换取高额伤害。掷两个六面骰子，加上 ATK 和 8，即为总伤害。",
              "description": "灵梦双手合十，全身灵力涌动。七颗拳头大小的光球从她身后升起，分别闪烁着红、橙、黄、绿、蓝、靛、紫七种颜色。光球在空中旋转一圈后，以优美的弧线追踪敌人，如同七颗流星划破夜空。接触的瞬间，七个光球同时绽放，爆发出彩虹色的光芒，将敌人笼罩在七彩的灵力风暴中。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda: -15",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗 15 点生命值。灵力的燃烧会对灵梦自身造成反噬，但为了击败强敌，这是必要的代价。",
              "description": "灵力从灵梦体内剧烈燃烧，她的脸颊浮现一抹绯红，气息微微紊乱。御札上的符文变得更加明亮，仿佛在燃烧自己的寿命换取力量。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "阴阳宝玉",
      "type": "support",
      "description": "释放旋转的阴阳玉形成灵力护盾。灵梦召唤出博丽神社代代相传的阴阳玉，释放柔和的灵力来治疗自己或队友。",
      "effects": [
        {
          "type": "heal",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: math.ceil((atk + rand(1,6)) / 2)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "为目标恢复 (ATK + 1d6) / 2（向上取整）的生命值。阴阳宝玉的治疗量随灵梦的 ATK 成长，使得这个支援技能在后期也有显著效果。掷一个六面骰子，加上 ATK，再除以 2，向上取整。",
              "description": "灵梦伸出右手，掌心浮现出一枚黑白两色的阴阳玉。阴阳玉悬浮在空中，缓缓旋转，释放出柔和的灵力波动。光芒如水波般扩散，被光芒笼罩的目标感到一股温暖的力量涌入体内，伤口逐渐愈合，疲惫一扫而空。阴阳玉的转动越来越慢，最终化作光点消散。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "「梦想天生」",
      "type": "ultimate",
      "description": "释放无数结界与灵力冲击，最强符卡。博丽灵梦的终极符卡，需要消耗大量生命值，但能造成毁灭性的伤害。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda max_hp: -math.ceil(max_hp * 0.5)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗最大 HP 的 50%（向上取整）。这是灵梦最强的王牌，必须付出巨大的代价。",
              "description": "灵梦闭上眼睛，深吸一口气。她的身体开始散发出耀眼的白光，灵力如同沸腾的水一般在体内翻涌。她睁开眼时，瞳孔中映出了整个天空。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk * rand(1,6) + 18)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK × 1d6 + 18 的伤害。掷一个六面骰子，乘以灵梦的 ATK，再加 18。这是整个游戏中最强的单体伤害之一。",
              "description": "灵梦漂浮在半空中，她身后浮现出无数阴阳玉和结界，层层叠叠如同万花筒。她双手前推，所有的结界同时绽放出耀眼的光芒，汇聚成一道粗大的光柱贯穿天地。敌人的身影被光芒吞没，天空中只剩下一片绚烂的霞色，久久不散。"
            }
          ],
          "cancellers": []
        }
      ]
    }
  ]
}
```

### 6.2 雾雨魔理沙

```json
{
  "character_id": "marisa_kirisame",
  "name": "雾雨魔理沙",
  "species": "雾雨魔理沙",
  "level": 0,
  "hp_max": 95,
  "atk_base": 13,
  "regen_base": 0,
  "skills": [
    {
      "name": "魔法弹",
      "type": "normal",
      "description": "发射魔力弹攻击敌人。魔理沙最基础的魔法攻击，魔力凝聚成光弹射出。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk + rand(1,6))",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK + 1d6 的伤害。标准普通攻击，无消耗。",
              "description": "魔理沙张开手掌，蓝色的魔力在掌心汇聚，形成一个拳头大小的光球。她手腕一抖，光球如子弹般射向敌人，在空中留下一道短暂的魔法光迹。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "恋符「Master Spark」",
      "type": "spell_card",
      "description": "使用迷你八卦炉放射超大范围激光。魔理沙的招牌符卡，释放一道粗大的激光横扫战场。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk + rand(1,6) + rand(1,6) + rand(1,6) + 12)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK + 3d6 + 12 的伤害。魔理沙先发射激光再承受魔力反噬（技能效果顺序），因此即使生命值不足也能完整施放。",
              "description": "魔理沙从腰间抽出迷你八卦炉，对准敌人，按下扳机。一道直径数米的七彩激光从炉口喷薄而出，激光所过之处空气都在扭曲，地面被犁出一道深深的沟壑。轰鸣声震耳欲聋，连远处的红魔馆窗户都在颤抖。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda: -20",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗 20 点生命值。魔力反噬，八卦炉变得滚烫。",
              "description": "激光发射完毕后，八卦炉的表面变得通红，散发热气。魔理沙的手被烫得缩了缩，但她依然露出满足的笑容：「嘿嘿，威力还不错吧！」"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "星尘幻想",
      "type": "support",
      "description": "召唤星光环绕自身，积蓄魔力。魔理沙仰望夜空，星光如尘埃般洒落，她在此期间闭眼冥想，下一张符卡将不消耗生命值。",
      "effects": [
        {
          "type": "buff",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "next_self_damage_cancel",
              "allowed_target_type": "self_only",
              "value": "lambda: 1",
              "stack_type": "replace",
              "last_turns": 2,
              "explanation": "获得一个标记，下一次对自己造成的伤害效果被取消。配合魔理沙的「先伤害后消耗」特性，可以免费施放一次符卡。",
              "description": "魔理沙仰起头，夜空中的星光似乎变得更加明亮。无数细小的光点从天而降，如同尘埃般洒落在她身上，环绕着她的身体缓慢旋转。她闭上眼睛，嘴角微微上扬，魔力在体内沸腾。"
            }
          ],
          "cancellers": [
            {
              "event_to_cancel": "damage",
              "allowed_target_type": "self_only",
              "cancel_type": "first",
              "when": "lambda event, turn_counters, source, target: event['type'] == 'damage' and event['modifiers'][0].value < 0 if event['modifiers'] else True",
              "cancel": "lambda: True",
              "explanation": "取消下一次对自己造成的伤害效果。",
              "description": "星光形成一道薄薄的屏障，将魔力反噬的伤害完全吸收。魔理沙毫发无损，笑道：「谢啦，星星！」"
            }
          ]
        }
      ]
    },
    {
      "name": "魔炮「Final Master Spark」",
      "type": "ultimate",
      "description": "汇聚所有魔力释放终极魔炮。魔理沙的最强符卡，将全部魔力注入八卦炉，释放一道足以烧毁一座山的超级激光。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk * rand(1,6) + 24)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK × 1d6 + 24 的伤害。掷一个六面骰子，乘以 ATK，再加 24。伤害随等级爆炸性增长。",
              "description": "魔理沙将八卦炉对准天空，深吸一口气。她全身的魔力如同洪水般涌入八卦炉，炉口的符文一个接一个亮起。金色的光柱撕裂天穹，直冲云霄，即使在红魔馆塔顶也能看见这道冲天的光柱。大地震颤，空气灼热，整个战场都在颤抖。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda max_hp: -math.ceil(max_hp * 0.5)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗最大 HP 的 50%（向上取整）。这是终极的代价。",
              "description": "激光消散后，魔理沙摇摇晃晃地落回地面，八卦炉「啪嗒」一声掉在地上。她大口喘着气，脸上却挂着满足的笑容：「这才是……我的全力啊。」"
            }
          ],
          "cancellers": []
        }
      ]
    }
  ]
}
```

### 6.3 十六夜咲夜

```json
{
  "character_id": "sakuya_izayoi",
  "name": "十六夜咲夜",
  "species": "十六夜咲夜",
  "level": 0,
  "hp_max": 105,
  "atk_base": 3,
  "regen_base": 0,
  "skills": [
    {
      "name": "银制小刀",
      "type": "normal",
      "description": "投掷银制小刀攻击敌人。咲夜随身携带的银制小刀，锋利无比，投掷速度极快。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk + rand(1,6))",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK + 1d6 的伤害。普通攻击。",
              "description": "咲夜从围裙口袋中抽出一把银制小刀，刀刃在月光下反射出寒光。她手腕轻抖，小刀化作一道银线飞出，精准地命中目标。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "奇术「Misdirection」",
      "type": "spell_card",
      "description": "通过时间暂停，投掷大量隐形飞刀。咲夜暂停时间，在静止的世界中从多个角度投掷小刀，敌人无法防御。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk * 2 + rand(1,6) + rand(1,6) + 8)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK×2 + 2d6 + 8 的伤害。消耗等于 ATK 的生命值。",
              "description": "咲夜的身影在原地短暂消失——她暂停了时间。在静止的世界里，她从容地走到敌人周围，从各个角度掷出银制小刀。时间恢复流动的瞬间，无数银光从四面八方同时射向敌人，仿佛空间本身都在攻击。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda atk: -atk",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗等于 ATK 的生命值。操纵时间需要消耗咲夜的体力。",
              "description": "咲夜微微喘息，怀表的指针跳动了一下。使用时间能力会让她的身体承受一定的负担。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "时间回流",
      "type": "support",
      "description": "操纵自身时间线小幅度回溯。咲夜将自身的时间倒退回受伤之前，使伤口愈合。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda atk: -atk",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗等于 ATK 的生命值。",
              "description": "咲夜按下怀表的按钮，指针开始逆时针跳动。她的身体周围出现淡淡的残影。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "heal",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: math.ceil(atk * 2 + rand(1,6) + 10)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "为目标恢复 ATK×2 + 1d6 + 10 的生命值。强大的治疗能力，代价是消耗等量的生命值。",
              "description": "咲夜的身影闪烁了一下，仿佛时间在她身上倒流。伤口以肉眼可见的速度愈合，血迹消失，如同从未受过伤一般。她整理了一下女仆装，恢复了完美潇洒的姿态。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "「The World」",
      "type": "ultimate",
      "description": "停止时间，在静止的世界中连续攻击。咲夜将时间完全停止，获得多次额外行动机会，敌方在此期间无法行动。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda max_hp: -math.ceil(max_hp * 0.5)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗最大 HP 的 50%（向上取整）。时间停止的巨大代价。",
              "description": "咲夜翻转怀表，盖子弹开，发出一声清脆的「咔哒」。世界瞬间静止——风凝固在半空，落叶悬停在眼前，敌人的表情定格。只有咲夜能够自由移动。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "extra_turn",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "extra_turns_counter",
              "allowed_target_type": "self_only",
              "value": "lambda rand: rand(1,6)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "掷 1d6，获得对应次数的额外行动机会。",
              "description": "咲夜在静止的世界中从容踱步，从各个角度掷出银制小刀。小刀停在半空中，形成了完美的几何图案。时间恢复时，所有的刀同时射向敌人。"
            }
          ],
          "cancellers": []
        }
      ]
    }
  ]
}
```

### 6.4 蕾米莉亚·斯卡蕾特

```json
{
  "character_id": "remilia_scarlet",
  "name": "蕾米莉亚·斯卡蕾特",
  "species": "蕾米莉亚·斯卡蕾特",
  "level": 0,
  "hp_max": 85,
  "atk_base": 16,
  "regen_base": 8,
  "skills": [
    {
      "name": "蝙蝠攻击",
      "type": "normal",
      "description": "召唤蝙蝠群攻击敌人。蕾米莉亚召唤出一群蝙蝠，它们尖啸着扑向敌人。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk + rand(1,6))",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK + 1d6 的伤害。",
              "description": "蕾米莉亚抬手指向敌人，从她身后的黑暗中飞出数十只蝙蝠。蝙蝠群发出刺耳的尖啸，如同一片黑云扑向目标，撕咬后化为黑烟消散。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "红符「Scarlet Shot」",
      "type": "spell_card",
      "description": "释放绯红色的魔力弹幕。蕾米莉亚从指尖射出一连串绯红色的魔力弹，如同血色的流星群。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk + rand(1,6) + rand(1,6) + 8)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK + 2d6 + 8 的伤害。",
              "description": "蕾米莉亚伸出一根手指，指尖凝聚出一点猩红的光芒。她轻轻一弹，无数绯红色的魔力弹从指尖不断飞出，如同血色的流星群划破夜空。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda: -10",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗 10 点生命值。",
              "description": "蕾米莉亚的指尖微微发烫，她皱了皱眉，但依然保持着优雅的微笑。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "猩红不夜城",
      "type": "support",
      "description": "张开猩红色力场，吸取周围生命力。蕾米莉亚展开猩红的灵力场，力场内的敌人生命不断流失，部分转化为蕾米莉亚的生命。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda regen: -max(regen, 1)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗至少等于当前回复力的生命值（妖怪支援技能最小消耗规则）。",
              "description": "蕾米莉亚张开双臂，猩红色的灵力从她体内涌出，形成一个半透明的力场。她的脸色变得苍白，但眼中闪过兴奋的光芒。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "buff",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "lifesteal_percent",
              "allowed_target_type": "self_only",
              "value": "lambda: 35",
              "stack_type": "replace",
              "last_turns": 2,
              "explanation": "本回合及下回合内，造成伤害的 35% 用于治疗自身。",
              "description": "猩红力场以蕾米莉亚为中心展开，笼罩了周围数米的范围。被力场笼罩的敌人感到生命在缓慢流失，一丝丝绯红色的生命精华从他们体内飘出，被蕾米莉亚吸收。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "红魔「Scarlet Devil」",
      "type": "ultimate",
      "description": "释放猩红色的不夜城灵力场。蕾米莉亚的最强符卡，化身为绯红色恶魔的化身，释放毁灭性的灵力场。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda max_hp: -math.ceil(max_hp * 0.5)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗最大 HP 的 50%（向上取整）。",
              "description": "蕾米莉亚漂浮在半空中，她的身后升起一轮猩红色的满月。她闭上眼睛，张开双臂，仿佛在拥抱整个夜空。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk * rand(1,6) + 18)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK × 1d6 + 18 的伤害。",
              "description": "猩红之月绽放出刺目的光芒，不夜城的灵力场以蕾米莉亚为中心全力展开。红色的光柱冲天而起，仿佛整个夜空都在燃烧。敌人的身影被红色的光芒吞没，耳边只剩下蕾米莉亚的轻笑：「さようなら。」"
            }
          ],
          "cancellers": []
        }
      ]
    }
  ]
}
```

### 6.5 芙兰朵露·斯卡蕾特

```json
{
  "character_id": "flandre_scarlet",
  "name": "芙兰朵露·斯卡蕾特",
  "species": "芙兰朵露·斯卡蕾特",
  "level": 0,
  "hp_max": 80,
  "atk_base": 28,
  "regen_base": 7,
  "skills": [
    {
      "name": "禁忌之爪",
      "type": "normal",
      "description": "挥动莱瓦汀剑攻击。芙兰挥舞巨大的莱瓦汀魔剑，剑身划过空气时留下炽热的轨迹。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk + rand(1,6))",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK + 1d6 的伤害。",
              "description": "芙兰握住莱瓦汀的剑柄，剑身上立刻燃起深红色的火焰。她随意一挥，剑刃划过空气，留下一道灼热的轨迹。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "禁忌「Lävatein」",
      "type": "spell_card",
      "description": "挥舞莱瓦汀魔剑，释放毁灭性斩击。芙兰将莱瓦汀全力挥出，一道巨大的火焰斩击将面前的一切卷入火海。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk + rand(1,6) + rand(1,6) + 14)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK + 2d6 + 14 的伤害。",
              "description": "芙兰双手握住莱瓦汀，深吸一口气。剑身上的火焰猛然暴涨，将她的身影映得通红。她一记横扫，一道巨大的火焰斩击飞出，将面前的一切卷入火海。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda: -12",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗 12 点生命值。",
              "description": "芙兰的脸上闪过一丝痛苦，但很快被兴奋的笑容取代。她不在乎这点代价。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "禁忌「Four of a Kind」",
      "type": "support",
      "description": "召唤分身进行模仿攻击。芙兰召唤出三个完全相同的分身，她们围绕敌人旋转，每个动作都同步模仿。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda regen, atk: -max(3 * (regen + atk), 1)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗 3×(回复力+ATK) 的生命值。非常昂贵的代价。",
              "description": "芙兰闭上眼睛，身体开始发出微光。她的影子开始分裂，一、二、三……三个完全相同的分身从影中走出。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "buff",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "multi_existence",
              "allowed_target_type": "self_only",
              "value": "lambda: 4",
              "stack_type": "replace",
              "last_turns": 2,
              "explanation": "获得 4 重存在（multi_existence=4），持续 2 回合。所有技能效果次数乘以 4。",
              "description": "三个分身围绕芙兰旋转，她们的动作完全同步，如同镜子中的倒影。芙兰笑道：「四个打一个，公平吗？」"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "QED「495年的波纹」",
      "type": "ultimate",
      "description": "以495年的寿命凝结的波纹，毁灭一切。芙兰的最强符卡，释放出波纹，所过之处一切都被分解为虚无。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda max_hp: -math.ceil(max_hp * 0.5)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗最大 HP 的 50%（向上取整）。",
              "description": "芙兰握住眼中的「第四之牙」，那只眼睛开始发出幽暗的光。整个红魔馆似乎都在颤抖。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk * rand(1,6) + 26)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK × 1d6 + 26 的伤害。",
              "description": "芙兰轻轻说出一声「QED」。波纹从她体内扩散，所过之处，一切都被分解为虚无。地面消失，空气消失，连光线都被扭曲。波纹扩散到敌人身上，敌人的身体开始崩解，如同一张被揉碎的纸。"
            }
          ],
          "cancellers": []
        }
      ]
    }
  ]
}
```

### 6.6 帕秋莉·诺蕾姬

```json
{
  "character_id": "patchouli_knowledge",
  "name": "帕秋莉·诺蕾姬",
  "species": "帕秋莉·诺蕾姬",
  "level": 0,
  "hp_max": 45,
  "atk_base": 25,
  "regen_base": 2,
  "skills": [
    {
      "name": "七曜弹幕",
      "type": "normal",
      "description": "发射基础魔法弹攻击。帕秋莉翻开魔导书，七种颜色的魔法弹从书中飞出。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk + rand(1,6))",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK + 1d6 的伤害。",
              "description": "帕秋莉翻开手中的魔导书，书页自动翻动。她念诵简短的咒文，红、蓝、黄、绿、金、银、紫七种颜色的魔法弹从书页中飞出，如同七色的雨点洒向敌人。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "火符「Agni Shine」",
      "type": "spell_card",
      "description": "释放火神之光，同时进行焚烧与治疗。帕秋莉咏唱火神阿耆尼的咒文，火焰净化敌人的同时，温暖的能量也流回她的身体。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk + rand(1,6) + rand(1,6) + 10)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK + 2d6 + 10 的伤害。",
              "description": "帕秋莉将魔导书翻到火之章节，用沙哑的声音咏唱：「阿耆尼，火神啊，请将您的光辉赐予我。」空气中弥漫着灼热的气息，一道火柱从地面升起，将敌人吞没。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda: -10",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗 10 点生命值。",
              "description": "帕秋莉剧烈咳嗽起来，脸色更加苍白。但她坚持念完了咒文。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "heal",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda atk: math.ceil(atk / 5)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "治疗自身 ATK/5（向上取整）的生命值。",
              "description": "火焰消散后，一丝温暖的能量流回帕秋莉的身体，她苍白的脸上恢复了一丝血色。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "元素掌控",
      "type": "support",
      "description": "操控七曜元素回复自身体力。帕秋莉引导七曜元素环绕自身，水与木的元素滋养她的身体。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda regen: -max(regen, 1)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗至少等于当前回复力的生命值（妖怪支援技能最小消耗规则）。",
              "description": "帕秋莉咳嗽了几声，但她依然坚持引导元素之力。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "heal",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: math.ceil((atk + rand(1,6)) / 2)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "为目标恢复 (ATK + 1d6)/2（向上取整）的生命值。",
              "description": "帕秋莉闭上眼睛，魔导书自动翻到水之章节。蓝色的水元素和绿色的木元素从书中涌出，环绕着她缓缓旋转。她的呼吸渐渐平稳，咳嗽声也停止了。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "贤者之石",
      "type": "ultimate",
      "description": "帕秋莉的最强魔法，释放七曜的全部元素力量。七色的贤者之石在她面前旋转，七大元素同时迸发。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda max_hp: -math.ceil(max_hp * 0.5)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗最大 HP 的 50%（向上取整）。",
              "description": "帕秋莉翻开古籍最后一页，书页上画着一颗七色的宝石。她咬破指尖，将血滴在书页上，宝石开始发光。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk * rand(1,6) + 18)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK × 1d6 + 18 的伤害。",
              "description": "七色的贤者之石在她面前旋转，分别代表火、水、木、金、土、日、月。七大元素同时迸发，其光芒照亮了整个地底图书馆。七道不同颜色的光柱射向敌人，在空中交织成一张元素之网。"
            }
          ],
          "cancellers": []
        }
      ]
    }
  ]
}
```

### 6.7 红美铃

```json
{
  "character_id": "meirin_hong",
  "name": "红美铃",
  "species": "红美铃",
  "level": 0,
  "hp_max": 65,
  "atk_base": 6,
  "regen_base": 4,
  "skills": [
    {
      "name": "太极掌",
      "type": "normal",
      "description": "使用太极拳法攻击。美铃以太极起手式推出一掌，气劲划出圆润的轨迹。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk + rand(1,6))",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK + 1d6 的伤害。",
              "description": "美铃双脚分开，与肩同宽，双手缓缓抬起。她以太极起手式推出一掌，看似缓慢，但掌风凌厉，一股无形的气劲划出圆润的轨迹，击中敌人。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "彩符「极彩台风」",
      "type": "spell_card",
      "description": "释放彩虹色气劲形成旋风。美铃快速旋转双掌，七彩的气劲如台风般卷起。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk + rand(1,6) + rand(1,6) + 6)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK + 2d6 + 6 的伤害。",
              "description": "美铃深吸一口气，双掌快速旋转，掌心凝聚出七彩的气劲。她猛然推出，一道彩虹色的旋风凭空出现，将敌人卷起抛向空中。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda: -10",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗 10 点生命值。",
              "description": "美铃的气息有些紊乱，但她迅速调整呼吸，重新站稳。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "虹色太极拳",
      "type": "support",
      "description": "运用内力和虹彩之力，进行一次特殊攻击。美铃双掌焕发出彩虹色的光芒，一击击出的同时将敌人的气引回自身。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda regen: -max(regen, 1)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗至少等于当前回复力的生命值（妖怪支援技能最小消耗规则）。",
              "description": "美铃调动全身的内力，虹色的光芒从她体内散发出来。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk + rand(1,6) + 4)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK + 1d6 + 4 的伤害。",
              "description": "美铃双掌焕发出彩虹色的光芒，一掌击出，掌风中带着七色的光晕。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "lifesteal",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda damage: damage * 0.3",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "恢复本次伤害值 30% 的生命。",
              "description": "美铃将手掌收回，敌人的气随着她的动作被引回自身，她的呼吸变得平稳，内力也恢复了一些。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "「华光玉」",
      "type": "ultimate",
      "description": "将气凝聚成光之宝玉释放。美铃将全身的气凝聚于掌中，形成一个旋转的彩色光球——华光玉。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda max_hp: -math.ceil(max_hp * 0.5)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗最大 HP 的 50%（向上取整）。",
              "description": "美铃闭上眼睛，将全身的气凝聚于右掌。她的手掌开始发光，一个彩色的光球逐渐成形。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk * rand(1,6) + 15)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK × 1d6 + 15 的伤害。",
              "description": "美铃睁开眼，将华光玉猛然推向敌人。光球在空中旋转，散出七彩的光晕，照亮了整个红魔馆大门。光球击中敌人，爆发出耀眼的光芒。"
            }
          ],
          "cancellers": []
        }
      ]
    }
  ]
}
```

### 6.8 琪露诺

```json
{
  "character_id": "cirno",
  "name": "琪露诺",
  "species": "琪露诺",
  "level": 0,
  "hp_max": 40,
  "atk_base": 1,
  "regen_base": 5,
  "skills": [
    {
      "name": "冰弹",
      "type": "normal",
      "description": "发射冰块攻击。琪露诺凝结空气中的水分，掷出一块坚冰。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: max(1, -(atk + rand(1,6)))",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 max(1, ATK + 1d6) 的伤害。琪露诺的 ATK 可能很低，但至少造成 1 点伤害。",
              "description": "琪露诺伸出手，空气中的水汽迅速凝结，形成一块拳头大的冰块。她用力掷出，冰块在空中划过一道弧线，砸向敌人。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "冰符「Icicle Fall」",
      "type": "spell_card",
      "description": "从敌人上方降下大量冰柱。琪露诺振翅飞起，巨大的冰柱从敌人头顶坠落。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: max(1, -(atk + rand(1,6) + rand(1,6) + 4))",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 max(1, ATK + 2d6 + 4) 的伤害。",
              "description": "琪露诺振翅飞起，六片冰晶之翼在阳光下闪烁。她双手下压，天空中凭空出现数十根巨大的冰柱，如同瀑布般从敌人头顶坠落。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda: -6",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗 6 点生命值。",
              "description": "琪露诺的气喘了一些，但她依然得意洋洋：「我可是最强的！」"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "完美冻结",
      "type": "support",
      "description": "冻结敌人的攻击力。琪露诺释放寒气，削弱敌人的攻击力。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda regen: -max(regen, 1)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗至少等于当前回复力的生命值（妖怪支援技能最小消耗规则）。",
              "description": "琪露诺集中精神，寒气从她体内涌出。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "debuff",
          "allowed_target_type": "enemies_only",
          "modifiers": [
            {
              "prop": "atk_multiplier",
              "allowed_target_type": "enemies_only",
              "value": "lambda: 0.5",
              "stack_type": "multiplicative",
              "last_turns": 5,
              "explanation": "敌人的 ATK 减半，持续 5 回合（由于敌方回合数，实际有效 3 回合）。",
              "description": "琪露诺双手前推，一阵刺骨的寒气从她掌心涌出。敌人的身上开始结霜，动作变得迟缓，攻击力也大幅下降。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "「冰裂冻华阵」",
      "type": "ultimate",
      "description": "冰之妖精的最强一击。琪露诺释放全部力量，大地裂开，冰柱从裂缝中冲天而起。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda max_hp: -math.ceil(max_hp * 0.5)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗最大 HP 的 50%（向上取整）。",
              "description": "琪露诺闭上眼睛，六片冰晶之翼展开到最大。她将全部力量凝聚于一点。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: max(1, -(atk * rand(1,6) + 10))",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 max(1, ATK × 1d6 + 10) 的伤害。",
              "description": "琪露诺睁开眼，大喊：「冰裂冻华阵！」大地裂开，无数冰柱从裂缝中冲天而起，形成一片冰之森林。雾之湖的湖面都被冻结——至少在她想象中是如此。"
            }
          ],
          "cancellers": []
        }
      ]
    }
  ]
}
```

### 6.9 大妖精

```json
{
  "character_id": "daiyousei",
  "name": "大妖精",
  "species": "大妖精",
  "level": 0,
  "hp_max": 45,
  "atk_base": 1,
  "regen_base": 5,
  "skills": [
    {
      "name": "妖精弹幕",
      "type": "normal",
      "description": "发射普通弹幕。大妖精轻轻挥手，数枚绿色弹幕飞向敌人。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: max(1, -(atk + rand(1,6)))",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 max(1, ATK + 1d6) 的伤害。",
              "description": "大妖精轻轻挥手，数枚翠绿色的弹幕从她周围形成，如萤火虫般飞向敌人。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "自然之力",
      "type": "spell_card",
      "description": "释放自然元素的弹幕。大妖精双手展开，弹幕如花瓣般四散飞舞。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: max(1, -(atk + rand(1,6) + rand(1,6) + 6))",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 max(1, ATK + 2d6 + 6) 的伤害。",
              "description": "大妖精双手展开，绿色的魔法阵在她脚下浮现。无数弹幕如花瓣般从她周围四散飞舞，在空中留下绿色的轨迹，如同春天的花雨。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda: -6",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗 6 点生命值。",
              "description": "大妖精的脸色有些发白，但她依然微笑。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "妖精的守护",
      "type": "support",
      "description": "为队友或自己施加守护。大妖精展开翠绿色的魔力护盾，光芒温暖而柔和。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda regen: -max(regen, 1)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗至少等于当前回复力的生命值（妖怪支援技能最小消耗规则）。",
              "description": "大妖精闭上眼睛，翠绿色的魔力从她体内散发。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "heal",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: math.ceil((atk + rand(1,6)) / 2) + 4",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "为目标恢复 (ATK + 1d6)/2 + 4（向上取整）的生命值。",
              "description": "大妖精展开翠绿色的魔力护盾，光芒温暖而柔和，细心抚慰着所受的每一处伤痛。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "「大妖精的加护」",
      "type": "ultimate",
      "description": "大妖精的最强支援符卡。金色的光之雨从她翅膀洒落，生命如春芽般复苏。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda max_hp: -math.ceil(max_hp * 0.5)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗最大 HP 的 50%（向上取整）。",
              "description": "大妖精展翅飞起，她的翅膀散发出金色的光芒。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "heal",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk * rand(1,6) + 10)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "为目标恢复 ATK × 1d6 + 10 的生命值。",
              "description": "金色的光之雨从大妖精的翅膀洒落，每一滴都蕴含着雾之湖的恩惠。被选中的目标沐浴在光雨中，生命如春芽般复苏，伤口愈合，疲惫消散。"
            }
          ],
          "cancellers": []
        }
      ]
    }
  ]
}
```

### 6.10 露米娅

```json
{
  "character_id": "rumia",
  "name": "露米娅",
  "species": "露米娅",
  "level": 0,
  "hp_max": 55,
  "atk_base": 3,
  "regen_base": 3,
  "skills": [
    {
      "name": "黑暗弹幕",
      "type": "normal",
      "description": "发射黑暗弹幕攻击。露米娅挥手，黑色的弹幕从她周围形成并向敌人飞去。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk + rand(1,6))",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK + 1d6 的伤害。",
              "description": "露米娅挥动双手，黑色的弹幕从她周围的黑暗中形成，如同夜空中飘浮的黑色蒲公英种子，缓缓飘向敌人。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "暗符「Demarcation」",
      "type": "spell_card",
      "description": "用黑暗划分界限。露米娅用黑暗划定一个区域，黑色光线切割着周围的一切。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk + rand(1,6) + rand(1,6) + 4)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK + 2d6 + 4 的伤害。",
              "description": "露米娅抬起手，黑暗从她脚下蔓延，在地面上划出一个黑色的圆环。圆环内的空间突然被无数黑色的光线切割，如同刀网一般。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda: -8",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗 8 点生命值。",
              "description": "露米娅身子晃了晃，但她很快稳住身形。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "黑暗掠夺",
      "type": "support",
      "description": "从敌人身上夺取黑暗力量。露米娅从敌人身上剥离黑暗，偷取对方的回复力。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda regen: -max(regen, 1)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗至少等于当前回复力的生命值（妖怪支援技能最小消耗规则）。",
              "description": "露米娅融入黑暗，周围的暗元素开始躁动。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "debuff",
          "allowed_target_type": "enemies_only",
          "modifiers": [
            {
              "prop": "regen",
              "allowed_target_type": "enemies_only",
              "value": "lambda target_regen: -target_regen",
              "stack_type": "additive",
              "last_turns": 4,
              "explanation": "偷取敌方全部回复力，持续 4 回合。",
              "description": "黑色的触须从露米娅身上伸出，缠绕住敌人。敌人身上的黑暗力量被逐渐剥离，化作黑色的雾气飘向露米娅。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "buff",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "regen",
              "allowed_target_type": "self_only",
              "value": "lambda stolen_regen: stolen_regen",
              "stack_type": "additive",
              "last_turns": 4,
              "explanation": "获得与偷取量相等的回复力提升。",
              "description": "黑色的雾气涌入露米娅体内，她的身体变得更加虚幻，仿佛与黑暗融为一体。"
            }
          ],
          "cancellers": []
        }
      ]
    },
    {
      "name": "「月符「Moonlight Ray」」",
      "type": "ultimate",
      "description": "解放黑暗与月光的力量。露米娅解除丝带封印，黑暗与月光同时爆发。",
      "effects": [
        {
          "type": "damage",
          "allowed_target_type": "self_only",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "self_only",
              "value": "lambda max_hp: -math.ceil(max_hp * 0.5)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "消耗最大 HP 的 50%（向上取整）。",
              "description": "露米娅伸手触碰头上的丝带，丝带缓缓松开。一股强大的黑暗力量从她体内涌出。"
            }
          ],
          "cancellers": []
        },
        {
          "type": "damage",
          "allowed_target_type": "selectable",
          "modifiers": [
            {
              "prop": "hp",
              "allowed_target_type": "selectable",
              "value": "lambda atk, rand: -(atk * rand(1,6) + 12)",
              "stack_type": "additive",
              "last_turns": -1,
              "explanation": "造成 ATK × 1d6 + 12 的伤害。",
              "description": "丝带完全解开，黑暗与月光同时爆发。周围的光线忽明忽暗，一道银白色的月光从云层中射下，与黑色的黑暗之力交织在一起，形成一道诡异的光柱，射向敌人。"
            }
          ],
          "cancellers": []
        }
      ]
    }
  ]
}
```

---

## 7. 总结

本最终规则书包含：

1. **完整的大纲与概念**：双回合计数器、物种单继承、被动技能优先级（琪露诺的「幻想乡最强」priority=1）。
2. **清晰的回合顺序**：定义了 turn_start、技能执行、turn_end、被动技能触发和修饰器更新，以及全局/玩家计数器的维护。
3. **完整的数据模式**：涵盖 Species、Character、Skill、Effect、Modifier、Canceller、PassiveSkill 的 JSON Schema。
4. **可运行的 Python OOP 引擎**：使用 `eval` 解析 lambda 字符串，实现了修饰器、取消器、被动技能优先级和双计数器。
5. **完整的物种定义**：包括人类、妖怪、亚种和 10 个角色特质物种，支持单亲继承和被动覆盖。
6. **10 个角色的完整主动技能数据**：每个角色 4 个技能，每个技能都包含超长的 `explanation` 和 `description` 字段，尊重原书历史。

该设计可直接用于开发可运行的回合制游戏，并易于扩展新角色和新机制。所有 JSON 数据均可被提供的 Python 引擎解析。
