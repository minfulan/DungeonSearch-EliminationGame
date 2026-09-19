# 文本RPG游戏设计文档

## 1. 项目概述

一款运行在网页端的**纯单人 PVE 文本 RPG**，融合以下核心要素：

- **大量文本剧情**：分支对话与环境叙事，文本预先写好。
- **NetHack 式随机地宫**：10 层地牢，每轮重新生成，越深敌人越强。
- **经典 Roguelike 回合制**：玩家行动 → 怪物行动 → 环境结算。
- **搜打撤玩法**：探索、战斗、搜索容器，在撤离点带出物资。
- **永久养成**：技能与属性死亡不重置，等级死亡清零。

后端使用 **Python + FastAPI + WebSocket**，前端为**终端风格**网页，部署于 Ubuntu Server 虚拟机。

---

## 2. 核心玩法循环

1. 在城镇/营地整理装备、合成物品、接受任务。
2. 进入随机生成的地牢，逐层向下探索。
3. 移动、战斗、搜索容器、拾取物品。
4. 在撤离点（8 方向无敌人，不含中立 NPC）按 `>` 直接撤离，本轮结束。
5. 结算：装备、合成材料、纪念品保留；其余换算金币。
6. 死亡：等级清零，背包内物品无法结算，本轮结束；技能与属性永久保留。
7. 用金币购买材料，用材料合成装备，继续下一轮。

---

## 3. 游戏规则详细设计

### 3.1 模式与回合制

- **纯单人 PVE**。
- **经典 Roguelike 回合制**：玩家行动一次 → 所有怪物行动一次 → 环境效果结算。
- 玩家不操作时游戏暂停。
- 玩家可以**等待**一回合（`.` 或空格）。
- **速度系统**沿用 NetHack：
  - 普通怪物：1 回合 1 动。
  - 快速怪物：1 回合 2 动。
  - 慢速怪物：2 回合 1 动。

### 3.2 地牢生成

- **层数**：共 10 层。
- **生成方式**：每次进入重新生成，房间位置、大小、道路、内部物品随机。
- **难度**：越深敌人越强。
- **剧情舞台房间**：
  - 共 2 个。
  - 第 3 或 4 层随机生成一个。
  - 第 7 或 8 层随机生成一个。
  - 内部固定布置，进入时触发 NPC 交互。
- **撤离点**：
  - 每层生成时随机放置 1~2 个。
  - 位置不固定。
  - 用 `>` 表示。
- **不能返回上层**，只能向下或撤离。

### 3.3 战斗系统

- **命中与伤害**：AC + 武器固定伤害 + 攻击骰。
- **有法术、技能、远程武器、负面效果**。
- **负面效果**：先实现中毒、麻痹；后续可扩展失明、恐惧等。
- **远程武器**：弓箭、投掷物。
- **法术**：通过卷轴学习，消耗 MP，MP 上限由 INT 决定。
- **技能**：战斗技能、通用技能、法术技能三类。

### 3.4 物品与背包

- **背包容量**：20 格，可通过任务扩展到 30 格。
- **不自动拾取**：散落物品踩上去提示“地上有 X，按 `g` 拾取”。
- **拾取/放下**：不消耗回合。
- **容器搜索**：
  - 按容器大小分两档：
    - 小容器（罐子、小箱）：搜索消耗 1 回合。
    - 大容器（宝箱、书架、尸体）：搜索消耗 2 回合。
  - 搜索期间可能被怪物攻击，中断搜索。
  - 搜索有几率触发陷阱。
  - 搜索成功率受 WIS + 搜索技能影响。
- **物品分类**：
  - 装备（武器/护甲/饰品）
  - 消耗品（药水/卷轴）
  - 合成材料
  - 纪念品
  - 金币
  - 其他（换算金币）

### 3.5 撤离与结算

- **撤离点随机刷新**：每层生成时随机放置 1~2 个。
- **撤离条件**：撤离点周围 8 方向内没有敌对目标（不包括中立 NPC）。
- **撤离方式**：不读条，按 `>` 直接撤离。
- **撤离后**：直接结算，本轮结束。
- **未探索区域、未搜索容器全部浪费**——故意的风险回报设计。

### 3.6 死亡惩罚

- **等级清零**。
- **背包内物品无法结算**，直接结束本轮游戏。
- **技能、属性、已学配方、任务进度、纪念品永久保留**。

### 3.7 养成系统

- **属性**：力量、敏捷、体质、智力、感知、魅力。
  - 初始值 8~12，创建角色时分配。
  - 升级给属性点，可永久增加。
  - 死亡后已加的属性点保留。
- **技能**：战斗、通用、法术三类。
  - 使用技能累积熟练度，达到阈值自动提升等级（0~5 级）。
  - 每级升级给 1 技能点，可加任意技能。
  - 死亡后技能等级和熟练度全部保留。
- **等级**：死亡清零。
- **金币**：用于购买材料。

### 3.8 剧情与任务

- **剧情呈现**：分支对话 + 环境叙事。
- **文本预先写好**，存储为结构化数据（JSON/YAML）。
- **NPC 交互**：在舞台房间触发，可能：
  - 触发任务
  - 改变结局
  - 引来敌人
  - 使文本变化
- **任务系统**：
  - NPC 给任务，有目标（如“找到第 5 层的某物品”）。
  - 任务跨轮保留。
  - 完成给永久奖励（属性点、技能点、装备）。
  - 状态：未接、进行中、已完成、失败。
- **结局**：
  - 多结局，由舞台房间选择 + 是否通关决定。
  - 通关第 10 层是结局之一，但非唯一。
  - 结局解锁后可在主菜单查看。
- **纪念品**：
  - 收集品，解锁剧情碎片。
  - 部分纪念品影响结局分支。
  - 死亡不丢失，永久保留。

### 3.9 合成系统

- **材料**在城镇合成。
- **配方**通过探索/任务获得，一旦学会永久保留。
- **合成产物**：装备、消耗品、升级装备。

---

## 4. 属性与技能系统

### 4.1 属性（6 项）

| 属性 | 影响 |
|---|---|
| 力量 STR | 近战伤害、负重 |
| 敏捷 DEX | 命中、闪避、先手 |
| 体质 CON | HP 上限、抗毒 |
| 智力 INT | 法术威力、技能点 |
| 感知 WIS | 搜索成功率、察觉陷阱 |
| 魅力 CHA | NPC 交互、商店价格 |

### 4.2 技能（三类）

- **战斗技能**：剑术、斧术、弓术、投掷、徒手
- **通用技能**：搜索、开锁、潜行、闪避、谈判
- **法术技能**：火系、冰系、治疗、增益、诅咒

### 4.3 法术学习

- 通过**卷轴**学习，消耗卷轴永久学会。
- 施法消耗 MP，MP 上限由 INT 决定。
- 法术有等级要求，技能等级不够无法施展。

---

## 5. 前端交互

- **终端风格**，使用 xterm.js 或纯 HTML `<pre>`。
- **地图字符画**：
  - `#` 墙
  - `.` 地板
  - `@` 玩家
  - `g` 怪物
  - `>` 撤离点
  - `+` 门
- **键盘指令**：
  - 移动：`h j k l y u b n`（8 方向）
  - 等待：`.` 或空格
  - 拾取：`g`
  - 搜索：`s`
  - 攻击：`a` + 方向
  - 施法：`c`
  - 背包：`i`
  - 撤离：`>`
- 下方为日志区，显示战斗、搜索、剧情文本。
- 侧边栏或状态栏显示 HP、MP、等级、金币、背包容量。

---

## 6. 技术架构

```
前端（浏览器）
  ├── WebSocket 连接
  ├── 渲染地图/日志/状态
  └── 发送指令（move/search/attack/extract）

后端（Ubuntu Server + Python）
  ├── FastAPI + WebSocket
  ├── GameEngine（内存中运行）
  │   ├── 地牢生成器（BSP / RDGen）
  │   ├── 回合调度器
  │   ├── 战斗系统
  │   ├── 物品/背包系统
  │   ├── 剧情引擎
  │   └── 撤离/结算系统
  ├── 数据库（SQLite / PostgreSQL）
  │   ├── 玩家存档
  │   ├── 仓库
  │   └── 剧情进度
  └── LLM 接口（可选，后期）
```

### 技术栈确认

- **后端**：FastAPI + WebSocket + SQLAlchemy (async) + SQLite
- **地牢生成**：RDGen 或自己写 BSP
- **前端**：HTML/CSS/JS + xterm.js（终端风格）
- **部署**：Ubuntu Server + Nginx + Uvicorn

---

## 7. 数据库表设计

```sql
-- 玩家账号
players (
    id INTEGER PRIMARY KEY,
    username TEXT UNIQUE,
    password_hash TEXT,
    created_at TIMESTAMP
)

-- 角色永久数据（死亡不重置）
characters (
    id INTEGER PRIMARY KEY,
    player_id INTEGER,
    name TEXT,
    str INTEGER, dex INTEGER, con INTEGER,
    int INTEGER, wis INTEGER, cha INTEGER,
    skill_points INTEGER,
    gold INTEGER,
    level INTEGER DEFAULT 1,  -- 死亡清零
    exp INTEGER DEFAULT 0,
    created_at TIMESTAMP
)

-- 技能熟练度（永久）
character_skills (
    id INTEGER PRIMARY KEY,
    character_id INTEGER,
    skill_name TEXT,
    level INTEGER DEFAULT 0,
    proficiency INTEGER DEFAULT 0
)

-- 仓库（永久保留）
warehouse (
    id INTEGER PRIMARY KEY,
    character_id INTEGER,
    item_type TEXT,  -- equipment/material/souvenir
    item_data JSON,
    quantity INTEGER
)

-- 已学配方
recipes (
    id INTEGER PRIMARY KEY,
    character_id INTEGER,
    recipe_name TEXT
)

-- 任务状态
quests (
    id INTEGER PRIMARY KEY,
    character_id INTEGER,
    quest_name TEXT,
    status TEXT,  -- active/completed/failed
    progress JSON
)

-- 剧情进度
story_progress (
    id INTEGER PRIMARY KEY,
    character_id INTEGER,
    node_id TEXT,
    choice_made TEXT,
    unlocked_endings JSON,
    souvenirs JSON
)

-- 本轮游戏存档（死亡或撤离后清空）
current_run (
    id INTEGER PRIMARY KEY,
    character_id INTEGER,
    dungeon_state JSON,  -- 地牢、玩家位置、怪物、物品
    turn_count INTEGER,
    started_at TIMESTAMP
)
```

---

## 8. GameEngine 模块划分

```
game_engine/
├── __init__.py
├── engine.py              # 主引擎，回合调度
├── dungeon/
│   ├── generator.py       # BSP 地牢生成
│   ├── rooms.py           # 房间类型、剧情舞台房间
│   └── tiles.py           # 地图格子定义
├── entities/
│   ├── player.py          # 玩家角色
│   ├── monster.py         # 怪物
│   ├── npc.py             # 中立 NPC
│   └── entity.py          # 基类
├── combat/
│   ├── attack.py          # 近战/远程攻击
│   ├── spells.py          # 法术
│   ├── effects.py         # 负面效果
│   └── damage.py          # 伤害计算
├── items/
│   ├── item.py            # 物品基类
│   ├── equipment.py       # 装备
│   ├── consumable.py      # 消耗品
│   ├── container.py       # 容器
│   └── inventory.py       # 背包
├── story/
│   ├── dialogue.py        # 分支对话
│   ├── quest.py           # 任务
│   └── narrative.py       # 环境叙事
├── extraction/
│   └── extraction.py      # 撤离点逻辑
├── settlement/
│   └── settlement.py      # 结算系统
└── utils/
    ├── rng.py             # 随机数（种子可复现）
    └── pathfinding.py     # 寻路
```

---

## 9. WebSocket 消息协议

### 客户端 → 服务器（动作）

```json
{"type": "action", "action": "move", "direction": "n"}
{"type": "action", "action": "wait"}
{"type": "action", "action": "pickup"}
{"type": "action", "action": "search"}
{"type": "action", "action": "attack", "direction": "ne"}
{"type": "action", "action": "cast", "spell": "fireball", "target": [5, 3]}
{"type": "action", "action": "extract"}
{"type": "action", "action": "dialogue_choice", "choice_id": "a"}
```

### 服务器 → 客户端（事件）

```json
{"type": "map_update", "map": "...", "player_pos": [5, 5]}
{"type": "message", "text": "你发现了一个宝箱。"}
{"type": "combat", "attacker": "player", "target": "goblin", "damage": 5}
{"type": "status", "hp": 20, "mp": 10, "level": 3, "gold": 100}
{"type": "dialogue", "npc": "old_man", "text": "...", "choices": [...]}
{"type": "game_over", "reason": "death", "settlement": {...}}
{"type": "game_over", "reason": "extraction", "settlement": {...}}
```

---

## 10. 目录结构

```
text-rpg/
├── backend/
│   ├── main.py              # FastAPI 入口
│   ├── ws.py                # WebSocket 处理
│   ├── game_engine/         # 引擎模块
│   ├── models/              # SQLAlchemy 模型
│   ├── data/
│   │   ├── monsters.json    # 怪物数据
│   │   ├── items.json       # 物品数据
│   │   ├── dialogue.json    # 对话数据
│   │   └── recipes.json     # 配方数据
│   └── requirements.txt
├── frontend/
│   ├── index.html
│   ├── style.css
│   └── game.js
└── README.md
```

---

## 11. 开发顺序

| 阶段 | 内容 |
|---|---|
| 1 | FastAPI + WebSocket 骨架，前后端能通信 |
| 2 | 地牢生成（BSP），玩家移动，回合制循环 |
| 3 | 怪物、战斗、AC + 攻击骰 + 伤害 |
| 4 | 物品、背包、容器搜索、拾取 |
| 5 | 撤离点、结算系统、死亡惩罚 |
| 6 | 存档、仓库、角色永久数据 |
| 7 | 剧情舞台房间、分支对话、任务 |
| 8 | 法术、技能、负面效果 |
| 9 | 合成系统、金币购买 |
| 10 | 多结局、纪念品、剧情碎片 |

---

## 12. 附录：关键设计决策摘要

- **单人 PVE**，经典 Roguelike 回合制。
- **10 层地牢**，每轮重新生成，越深越强。
- **剧情舞台房间**：第 3/4 层随机一个，第 7/8 层随机一个，内部固定布置。
- **撤离点**：每层随机 1~2 个，8 方向无敌人（不含中立 NPC）时按 `>` 直接撤离。
- **死亡惩罚**：等级清零，背包内物品无法结算；技能、属性、配方、任务、纪念品永久保留。
- **结算保留**：装备、合成材料、纪念品；其余换算金币。
- **金币**用于购买材料。
- **容器搜索**：小容器 1 回合，大容器 2 回合。
- **拾取/放下**不消耗回合，不自动拾取。
- **背包** 20 格，可扩展至 30 格。
- **战斗**：AC + 武器固定伤害 + 攻击骰；有法术、技能、远程武器、负面效果。
- **剧情**：分支对话 + 环境叙事，文本预先写好。
- **技术栈**：FastAPI + WebSocket + SQLAlchemy (async) + SQLite；前端 xterm.js 终端风格；Ubuntu Server + Nginx + Uvicorn。

---

这份文档可作为项目的完整设计基线，后续开发可直接按此推进。
