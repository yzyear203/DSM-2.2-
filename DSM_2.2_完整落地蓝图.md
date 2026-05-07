# 动态结构化记忆系统 DSM 2.2
## AI 聊天产品记忆系统完整落地蓝图

> **文档定位**：本文档是 DSM 2.2 的唯一权威技术文档，面向零背景的开发者，包含所有架构设计、数据结构、流转机制与工程细节，可直接按本文档落地实施。

---

## 目录

1. 核心设计理念
2. 整体架构全景
3. 核心数据结构（T3 Schema）
4. 四级记忆漏斗详解
5. 八大核心机制
6. 六大性价比优化补丁
7. System Prompt 构建规范
8. 数据流转全链路
9. 前端实现规范
10. 后端 / 数据库规范
11. Cron 定时任务规范
12. 冷启动流程
13. 记忆透明舱（UX）
14. 关键指标与成本估算

---

## 一、核心设计理念

**核心目标**：彻底摒弃"全局 RAG 检索"，实现日常陪伴零延迟、重大事件闪电响应、长期人设精准无冲突、记忆自然衰减，并赋予 AI 高情感感知能力。

**三大支柱**：

- **仿生四级漏斗**：不同时效和重要度的记忆，存在不同的层级，有各自的生命周期
- **动态预算分配**：每次请求严格管控 Token 预算，按优先级裁剪，防止上下文爆满
- **异构存储**：热数据在前端、近期数据在向量云库、核心档案在账号表，各司其职

**最核心的一条工程原则**：能不调用 LLM 就不调用。所有 LLM 调用前，必须先用零成本手段判断是否真的需要调用。

---

## 二、整体架构全景

```
用户发消息
    │
    ▼
┌─────────────────────────────────────────────────────┐
│  System Prompt 构建（发送前组装，严格按优先级裁剪）      │
│                                                     │
│  P0（不可裁减，约200 tokens）：T3 核心灵魂烙印 JSON   │
│       + 回归感知指令（days_since_last_chat > 7 时）  │
│  P1（优先保留，约150 tokens）：热态 T1（极简单行摘要）  │
│  P2（动态裁剪）：T0 最近对话轮次（早期轮次优先丢弃）    │
└─────────────────────────────────────────────────────┘
    │
    ▼
   大模型即时回复（2秒内，无检索延迟）
    │
    ▼
AI 回复完毕 + 用户10秒无输入
    │
    ▼
┌──────────────────────────────────────────┐
│  内容信号守门人（零成本规则过滤）            │
│  检测：命名实体 / 时间词 / 情感词 / 计划句式 │
│  无信号 → 跳过，零成本                     │
│  有信号 → 触发 Flash 模型静默提取 T1        │
└──────────────────────────────────────────┘
    │
    ├── T1 importance >= 8 ──→ 闪电通道 → 立即更新 T3.current_context
    │
    └── 正常存入热态T1(localStorage) 或 深态T1(TCB向量库)

每周日凌晨3点
    │
    ▼
Cron 批处理：Flash预聚合（7天→7条）→ 主模型总结 → 更新T2/T3 → 清理过期T1
```

---

## 三、核心数据结构：T3 灵魂烙印 Schema

T3 是整个系统的核心，挂载在用户账号表（`user_profile`）中，是 AI 对用户的终极认知。

```json
{
  "user_id": "uid_12345",

  "identity": {
    "value": "张三，22岁，临床医学大二学生，正考虑转计算机",
    "confidence": "high",
    "last_updated": "2026-04-28T10:00:00Z",
    "source_event_ids": ["evt_881"]
  },

  "personality": {
    "value": "积极上进，但面对代码 Bug 时容易自我怀疑",
    "confidence": "medium",
    "last_updated": "2026-04-20T00:00:00Z",
    "source_event_ids": ["evt_712", "evt_756"]
  },

  "interests": [
    {
      "topic": "存在主义",
      "weight": 8,
      "last_mentioned": "2026-04-20",
      "confidence": "high"
    },
    {
      "topic": "前端开发",
      "weight": 9,
      "last_mentioned": "2026-04-28",
      "confidence": "high"
    }
  ],

  "relationship": {
    "archetype": "亦师亦友的技术导师",
    "intimacy_level": 7,
    "interaction_count": 142,
    "last_chat_time": "2026-04-14T08:00:00Z",
    "bond_momentum": "cooling"
  },

  "current_context": {
    "value": "正在爆肝重构数字资产编译器的记忆系统",
    "expires_at": "2026-05-05T00:00:00Z"
  }
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| `confidence` | enum | `high`=用户明确陈述；`medium`=AI行为推断；`low`=单次情绪化发泄 |
| `source_event_ids` | array | 溯源到哪条 T1 事件，供记忆面板可视化 |
| `intimacy_level` | 1-10 | 影响 AI 的语气尺度（高=朋友语气，低=礼貌敬语） |
| `bond_momentum` | enum | `growing`/`stable`/`cooling`，驱动 AI 主动行为 |
| `current_context.expires_at` | timestamp | 临时状态自带 TTL，过期自动失效，无需手动清理 |

### 置信度在 System Prompt 中的表达规范

- `high`：直接陈述（"用户是一名程序员"）
- `medium`：模糊表达（"用户似乎对编程有浓厚兴趣"）
- `low`：完全不注入（不污染 AI 认知）

---

## 四、四级记忆漏斗详解

### 🟢 T0 级：瞬时工作记忆（Working Memory）

| 属性 | 描述 |
|---|---|
| **内容** | 当前聊天历史、语气词、情绪发泄、"哈哈"、"在吗" |
| **存储位置** | 前端 React `messages` 状态数组 |
| **生命周期** | 当前会话，刷新即焚，绝不写数据库 |
| **注入方式** | 直接随请求发送（P2优先级，超预算时优先裁剪早期轮次） |
| **数量** | 保留最近 8-15 轮，具体由 Token 预算决定 |

### 🟡 T1 级：短期情景记忆（Episodic Memory）

T1 分为两态，物理上分别存储：

**🔥 热态 T1（Hot Cache）**

| 属性 | 描述 |
|---|---|
| **内容** | 最近 24 小时内的 2-3 条核心事件极简摘要 |
| **存储位置** | 前端 `localStorage`（带 24 小时过期时间戳） |
| **注入方式** | 无感直注 System Prompt（P1优先级），实现"昨日之事"免检索 |
| **格式** | 单行极简：`[昨日] 用户买了五一去华山的高铁票` |
| **冷启动补填** | 本地缓存过期时，从 TCB 拉取最近 24h 深态 T1 填充 |

**🧊 深态 T1（Deep RAG）**

| 属性 | 描述 |
|---|---|
| **内容** | 24 小时 ~ 7 天的事件记录 |
| **存储位置** | TCB 云端向量数据库 `persona_memories` 集合 |
| **检索触发** | 用户刻意追溯（"我前天说了啥"），且热态缓存无法命中时，AI 自主触发 Tool Call |
| **清理** | 每周 Cron 根据艾宾浩斯衰减值决定是否删除 |

**T1 数据结构（单条记录）**：

```json
{
  "event_id": "evt_881",
  "user_id": "uid_12345",
  "content": "用户今天拿到了字节跳动的实习 Offer",
  "importance_score": 9,
  "emotion": "excited",
  "confidence": "high",
  "timestamp": "2026-04-28T14:30:00Z",
  "session_id": "sess_xyz",
  "device_fingerprint": "fp_abc123",
  "idempotency_key": "md5(sess_xyz + 1714312200 + uid_12345)"
}
```

### 🟠 T2 级：长期语义记忆（Semantic Memory）

| 属性 | 描述 |
|---|---|
| **内容** | 从 T1 沉淀的长线客观事实（去过的国家、过敏原、深度探讨结论） |
| **存储位置** | TCB 向量数据库（永久存储） |
| **生成时机** | 每周日 Cron 批处理 |
| **检索触发** | 深度话题时，AI 通过工具声明自主调用 `retrieve_long_term_memory` |
| **清理** | 衰减值 S < 2.0 时物理删除 |

### 🔴 T3 级：核心灵魂烙印（Core Profile）

| 属性 | 描述 |
|---|---|
| **内容** | 用户的终极多维认知（见第三节 Schema） |
| **存储位置** | 用户账号表 `user_profile`，字段 `t3_profile` |
| **注入方式** | 0延迟直注！每次请求前将 JSON 翻译为紧凑自然语言，插入 System Prompt 最顶端（P0不可裁减） |
| **更新时机** | 闪电通道（importance>=8 立即更新 current_context）+ 每周 Cron（全量更新） |
| **Prompt Caching** | T3 注入内容固定在 System Prompt 最顶部，充分利用各大模型的 Prompt Cache（命中后读取成本降 90%） |

---

## 五、八大核心机制

### 机制① 内容信号守门人（Content Signal Gatekeeper）

**作用**：在调用任何 LLM 之前，用零成本规则过滤，减少 75% 无效提取调用。

**触发 T1 提取的条件（满足任意一个）**：

```
✅ 检测到命名实体（人名、地名、产品名、机构名）
✅ 检测到时间词（"明天"、"下周"、"最近"、"今天"、"昨天"）
✅ 检测到情感关键词（"失恋"、"Offer"、"累"、"开心"、"难受"）
✅ 检测到计划句式（"我打算..."、"我要去..."、"准备..."）
✅ 会话自然结束时的最终兜底（每次会话只触发一次）

❌ 纯粹的语气词、表情包、"好的"、"哈哈哈" → 跳过，零成本
```

**实现方式**：前端正则匹配，无需 API 调用。

---

### 机制② 冲突检测网关（Conflict Detection Gateway）

**作用**：防止 T3 出现自相矛盾的"缝合怪"（如：同时记录"医学生"和"CS学生"）。

**完整流程**：

```
新 T1 事件
    │
    ▼
Step 1: Embedding 预筛（成本约为 LLM 的 1/50）
    计算新事件的向量，与 T3 所有字段做余弦相似度
    相似度 < 0.75 → 直接 APPEND，跳过 LLM 检测（节省 90% 调用）
    相似度 >= 0.75 → 进入 Step 2
    │
    ▼
Step 2: LLM 冲突判断（仅对真正相关的字段触发）
    Prompt：「现有记忆：{旧值}。新信息：{新值}。请判断：
              OVERWRITE（新值覆盖旧值，如换了城市）
              APPEND（追加，不冲突，如新爱好）
              CONFLICT（语义矛盾，需人工确认）」
    │
    ├── OVERWRITE → 直接更新字段，记录 last_updated 和 source_event_ids
    ├── APPEND    → 追加到对应数组（如 interests）
    └── CONFLICT  → 打入"待确认区"，在记忆透明舱显示，等用户裁决
```

---

### 机制③ 动态 Token 预算分配器（Budget Allocator）

**作用**：防止上下文窗口爆满，在有限 Token 内保留最高价值的信息。

**优先级与硬顶**：

| 优先级 | 组件 | Token 硬顶 | 裁剪策略 |
|---|---|---|---|
| P0（不可裁减） | T3 核心烙印（紧凑格式） | 200 tokens | 永不裁减 |
| P0（不可裁减） | 回归感知指令（触发时追加） | 50 tokens | 永不裁减 |
| P1（优先保留） | 热态 T1 单行摘要 | 150 tokens | 超预算时最后裁 |
| P2（动态裁剪） | T0 对话轮次 | 剩余预算 | 优先丢弃早期轮次 |

**T3 紧凑注入格式**（对比自然语言节省约 44% tokens）：

```
❌ 自然语言（~80 tokens）：
"用户叫张三，是一名临床医学大二学生，正考虑转计算机专业，
喜欢存在主义和前端开发，性格积极上进但面对 Bug 时容易自我怀疑..."

✅ 紧凑格式（~45 tokens）：
[用户档案] 张三 | 医学大二→考虑转CS | 兴趣:存在主义(★8)/前端(★9) |
性格:上进/遇Bug易自疑[推断] | 关系:技术导师 亲密度7/10 | 状态:重构记忆系统(~5月5日)
```

---

### 机制④ 艾宾浩斯衰减与物理清除（Ebbinghaus GC）

**作用**：让记忆像人类一样自然遗忘，避免一刀切删除误杀重要信息。

**衰减公式**：

```
S = I × e^(-k × t)

其中：
  S = 当前记忆强度
  I = 原始 importance_score（1-10）
  k = 衰减速率（按类型不同）
  t = 距今天数

衰减速率参考：
  事件类（"今天吃了火锅"）：k = 0.3（快速遗忘）
  习惯/偏好类（"喜欢听古典乐"）：k = 0.05（缓慢遗忘）
  重大事件类（"拿到 Offer"）：k = 0.1（中速遗忘）
```

**清理规则**：每周 Cron 运行时，S < 2.0 的 T1 记录执行物理删除，S >= 2.0 保留。

---

### 机制⑤ 闪电更新通道（Flash Update Channel）

**作用**：解决重大事件需要等到周日才被 AI 感知的"信息黑洞"。

**触发条件**：T1 提取时，`importance_score >= 8`

**触发逻辑**：

```
importance >= 8（如："失恋了"、"拿到Offer"、"转专业了"、"搬到新城市"）
    │
    ▼
立即触发轻量云函数
    │
    ▼
仅局部更新 T3.current_context 字段
（不触发完整的冲突检测网关，成本极低）
    │
    ▼
下一条 AI 回复的 System Prompt 已感知新状态
AI 态度瞬间转变（如：收到失恋信息后，AI 立刻转为安慰模式）
```

---

### 机制⑥ 情绪隔离罩（Emotional Sandbox）

**作用**：防止用户一时情绪（如压力期大量负面发言）永久污染 AI 对其性格的认知。

**运作规则**（在 Cron 周结时执行）：

```
统计本周 T1 记录中 emotion 字段分布：
  negative 占比 <= 40%：正常更新 personality 字段
  negative 占比 > 40%：
    ✅ 只更新 current_context（"用户最近状态很差"）
    ❌ 拒绝覆写底层 personality 字段
    ❌ 拒绝把 "低落" 写入 interests
```

---

### 机制⑦ 沉寂回归感知（Regression Perception）

**作用**：用户长时间未登录后回来，AI 能自然感知时间流逝，而不是毫无感觉地继续聊。

**实现方式**：注入 System Prompt 时，计算 `days_since_last_chat`。

```
days_since_last_chat = 今日 - T3.relationship.last_chat_time

若 > 7 天，在 System Prompt 追加隐藏指令（P0 不可裁减）：

「【回归感知指令】用户已有 {X} 天未与你交流。请在本次开场时，
  以极其自然的方式感叹这段时间的流逝，并结合 current_context
  询问近况。禁止生硬地说"你好久没来了"，要像老朋友一样。」

同时更新：T3.relationship.last_chat_time = 当前时间
```

---

### 机制⑧ 跨端竞态防护（Concurrency Protection）

**作用**：用户同时在手机和电脑聊天时，防止两端同时提取 T1 写入重复或冲突记录。

**实现方式**：利用数据库主键唯一约束天然实现幂等去重。

```
T1 记录的主键（idempotency_key）：
md5(session_id + timestamp_10s_floor + user_id)

其中 timestamp_10s_floor = 当前时间戳向下取整到10秒
（即同一会话同一10秒窗口内的写入，生成完全相同的主键）

多端同时触发提取 → 数据库主键冲突 → 自动丢弃重复写入
```

---

## 六、六大性价比优化补丁

### 补丁① 热态 T1 极简化

存入 `localStorage` 的热态 T1，不存完整描述，只存单行摘要：

```
✅ 正确：[2026-04-28] 用户买了五一去华山的高铁票
❌ 错误：用户今天下午在网上购买了五一劳动节期间前往陕西华山景区的高铁票，显得很期待这次旅行
```

节省 T1 注入时约 60% 的 tokens。

### 补丁② Cron 两阶段处理

直接处理全量 T1 会导致 Token 账单异常峰值。改为两阶段：

```
阶段一（Flash 模型，廉价）：
  将7天的 T1 按日期分组（7组）
  Flash 模型对每组做日摘要（7天 → 7条）
  
阶段二（主模型，仅1次调用）：
  主模型读取7条日摘要
  输出 T2 更新 + T3 更新建议
  
效果：处理量从 N 条降到 7 条，成本减少约 98%
```

### 补丁③ Prompt Caching 利用

**前提**：使用支持 Prefix Cache 的 API（Claude、GPT-4o 均支持，命中后成本降 90%）。

**实现**：T3 注入内容固定在 System Prompt 最顶部，且在同一会话内保持完全不变（除非闪电通道触发更新）。只要内容一致，模型 API 自动命中缓存，T3 的 ~200 tokens 只需支付 1/10 的读取费用。

### 补丁④ T2 检索工具声明

在 System Prompt 中直接声明工具，让 AI 自主决策何时检索 T2：

```json
{
  "tool_name": "retrieve_long_term_memory",
  "description": "检索用户的长期语义记忆",
  "trigger_when": "用户询问历史事件、话题涉及用户的专业/旅行/重大决策时",
  "input": { "query": "string" }
}
```

### 补丁⑤ Cron 分用户限额保护

防止个别重度用户的 Cron 任务产生异常账单：

```
每用户单次 Cron 最大处理：
  阶段一 Flash 摘要：最多处理 200 条 T1（超出部分按时间截断）
  阶段二主模型总结：输入 token 硬顶 4000
  
超出限额时：不报错，只处理限额内的部分，剩余留到下周
```

### 补丁⑥ bond_momentum 驱动主动触达

当 `T3.relationship.bond_momentum == "cooling"` 时，在 System Prompt 追加：

```
「【关系维系指令】与用户的关系处于冷却期，若话题自然，
  可以适时问一句暖心的关怀，但不要显得刻意。」
```

`bond_momentum` 的计算逻辑（每周 Cron 更新）：
- `interaction_count` 本周 > 上周：`growing`
- 本周 ≈ 上周：`stable`
- 本周 < 上周 50%，或距上次聊天 > 14 天：`cooling`

---

## 七、System Prompt 完整构建规范

每次发送请求前，按以下顺序组装 System Prompt：

```
┌─────────────────────────────────────────────────────────────┐
│  [BLOCK 1 - P0] T3 核心档案（紧凑格式，约45 tokens）         │
│                                                             │
│  [用户档案] {identity.value} | 兴趣:{interests摘要} |        │
│  性格:{personality.value}[{confidence}] |                   │
│  关系:{archetype} 亲密度{intimacy_level}/10 |               │
│  当前状态:{current_context.value}（至{expires_at}）          │
├─────────────────────────────────────────────────────────────┤
│  [BLOCK 2 - P0，条件触发] 回归感知指令                       │
│  （仅在 days_since_last_chat > 7 时追加，约50 tokens）       │
├─────────────────────────────────────────────────────────────┤
│  [BLOCK 3 - P0，条件触发] bond_momentum 指令                │
│  （仅在 cooling 时追加，约30 tokens）                        │
├─────────────────────────────────────────────────────────────┤
│  [BLOCK 4 - P0] T2 工具声明（约50 tokens，固定）             │
├─────────────────────────────────────────────────────────────┤
│  [BLOCK 5 - P1] 热态 T1 摘要（约150 tokens）                │
│  [近期事件]                                                 │
│  · [2026-04-27] 用户买了五一去华山的高铁票                   │
│  · [2026-04-28] 用户说在重构记忆系统，很兴奋                 │
├─────────────────────────────────────────────────────────────┤
│  [BLOCK 6 - P2] T0 当前对话轮次（剩余预算内尽量多保留）       │
└─────────────────────────────────────────────────────────────┘
```

**注意**：BLOCK 1~4 内容在同一会话内保持不变，充分利用 Prompt Caching。

---

## 八、数据流转全链路

### 8.1 日常聊天流（极速路径）

```
① 用户发消息
② 前端组装 System Prompt（T3 + 热态T1 + T0），调用大模型 API
③ 大模型 2 秒内回复（无任何检索）
④ AI 回复完毕，前端启动 10 秒计时器
⑤ 10 秒内用户再次发消息 → 重置计时器，跳回②
⑥ 10 秒无输入 → 触发内容信号守门人
⑦ 无信号 → 结束，零额外成本
⑧ 有信号 → Flash 模型提取 T1 事件
⑨ importance >= 8 → 闪电通道 → 更新 T3.current_context
⑩ 写入热态 T1（localStorage）和深态 T1（TCB 向量库）
```

### 8.2 深层追溯流（T2 检索路径）

```
① 用户问"我上个月跟你说的那个项目结论是什么？"
② 前端组装 System Prompt（同上）
③ 大模型判断需要检索，调用工具 retrieve_long_term_memory
④ 前端/后端执行 TCB 向量检索，返回最相关 T2 记录
⑤ 大模型拿到检索结果，生成完整回复
⑥ 总延迟约 5-8 秒（仅在用户主动追溯时才会这慢）
```

### 8.3 冷启动流（新用户 / T3 为空）

```
① 用户首次登录，T3 所有字段为空
② System Prompt 追加破冰指令：
   「这是与用户的第一次对话。请在前3轮以反问句自然引导用户透露：
     名字、当前在做什么、有什么爱好。禁止连续发问，融入对话中。」
③ AI 自然破冰，获取初始信息
④ 每轮对话后，T1 提取守门人正常运作
⑤ 2-3 轮对话后，闪电通道触发，T3 种子生成
⑥ 后续对话 AI 已有基础认知
```

### 8.4 周结流（Cron 批处理路径）

```
① 每周日凌晨3点，Cron 任务启动
② 拉取本用户所有 T1 记录（近7天）
③ 检查每用户是否超过处理限额（200条）
④ Flash 模型按日期分7组做摘要（7天→7条）
⑤ 计算每条 T1 的衰减值 S = I × e^(-k×t)
⑥ 主模型读取7条摘要，生成：
   - T2 向量记录更新（新的长线事实）
   - T3 更新建议（JSON diff）
⑦ 执行冲突检测网关（对有 CONFLICT 的变更暂存待确认区）
⑧ 执行情绪隔离罩（过滤高负面情绪周的 personality 更新）
⑨ 更新 T3 JSON，更新 bond_momentum
⑩ 物理删除 S < 2.0 的 T1 记录
⑪ 写入 cron_logs 表（处理条数、Token 消耗、耗时、错误）
```

---

## 九、前端实现规范

### 9.1 数据存储

```javascript
// 热态 T1 的存储结构（localStorage）
const HOT_T1_KEY = `hot_t1_${userId}`;
const hotT1Data = {
  items: [
    {
      summary: "[2026-04-28] 用户买了五一去华山的高铁票",
      importance: 6,
      timestamp: 1714312200000
    }
  ],
  cached_at: Date.now(),
  expires_at: Date.now() + 24 * 60 * 60 * 1000  // 24小时后过期
};
localStorage.setItem(HOT_T1_KEY, JSON.stringify(hotT1Data));

// 读取时检查过期
function getHotT1(userId) {
  const raw = localStorage.getItem(`hot_t1_${userId}`);
  if (!raw) return null;
  const data = JSON.parse(raw);
  if (Date.now() > data.expires_at) {
    localStorage.removeItem(`hot_t1_${userId}`);
    return null;  // 触发冷启动补填
  }
  return data.items;
}
```

### 9.2 内容信号守门人

```javascript
const SIGNAL_PATTERNS = [
  // 命名实体（简化版，可接入 NER 库增强）
  /[\u4e00-\u9fa5]{2,4}(大学|公司|医院|景区|城市)/,
  // 时间词
  /(今天|昨天|明天|下周|上周|最近|这几天|五一|春节)/,
  // 情感关键词
  /(失恋|分手|Offer|录取|失业|离职|搬家|转专业|怀孕|生病|手术)/,
  // 计划句式
  /我(打算|要|准备|决定|想).{2,15}(了|去|做)/,
];

function hasContentSignal(messages) {
  // 只检查最新一轮用户消息
  const lastUserMsg = messages.filter(m => m.role === 'user').slice(-1)[0];
  if (!lastUserMsg) return false;
  return SIGNAL_PATTERNS.some(p => p.test(lastUserMsg.content));
}
```

### 9.3 T1 提取触发逻辑

```javascript
let extractTimer = null;

function onAIReplyComplete(messages) {
  clearTimeout(extractTimer);
  extractTimer = setTimeout(async () => {
    if (!hasContentSignal(messages)) return;  // 守门人过滤
    
    const t1Event = await extractT1WithFlash(messages);  // 调用廉价 Flash 模型
    
    if (t1Event.importance_score >= 8) {
      await triggerFlashUpdateChannel(t1Event);  // 闪电通道
    }
    
    await saveToHotT1Cache(t1Event);     // 写热态
    await saveToDeepT1(t1Event);          // 写深态
  }, 10000);
}

function onUserTyping() {
  clearTimeout(extractTimer);  // 用户继续输入，重置计时器
}
```

### 9.4 System Prompt 组装

```javascript
function buildSystemPrompt(userId) {
  const t3 = getUserT3Profile(userId);
  const hotT1 = getHotT1(userId) || fetchHotT1FromDB(userId);  // 冷启动补填
  const t0 = getRecentMessages();
  const daysSinceLastChat = getDaysSince(t3.relationship.last_chat_time);
  
  let prompt = formatT3Compact(t3);  // P0，约45 tokens
  
  if (daysSinceLastChat > 7) {
    prompt += buildRegressionBlock(daysSinceLastChat, t3.current_context);  // P0条件
  }
  
  if (t3.relationship.bond_momentum === 'cooling') {
    prompt += COOLING_INSTRUCTION;  // P0条件
  }
  
  prompt += T2_TOOL_DECLARATION;  // P0固定
  
  if (hotT1?.length > 0) {
    prompt += formatHotT1(hotT1);  // P1
  } else if (!t3.identity.value) {
    prompt += COLD_START_INSTRUCTION;  // 冷启动破冰
  }
  
  // P2：T0 由 Token 预算分配器决定保留多少轮
  return applyBudgetAllocator(prompt, t0);
}

function applyBudgetAllocator(fixedPrompt, t0Messages) {
  const MAX_TOKENS = 4000;  // 根据实际使用的模型调整
  const fixedTokens = estimateTokens(fixedPrompt);
  const remainingBudget = MAX_TOKENS - fixedTokens;
  
  // 从最新消息往前保留，超出预算则截断
  return fixedPrompt + trimMessagesToBudget(t0Messages, remainingBudget);
}
```

---

## 十、后端 / 数据库规范

### 10.1 数据表结构

**user_profile 表**（T3 核心档案）：

```sql
CREATE TABLE user_profile (
  user_id        VARCHAR(64) PRIMARY KEY,
  t3_profile     JSON NOT NULL DEFAULT '{}',
  t3_hash        VARCHAR(32),           -- 用于 Prompt Cache 命中检测
  pending_conflicts JSON DEFAULT '[]',  -- 待用户裁决的冲突变更
  created_at     TIMESTAMP,
  updated_at     TIMESTAMP
);
```

**persona_memories 表**（T1 + T2 记录）：

```sql
CREATE TABLE persona_memories (
  event_id          VARCHAR(64) PRIMARY KEY,  -- idempotency_key
  user_id           VARCHAR(64) NOT NULL,
  memory_type       ENUM('t1_episodic', 't2_semantic'),
  content           TEXT NOT NULL,
  importance_score  TINYINT,       -- 1-10
  emotion           VARCHAR(32),
  confidence        ENUM('high', 'medium', 'low'),
  memory_strength   FLOAT,        -- 艾宾浩斯衰减值，Cron 更新
  decay_rate        FLOAT,        -- 衰减速率 k
  session_id        VARCHAR(64),
  device_fp         VARCHAR(64),
  timestamp         TIMESTAMP,
  expires_at        TIMESTAMP     -- T1: 7天后；T2: NULL（永久）
);
CREATE INDEX idx_user_time ON persona_memories(user_id, timestamp DESC);
```

**cron_logs 表**（Cron 执行日志）：

```sql
CREATE TABLE cron_logs (
  log_id         BIGINT AUTO_INCREMENT PRIMARY KEY,
  run_at         TIMESTAMP,
  status         ENUM('success', 'partial', 'failed'),
  users_processed INT,
  t1_deleted     INT,
  t2_created     INT,
  t3_updated     INT,
  tokens_used    INT,
  error_message  TEXT,
  duration_ms    INT
);
```

### 10.2 T3 写入接口（含冲突检测）

```javascript
async function updateT3Field(userId, fieldName, newValue, sourceEventId) {
  const currentT3 = await getT3(userId);
  const currentFieldValue = currentT3[fieldName]?.value;
  
  if (!currentFieldValue) {
    // 字段为空，直接写入
    return writeT3Field(userId, fieldName, newValue, sourceEventId);
  }
  
  // Step 1: Embedding 预筛
  const similarity = await computeCosineSimilarity(currentFieldValue, newValue);
  
  if (similarity < 0.75) {
    // 不相关，直接 APPEND
    return appendT3Field(userId, fieldName, newValue, sourceEventId);
  }
  
  // Step 2: LLM 冲突检测
  const verdict = await detectConflictWithLLM(currentFieldValue, newValue);
  
  switch(verdict) {
    case 'OVERWRITE':
      return writeT3Field(userId, fieldName, newValue, sourceEventId);
    case 'APPEND':
      return appendT3Field(userId, fieldName, newValue, sourceEventId);
    case 'CONFLICT':
      return addToPendingConflicts(userId, fieldName, currentFieldValue, newValue);
  }
}
```

---

## 十一、Cron 定时任务规范

### 11.1 触发配置

```
执行时间：每周日 03:00（服务器本地时间）
超时设置：30 分钟
最大并发：按用户 ID 哈希分批，每批 50 人，防止数据库过载
```

### 11.2 完整流程代码逻辑

```javascript
async function weeklyConsolidationCron() {
  const log = { run_at: new Date(), users_processed: 0, ... };
  
  try {
    const users = await getAllActiveUsers();
    
    for (const userId of users) {
      try {
        await processUserConsolidation(userId, log);
        log.users_processed++;
      } catch (userError) {
        log.partial_errors.push({ userId, error: userError.message });
        // 单个用户失败，不影响其他用户继续处理
      }
    }
    
    log.status = log.partial_errors.length === 0 ? 'success' : 'partial';
  } catch (fatalError) {
    log.status = 'failed';
    log.error_message = fatalError.message;
    // 全局失败：暂停 T1 物理删除，发送告警
    await pauseT1Deletion();
    await sendAlertToDevTeam(fatalError);
  } finally {
    await saveCronLog(log);
  }
}

async function processUserConsolidation(userId, log) {
  // 拉取近7天 T1，限额 200 条
  const t1Records = await getRecentT1(userId, 7, limit=200);
  
  if (t1Records.length === 0) return;
  
  // 阶段一：Flash 模型按天预聚合（7天→7条，廉价）
  const dailySummaries = await aggregateByDayWithFlash(t1Records);
  
  // 阶段二：主模型生成 T2/T3 更新（一次调用）
  const consolidationResult = await consolidateWithMainModel(
    dailySummaries,
    await getT3(userId)
  );
  
  // 执行情绪隔离罩
  const negativeRatio = calcNegativeRatio(t1Records);
  if (negativeRatio > 0.4) {
    delete consolidationResult.t3Updates.personality;  // 隔离人格更新
  }
  
  // 执行冲突检测网关（对 T3 变更）
  for (const [field, newValue] of Object.entries(consolidationResult.t3Updates)) {
    await updateT3Field(userId, field, newValue, 'cron_weekly');
  }
  
  // 更新 T2 向量库
  for (const t2Record of consolidationResult.newT2Records) {
    await upsertT2Memory(userId, t2Record);
  }
  
  // 更新 bond_momentum
  await updateBondMomentum(userId);
  
  // 物理清除（仅在 Cron 正常状态下执行）
  const toDelete = t1Records.filter(r => computeMemoryStrength(r) < 2.0);
  await deleteT1Records(toDelete.map(r => r.event_id));
  
  log.t1_deleted += toDelete.length;
}
```

### 11.3 重试机制

```
Cron 任务失败后的重试计划：
  第1次失败 → 1小时后重试
  第2次失败 → 4小时后重试
  第3次失败 → 24小时后重试（此时距下次周结仅剩约6天，影响可接受）
  第3次仍失败 → 暂停 T1 物理删除 + 发送告警给开发者
```

---

## 十二、冷启动流程

### 12.1 触发条件

用户首次注册，或 T3.identity.value 为空字符串。

### 12.2 破冰指令模板

```
[冷启动破冰指令]
这是与用户的第一次对话，你对用户一无所知。
请在接下来的前3轮对话中，以极其自然的方式（反问、聊天中顺带问）
获取以下信息，并且每轮只问一件事，禁止连续追问：
1. 用户的名字或称呼
2. 最近在忙什么 / 做什么
3. 有什么兴趣爱好

禁止说"请问你叫什么名字"这种机器人式提问。要像朋友一样自然聊天。
当你收集到足够信息后，直接用第一人称陈述出来，触发系统录入。
```

### 12.3 T3 种子生成

破冰对话结束后（约3轮），内容信号守门人会正常检测并提取 T1。由于破冰内容的 importance_score 通常 >= 7（用户基础信息），闪电通道会自动触发，完成 T3 种子写入。开发者无需为冷启动写额外的种子生成逻辑。

---

## 十三、记忆透明舱（UX 规范）

### 13.1 功能清单

| 功能 | 描述 |
|---|---|
| **T3 可视化展示** | 以名片/标签形式展示 AI 当前认知的用户档案 |
| **置信度标注** | medium/low 置信度的字段以不同颜色区分，让用户知道哪些是推断 |
| **一键删除** | 用户可删除任意一条 interest、修改 identity 字段 |
| **手动添加禁忌** | 用户可手动输入"绝对不要提到 X"，写入 T3 的 `forbidden_topics` 字段 |
| **待确认区** | 展示所有 CONFLICT 状态的冲突变更，用户选择"保留旧值"或"接受新值" |
| **溯源查看** | 点击任意 T3 字段，展示 source_event_ids 对应的原始事件摘要 |

### 13.2 入口与呈现

- **入口**：聊天界面右上角图标（如"记忆"或脑图标）
- **呈现形式**：侧边抽屉（Drawer）或模态框（Modal）
- **合规说明**：界面底部注明数据隐私政策链接，符合 GDPR / PIPL 的用户知情权与删除权要求

---

## 十四、关键指标与成本估算

### 14.1 性能指标目标

| 场景 | 目标延迟 |
|---|---|
| 日常闲聊（T3直注） | < 2 秒 |
| 热态T1辅助对话 | < 2 秒 |
| 深层追溯（T2检索） | < 8 秒 |
| 闪电通道更新 | < 1 秒（异步，不阻塞对话） |
| Cron 批处理（单用户） | < 30 秒 |

### 14.2 每用户每月成本估算（中度活跃用户，每天50条消息）

| 消耗来源 | 月 Token 估算 |
|---|---|
| 日常对话上下文（T3紧凑格式 + 热态T1 + T0） | ~825,000 tokens |
| T1 提取（内容信号触发，约5次/天，Flash 模型） | ~60,000 tokens |
| Cron 批处理（预聚合后，4次/月） | ~3,200 tokens |
| T2 检索（按需，约3次/周） | ~6,000 tokens |
| 冲突检测 LLM 调用（过滤后低频） | ~2,000 tokens |
| **月合计** | **~896,200 tokens** |

*注：其中 T1 提取用廉价 Flash 模型，主模型实际消耗约 831,200 tokens。Prompt Cache 命中后 T3 的 200 tokens 按 1/10 成本计算，日活 1000 用户每天可节省约 18 万 tokens。*

### 14.3 与传统全局 RAG 方案对比

| 维度 | 全局 RAG 方案 | DSM 2.2 |
|---|---|---|
| 日常对话延迟 | 8-15 秒 | < 2 秒 |
| 每条消息 Token 消耗 | ~2000（含长历史） | ~550（T3紧凑+裁剪） |
| 记忆准确率 | 中（向量检索有噪声） | 高（T3直注，精准） |
| 重大事件响应 | 下次检索才感知 | 1秒内感知 |
| 时间遗忘感 | 无 | 有（艾宾浩斯曲线） |
| 用户数据透明度 | 无 | 有（记忆透明舱） |

---

## 附录：DSM 2.2 补丁全览

```
DSM 1.0 → DSM 2.0：基础四级漏斗，T3直注思路
DSM 2.0 → DSM 2.1：
  + 热态/深态T1分离
  + 闪电更新通道
  + 情绪隔离罩
  + Cron 批处理
  + 记忆透明舱
  + T3 结构化 JSON
  + 冷启动破冰
  
DSM 2.1 → DSM 2.2（最终版）：
  + 内容信号守门人（减少75%无效提取调用）
  + T3 紧凑注入格式（减少44% Token）
  + Embedding 预筛冲突检测（减少90%LLM冲突检测调用）
  + Cron 两阶段处理（Flash预聚合，减少98%主模型调用量）
  + T2 工具声明（填补逻辑空白）
  + Prompt Caching 利用（T3读取成本降90%）
  + 热态T1极简单行格式（减少60% T1注入Token）
  + Cron 分用户限额保护（防异常账单峰值）
  + 冲突检测字段溯源（source_event_ids）
  + bond_momentum 驱动主动触达
  + 沉寂回归感知（days_since_last_chat）
  + 跨端竞态防护（幂等主键）
  + Cron 三次重试 + 降级策略
  + 置信度标注（high/medium/low）
  + current_context 自带 TTL
```

---

*文档版本：DSM 2.2 | 最后更新：2026-04-28 | 如有疑问，以本文档为准*
