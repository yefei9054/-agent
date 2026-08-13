# Agent 结构与记忆数据库设计规范

> 配套 `落地骨架/SKILL.md`。本文件补齐框架的"身体结构"：内部模块、记忆体系、数据库 schema、性格、行事准则。SKILL.md 已引用本文件。

---

## 一、Agent 内部结构（单一统一智能体的五脏六腑）

虽是单一人格、单一入口，内部按职责分六个模块协同：

| 模块 | 职责 | 关键产出 |
|---|---|---|
| 感知 / 输入 | 接收清单 / 图纸 / 规范文本并结构化 | 任务对象（阶段 + 工作流 + 清单项） |
| 规划 | 判定所处阶段、该跑哪条工作流 | 任务计划 |
| 执行 | 特征解析 / 套定额 / 组价 / 取费 / 价差 | 中间结果 |
| 缺口检测 | 实时比对资产层，发现缺失 | 补全提示卡 |
| 记忆与资产 | 读 / 写资产库与记忆层 | 持久化知识 |
| 可靠性闸门 | 执行三铁律，卡高风险 / 不确定 | 放行 or 停疑 |

**任务状态机（一次清单项的处理）**
`接入 → 盘点资产 → 规划 → 解析特征 → 查定额 → 查价 → 组价取费 → [缺口?→提示+引导] → [不确定?→停] → 出成果(留痕) → 回写资产`

**Agent 需要的能力（工具接口，实施时落地）**
- `search_quota(特征)` → 查定额库
- `get_price(材料, 规格)` → 查价库
- `parse_feature(文本)` → 提取属性并匹配特征规则
- `calc_fee(基数, 模板)` → 取费计算
- `web_lookup(规范/价)` → 联网补缺口
- `asset_read / asset_write` → 读写资产层与记忆层

---

## 二、记忆体系（四层）

| 层 | 内容 | 载体 | 生命周期 |
|---|---|---|---|
| 会话记忆 | 当前任务上下文、临时推断 | 上下文窗口 | 当次会话 |
| 资产记忆 | 六类资产（定额 / 价 / 规则 / 取费 / 案例 / 规范） | 本地库（默认 SQLite） | 长期，随用随补 |
| 用户画像 | 水平（资深 / 初中级）、偏好、补全习惯 | 库表 `user_profile` | 长期 |
| 缺口轨迹 | 哪些缺过、填没填、何时填 | 库表 `gap_log` | 长期 |

**记忆生命周期**
- 会话开始：加载画像 + 资产概览（已备 / 未备）+ 近期缺口 → 先盘点再干活。
- 任务中：读资产匹配；缺口写 `gap_log` 并即时提示；用户补全后写回对应资产表。
- 会话结束：固化资产与缺口轨迹，下次直接复用。

---

## 三、数据库设计（推荐 SQLite：`cost_agent_assets.db`）

> 框架通用，字段可按专业 / 地区增删。以下为机电安装示范结构。技术栈不强制——用户可用 SQLite / JSON / 其他，只要 agent 能读能写。

### 3.1 定额库 `quota_items`
```
id INTEGER PK
code TEXT           -- 定额编号 如 9-1
chapter TEXT        -- 章 如 C7.2 通风管道
name TEXT           -- 子目名称
unit TEXT           -- 单位 m2/台/m
labor REAL          -- 人工费
machine REAL        -- 机械费
material_note TEXT  -- 材料说明（主材乙供/甲供）
applicable_features TEXT -- 适用特征(JSON: 材质/规格区间/敷设)
spec_ref TEXT       -- 规范出处
source TEXT
created_at TEXT
```

### 3.2 材料价库 `material_prices`
```
id INTEGER PK
name TEXT           -- 材料名
spec TEXT           -- 规格
unit TEXT
price REAL
price_type TEXT     -- 信息价/市场价/合同价/甲供
source TEXT         -- 期刊号/合同号/供应商
date TEXT
conversion TEXT     -- 换算规则 t<->kg 等
created_at TEXT
```

### 3.3 特征规则库 `feature_rules`
```
id INTEGER PK
pattern TEXT        -- 特征模式
attrs_json TEXT     -- 提取属性: 材质/规格/压力/敷设
quota_code TEXT     -- 映射定额编号
confidence REAL
created_at TEXT
```

### 3.4 取费模板 `fee_templates`
```
id INTEGER PK
region TEXT         -- 地区
project_type TEXT   -- 工程类别
base_type TEXT      -- 取费基数 如 人工费+机械费
rates_json TEXT     -- 各费率: 管理/利润/规费/税金
created_at TEXT
```

### 3.5 案例库 `case_lib`
```
id INTEGER PK
name TEXT
region TEXT
type TEXT
unit_price REAL     -- 单方指标 元/m2
contents_json TEXT  -- 主要材料含量指标
created_at TEXT
```

### 3.6 规范文档 `spec_docs`
```
id INTEGER PK
title TEXT
path_or_url TEXT
scope TEXT          -- 适用阶段
key_clauses_json TEXT -- 关键条款索引
embedding BLOB      -- 可选：向量，供语义检索
```

### 3.7 用户画像 `user_profile`
```
user_level TEXT      -- 资深/初中级
preferences_json TEXT
created_at TEXT
```

### 3.8 缺口轨迹 `gap_log`
```
id INTEGER PK
task TEXT
asset_type TEXT     -- 定额/价/规则/取费/案例/规范
missing_item TEXT
prompted_at TEXT
filled_at TEXT NULL -- 未填为 NULL
resolution TEXT
```

> 规范文档可选接向量库做语义检索；结构化资产用 SQL 精确匹配即可。

---

## 四、性格规范

- **身份**：一位严谨、靠谱的资深造价师副驾，不是热情过度的客服，也不是爱炫技的模型。
- **骨子里的特质**：守规范、重证据、怕出错；宁可说"我不确定"，绝不编一个看起来对的答案。
- **对资深用户**：直接、干脆，给清单和口径，不绕弯子。
- **对初中级用户**：耐心、肯教，把一步拆开讲、配例子，但绝不替他把活干完。
- **说话方式**：用同行大白话，像造价室里带徒弟的老师傅；禁用"赋能 / 闭环 / 底层逻辑"等空话。
- **底线**：不做假、不估猜、不替用户拍板、不给无依据的"标准答案"。
- **情绪处理**：遇到缺口不慌，把它当教学机会；遇到违规请求直接拒，并说明原因。

---

## 五、行事准则（六条职业守则）

1. **可靠性**：出处可溯 / 拿不准就停 / 过程留痕（见 SKILL.md 三铁律）。
2. **职业操守**：严格按清单规范与定额规则办事；维护用户利益，发现量价异常主动提醒；不协助任何违规计价。
3. **协作分权**：常规环节自动；出成果文件 / 重大换算 / 无价决策 / 边界模糊卡人工确认，绝不越权定案。
4. **引导不代劳**：发现资产缺口，教用户"按什么标准建"，不替他凭空造一份资产糊弄过去。
5. **全程透明**：取费、折算、价差每步可复核；成果附计算溯源表。
6. **守住边界**：不造假数据、不猜测无据结论、不越权替用户决策、不泄露用户资产与项目涉密信息。

---

## 六、与落地骨架的对接

- SKILL.md 已嵌入「性格 / 行事准则 / 记忆与资产层交互」三节，并引用本文件。
- 资产层六表 + 画像 + 缺口轨迹，是本框架持久化的"身体"；没有它们，agent 只会空谈。
- 实施时：先建空的 `cost_agent_assets.db`（按上述 schema），agent 首次会话即引导用户盘点并逐步填充；规范文档可后接向量库做语义检索。
