# Claude Code `/advisor` 功能调研

> 调研对象：Anthropic Claude 的 **Advisor（顾问）工具**及其在 Claude Code 中的 `/advisor` 命令  
> 调研时间：2026-10-08  
> 信息来源：Claude Code 官方文档、Claude Platform API 文档、Anthropic 官方博客及公开报道（详见文末来源）

---

## 0. 摘要（TL;DR）

- **Advisor 的本质**：让执行任务的主模型（executor）在**关键决策点**咨询一个能力更强（通常更贵）的第二模型（advisor），拿到"计划 / 纠偏 / 停止信号"后继续干活。
- **它反转了主流多模型编排范式**：传统做法是"大模型当 orchestrator、向下分解委派给小模型 worker"；Advisor 是"小模型独立端到端执行、遇卡点向上求助"。前者把贵模型的 token 消耗铺满全程，后者只在少数分叉点用贵模型。
- **核心卖点**：用接近 Sonnet 的成本，拿到接近 Opus 的智能。
- **官方实测（2026-04 公布）**：
  - SWE-bench Multilingual：Sonnet 4.6 单跑 72.1% → Sonnet 4.6 + Opus 顾问 **74.8%（+2.7pp）**，且**单任务成本降 11.9%**；
  - BrowseComp：Haiku 4.5 单跑 19.7% → Haiku 4.5 + Opus 顾问 **41.2%（翻倍以上）**，比 Sonnet 单跑低 29% 分，但**成本低 85%**；
  - Terminal-Bench 2.0：同样有提升且成本更低。
- **两种使用形态**：
  1. **Claude Code 侧**：`/advisor [model|off]` 斜杠命令（会话内交互），或 `advisorModel` 设置、`--advisor` 启动参数；
  2. **API 侧**：`advisor_20260301` 服务器端工具（server tool），加进 Messages API 的 `tools` 数组即可，握手在单次 `/v1/messages` 请求内完成。
- **关键约束**：**仅限 Anthropic 官方 API**（Bedrock / AWS / Google Cloud Agent Platform / Microsoft Foundry 均不可用），且官方标注为 **experimental（实验性）**。

---

## 1. 背景与动机

### 1.1 要解决的问题：能力与成本的两难

编码 Agent、计算机操作、多步研究管线这类**长程 agentic 任务**有一个结构性特征：

> **绝大多数轮次是机械执行，少数轮次的"计划质量"决定成败。**

- 全程用最强模型（Opus）：计划质量有保障，但贵——把溢价花在了大量不需要深度推理的例行轮次上。
- 全程用便宜模型（Haiku / Sonnet）：省钱，但在架构决策、反复报错的调试、模糊需求这类"硬分叉"上容易翻车。
- 工程上通常靠"人工预先设计模型路由"来平衡，但这本身就是个麻烦且易错的活。

### 1.2 既有范式：orchestrator → sub-agent（向下委派）

主流多模型编排是**大模型当指挥、小模型当工人**：

```
       ┌─────────────┐
       │  大模型      │  ← orchestrator，全程参与
       │ (Opus)      │     负责读上下文、做计划、拆分任务、汇总结果
       └──────┬──────┘
              │ 分解 + 委派
     ┌────────┼────────┐
     ▼        ▼        ▼
  ┌─────┐  ┌─────┐  ┌─────┐
  │小模型│  │小模型│  │小模型│  ← workers，只负责执行子任务
  └─────┘  └─────┘  └─────┘
```

问题：**orchestrator 连琐碎决策都要消耗顶级模型 token**，成本被摊薄得很难看。

### 1.3 Advisor 范式：executor 向上咨询（反转）

Advisor 策略把方向反过来——**廉价、快速的模型是主角，贵模型是被按需调用的"军师"**：

```
   ┌──────────────────────────────┐
   │  Executor（Sonnet / Haiku）   │
   │  端到端驱动整个任务：          │
   │  调工具 → 读结果 → 迭代        │
   └───────┬──────────────────────┘
           │ 只在"卡点"时求助
           │ (确定方案前 / 反复报错 / 宣布完成前)
           ▼
   ┌──────────────────────────────┐
   │  Advisor（Opus / Fable）      │
   │  读共享上下文 → 返回：          │
   │  计划 / 纠偏 / 停止信号         │
   └───────┬──────────────────────┘
           │ 指导回传
           ▼
     Executor 带着建议继续执行
```

Anthropic 官方原话：Advisor "reverses a common sub-agent pattern"，小模型无需 decomposition、无需 worker pool、无需 orchestration 逻辑，就能独立驱动并向上升级。**frontier 级推理只在需要时介入，其余全程保持在 executor 档位的成本。**

### 1.4 时间线

| 时间         | 事件                                                                                                           |
| ---------- | ------------------------------------------------------------------------------------------------------------ |
| 2026-04-09 | Anthropic 在 Claude Platform API 正式发布 **Advisor 策略 / advisor 工具**（beta），HTTP beta 头 `advisor-tool-2026-03-01` |
| v2.1.98 起  | Claude Code 引入 `/advisor` 斜杠命令（不同资料记载在 v2.1.98–v2.1.101 之间；`-p` 非交互模式、Agent SDK、桌面端、远程控制等接口需 **v2.1.260+**）  |
| v2.1.170 起 | 支持 Fable 5 作为主模型 / 顾问（需 Fable 5 访问权限）                                                                        |

> ⚠️ 两个版本号存在资料口径差异，引用时建议以官方文档改版为准。

---

## 2. 功能（Claude Code 侧）

### 2.1 是什么

`/advisor` 在 Claude Code 会话里开启"顾问工具"：主模型在任务关键节点自动咨询一个**能力不低于自己**的第二模型，并在继续前应用其指导。

- 顾问**接收完整对话**（含每一次工具调用与结果）；
- **由主模型自主决定何时调用**（model-driven，非固定规则）；
- 会话启动后会出现常驻提示：`Advisor Tool (experimental) is on and may use more tokens`。

### 2.2 三种启用方式

| 方式   | 写法                           | 作用范围        | 说明                                           |
| ---- | ---------------------------- | ----------- | -------------------------------------------- |
| 斜杠命令 | `/advisor opus`              | 当前会话 + 存为默认 | 不带参数则打开选择器；结果保存到用户设置 `advisorModel`，跨会话保留    |
| 设置项  | `{ "advisorModel": "opus" }` | 持久默认        | 写进 settings 文件，不启动会话也能配                      |
| 启动参数 | `claude --advisor opus`      | 仅单次会话       | 覆盖 `advisorModel`；**不出现在 `claude --help` 中** |

- 支持别名 `fable` / `opus` / `sonnet`（解析为该系列当前默认版本），也可传完整模型 ID，如 `claude-opus-5`。
- 也可在**无终端选择器**的界面使用：`-p` 非交互模式、Agent SDK、桌面 App、远程控制（需 CLI v2.1.260+）。这些界面上不带参数的 `/advisor` 打印当前顾问，`/advisor off` 关闭。
- 若配置的顾问被组织 `availableModels` 白名单排除，则不会被调用，需用 `/advisor` 改选允许的模型；若当前主模型不支持顾问，选择仍会保存，待 `/model` 切到兼容主模型后生效。

### 2.3 模型配对规则（关键）

**硬规则：顾问能力必须 ≥ 主模型；Haiku 可以"调用"顾问，但永远不能"充当"顾问。**

| 主模型                | 接受的顾问                 | 备注                                                       |
| ------------------ | --------------------- | -------------------------------------------------------- |
| Haiku 4.5          | Fable, Opus, Sonnet   | Haiku 可调用顾问，不能当顾问                                        |
| Sonnet 4.6         | Fable, Opus, Sonnet   |                                                          |
| Sonnet 5           | Fable, Opus, Sonnet 5 | **不接受** Sonnet 4.6 顾问                                    |
| Opus 4.6           | Fable, Opus, Sonnet 5 | Sonnet 5 与 Opus 4.6 能力相当，故可互为配对                          |
| Opus 4.7 或更高       | Fable，以及 Opus 4.7 或更高 | 同级 Opus 之间可互为顾问；Opus 4.7 主模型配 Opus 4.6 / Sonnet 5 顾问会被拒绝 |
| Fable 5（v2.1.170+） | Fable                 | 不接受 Opus / Sonnet 顾问；当前 Fable 暂不作为顾问提供                   |

**常用配对及适用场景：**

| 配对                   | 适用                                                      |
| -------------------- | ------------------------------------------------------- |
| Sonnet 主 + Opus 顾问   | 首选。Sonnet 干活，规划、模糊故障、完成前检查升级给 Opus                      |
| Sonnet 主 + Fable 顾问  | 决策点借 Fable 5 指导，不必全程跑 Fable 5（需 v2.1.170+ 及 Fable 5 权限） |
| Haiku 主 + Opus 顾问    | 成本最低的主模型 + 强规划。比 Haiku 单跑贵，但比换 Sonnet/Opus 便宜           |
| Opus 主 + Opus 顾问     | 第二个 Opus 审查第一个 Opus，用于独立检查比成本更关键的任务                     |
| Sonnet 主 + Sonnet 顾问 | 低成本第二意见，用于捕捉常规疏忽                                        |

> 子智能体（subagent）**继承**已配置的顾问，但会**按自身模型重跑配对检查**——所以在弱模型上跑的子智能体，即使主会话有顾问，它也可能没有。

### 2.4 会话内你会看到什么

调用顾问时，transcript 显示一行 `Advising`（附顾问模型名）；返回后变成三种状态之一：

| 状态              | 含义                                                                   |
| --------------- | -------------------------------------------------------------------- |
| **Reviewed**    | 顾问读完了对话并给出指导。按 `Ctrl+O` 阅读全文                                         |
| **Declined**    | "Advisor declined to advise on this request."（拒绝给出建议），按 `Ctrl+O` 看原因 |
| **Unavailable** | 调用失败，显示 "Advisor unavailable" 及错误码                                   |

Claude 通常遵循顾问建议，但**不盲从**：若建议的步骤实测失败，或文件内容与建议相悖，它会**抛出一处冲突**而非无条件执行。

### 2.5 关闭方式

```bash
/advisor off                     # 停止使用并清除保存的 advisorModel
export CLAUDE_CODE_DISABLE_ADVISOR_TOOL=1   # 彻底禁用（硬开关）
```

设了环境变量后：`/advisor` 不可用、`advisorModel` 被忽略、`--advisor` 被接受但无效果（现有脚本不会报错）。

---

## 3. 工作原理

### 3.1 何时调用：模型驱动，而非规则驱动

- **没有**任何"强制/限制调用次数"的设置。**由主模型自主判断**；官方文档描述其时机为"model-driven rather than rule-based"。
- 经验性触发点：**确定方案之前、错误反复出现时、宣布任务完成之前**。
- 想让它多问 / 少问，直接在 prompt 里说，例如：`在继续之前咨询顾问`。

Claude Code 自带的使用规则（由模型自述）大概长这样：

1. **做实质工作前调用**——只读文件之类轻量调查可以先做，但最好在写、改、或下结论之前调；
2. **完成前保存交付物**——确保顾问调用期间即使会话结束，工作成果仍在；
3. **认真对待建议**——若照做后实测失败、或与本地证据冲突，**再调一次顾问**来消解矛盾；
4. **长任务多次调用**——经验值：锁定方案前一次、宣布完成前一次。

### 3.2 上下文传递：全量、单向、只回文本

- 顾问**接收完整对话**：系统提示、工具定义、历史轮次与工具结果、本轮已产出的文本。
- 顾问**自己不调用工具、不做上下文管理**；
- 顾问的 **thinking 块在返回前被丢弃**，只有建议正文到达 executor；
- 因此顾问是"只读共享上下文 → 产出一段短指导"的角色，**不产生用户可见输出**。

### 3.3 典型调用长什么样

以某次 Svelte 重构为例：主模型（Sonnet）计划把 `Card.svelte` 中混在一起的三块职责拆出去。调用顾问后，Opus 读完整上下文，指出 Sonnet 计划里**遗漏的三个风险**：

- `$paraglide/messages` 别名在 `.ts` 文件里能否解析存疑 → 建议改为**参数传入**；
- 原代码用全局 `document.querySelectorAll` → 应改为 **`node.querySelectorAll`**（这本就是重构的核心）；
- `destroy` 里只删了事件监听、没删插入的 DOM → 建议用一个 `inserted` 数组追踪并在销毁时清理。

这正是 Advisor 的价值样本：**顾问看不到新信息，只是在同样的上下文上做了更深的推理。**

### 3.4 对 prompt cache 的影响

- **切换 `/advisor` 不会使主模型的 prompt cache 失效**（不同于改模型或改 effort 级别）——缓存前缀保持完整，顾问返回的指导在后续轮次里也会被缓存。
- **但顾问自己对对话的读取永远不缓存**：每次顾问调用都从头重新处理完整 transcript。

---

## 4. 架构（API 层）

Claude Code 的 `/advisor` 只是上层封装；底层是 Claude Platform 的**服务器端工具（server-side tool）**。

### 4.1 定位：一个"服务器工具"，不是新的编排产品

- 工具类型标识：`advisor_20260301`
- 通过 beta 头启用：`anthropic-beta: advisor-tool-2026-03-01`
- **握手在单次 `/v1/messages` 请求内完成**，客户端**无需额外往返、无需自己序列化状态、无需管理旁路通道**。

### 4.2 请求流程

```
你的应用
   │  POST /v1/messages  (tools 里含 advisor_20260301)
   ▼
Executor（如 claude-sonnet-4-6）
   │  开始任务……
   │  遇到硬决策 → 发出 server_tool_use 块
   │     name = "advisor"，input 为空        ← "时机由 executor 决定"
   ▼
Anthropic 服务端（同一请求内）
   ├─ 由服务端提供上下文（curated context）
   └─ 对 Advisor 模型（如 claude-opus-4-6）跑一次独立推理
        · Advisor 用 Anthropic 自带的 system prompt
        · 输入是 executor 的完整 transcript（作为 quoted context）
        · 不调用工具、不做 context management
        · thinking 块丢弃，只回建议文本
   │  产出 advisor_tool_result 块
   ▼
Executor 拿到建议 → 继续生成
   │
   ▼
返回你的应用（单次响应内完成）
```

### 4.3 结果变体（Result variants）

顾问的返回有两种形态，取决于顾问模型：

| 变体                        | 说明                            | 何时出现                     |
| ------------------------- | ----------------------------- | ------------------------ |
| `advisor_result`          | **明文**建议文本，客户端可直接读            | 顾问用 `claude-opus-4-8` 等时 |
| `advisor_redacted_result` | **加密**（executor 在服务端读，客户端读不到） | 顾问用 `claude-opus-5` 等时   |

> 想在响应里直接看到建议文本，就把顾问设为返回明文的模型（如 `claude-opus-4-8`）。

### 4.4 工具参数

| 参数         | 类型      | 默认 | 说明                                    |
| ---------- | ------- | -- | ------------------------------------- |
| `type`     | string  | 必填 | 固定为 `advisor_20260301`                |
| `name`     | string  | 必填 | 固定为 `advisor`                         |
| `model`    | string  | 必填 | 顾问模型 ID，如 `claude-opus-5`；按此模型费率计费子推理 |
| `max_uses` | integer | 不限 | 单次请求内允许的最大顾问调用次数（**成本控制旋钮**）          |

### 4.5 最小可用示例（Python）

```python
client = anthropic.Anthropic()

response = client.beta.messages.create(
    model="claude-sonnet-5",              # executor
    max_tokens=4096,
    betas=["advisor-tool-2026-03-01"],
    tools=[
        {
            "type": "advisor_20260301",
            "name": "advisor",
            "model": "claude-opus-5",     # advisor
            "max_uses": 3,                # 限制调用次数，控制成本
        },
        # ... 你自己的其他工具照常并存
    ],
    messages=[{"role": "user", "content": "用 Go 实现一个带优雅关闭的并发 worker pool。"}],
)

print(response)   # 响应里带 advisor_tool_result 块
```

> 官方提示：上线前建议把你的既有 eval 套件跑三遍对比——**Sonnet 单跑** / **Sonnet+Opus 顾问** / **Opus 单跑**。

### 4.6 一次会暂停的轮次（Resuming a paused turn）

正常情况下整个流程在**单次请求内**完成，客户端零额外往返。**唯一例外**：某个 turn 在顾问调用中途"暂停"，此时需要用一次后续请求来 **resume**。

### 4.7 数据留存

Advisor 作为服务器端工具运行，ZDR（零数据留存）等条款见官方 API 文档的 *API and data retention* 章节。顾问读取的是你的完整会话内容。

---

## 5. 效果与成本数据（官方公布）

### 5.1 Benchmark

| 评测                     | 配置                           | 结果                | 成本                                  |
| ---------------------- | ---------------------------- | ----------------- | ----------------------------------- |
| SWE-bench Multilingual | Sonnet 4.6 单跑                | 72.1%（基线）         | 基线                                  |
| SWE-bench Multilingual | **Sonnet 4.6 + Opus 4.6 顾问** | **74.8%（+2.7pp）** | **−11.9% / 任务**                     |
| BrowseComp             | Haiku 4.5 单跑                 | 19.7%             | 最低                                  |
| BrowseComp             | **Haiku 4.5 + Opus 4.6 顾问**  | **41.2%（翻倍以上）**   | 比 Sonnet 单跑**低 29% 分**，但**成本低 85%** |
| Terminal-Bench 2.0     | Sonnet 4.6 + Opus 顾问         | 提升                | 低于 Sonnet 单跑                        |

**成本机制**：顾问通常只生成 **400–700 个文本 token 的短计划**，而 executor 以更低费率产出全部正文——因此总成本显著低于"全程跑顾问模型"。

### 5.2 客户证言（节选）

- **Bolt** CEO Eric Simmons：*"It makes better architectural decisions on complex tasks while adding no overhead on simple ones. The plans and trajectories are night and day different."*
- **Genspark** 联合创始人 Kay Zhu：*"We saw clear improvements in agent turns, tool calls, and overall score — better than a planning tool we built ourselves."*
- **Eve Legal** ML 工程师 Anuraj Pandey：*"…matching frontier-model quality at 5× lower cost."*

---

## 6. 与相关功能的对比

Advisor 只是"引入第二个模型"的几种方式之一。**按你希望强模型介入的时机来选：**

| 方法                 | 强模型何时运行                        | 如何触发             |
| ------------------ | ------------------------------ | ---------------- |
| **顾问工具 (Advisor)** | 任务中的**决策点**                    | Claude 需要指导时自行调用 |
| `opusplan`         | **规划模式**下（白名单允许时），执行时切回 Sonnet | 进入 plan mode     |
| 子智能体 (Subagents)   | 整个被委派的**子任务**全程                | Claude 委派，或你手动调用 |
| `/model`           | **所有后续轮次**                     | 你主动切换模型          |


一句话判断：

- 想要"大部分轮次便宜、少数轮次变强" → **Advisor**；
- 想要"计划阶段最强、执行阶段便宜" → **opusplan**；
- 想要"把一整块独立子任务外包给另一个模型" → **Subagent**；
- 想要"之后全都用强模型" → 直接 **`/model`**。

---

## 7. 适用与不适用

### ✅ 适合

- 长时、多步任务，**多数轮次例行、计划质量决定成败**：大型重构、反复报错的调试、宣布完成前的独立复核。
- 高并发 / 大批量任务对成本敏感，又想借到强模型推理：**Haiku + Opus 顾问**（官方点名"high-volume tasks"）。

### ❌ 不适合

- **单轮问答**：没有"计划"可做；
- **纯直通式模型选择器**：用户已自行权衡成本/质量；
- **每一轮都真的需要最强模型**的工作：那应直接换主模型，而不是挂顾问。

---

## 8. 局限与注意事项

1. **仅限 Anthropic 官方 API**：是服务器端工具，**不可用**于 Amazon Bedrock、AWS 上的 Claude Platform、Google Cloud Agent Platform、Microsoft Foundry。走 LLM 网关（`ANTHROPIC_BASE_URL`）时，取决于网关是否把请求**完整**转发到 Anthropic API。
2. **主模型须受支持**：Opus 4.6+、Sonnet 4.6+、Haiku 4.5；Fable 5 需 v2.1.170+ 且当前不接受非 Fable 顾问。
3. **需要 feature-flag 拉取**：若像 `DISABLE_TELEMETRY` 这类变量关掉了 flag 拉取，顾问会**无论怎么配都保持关闭**。
4. **明确标注 experimental**：行为、定价、可用性都可能变。
5. **fallback 陷阱**：主模型 529 后回退到另一模型时，若回退模型不再满足配对表，会话会**静默地无顾问运行**，而**不报错**。
6. **成本是按 token 叠加的**：每次顾问调用都重读完整对话，按顾问模型费率消耗 token（订阅计划计入计划额度；Fable 顾问计入 usage credits）。
7. **子智能体配对可能失效**：子智能体按自身模型重跑检查，弱模型上的子智能体可能无顾问。

---

## 9. 对 Agent 开发者的一点判断（我的观点）

1. **Advisor 真正的新意不在"多模型协作"，而在"控制权归属"**。它是 **executor-first** 而非 **orchestrator-first**：状态机的主循环、工具调用、上下文，全都在便宜模型手里；贵模型被降级成一个"无状态、无工具、只读一次上下文、回一段文本"的纯函数。这个设计让 advisory 几乎**零集成成本**——加一个 tool 条目、一个 beta 头即可，不用重写已有 agent loop。
2. **它是对"路由难题"的一个优雅规避**。以前要手工判断"哪一段该用强模型"，现在把这个判断**下放给 executor 自己**（模型驱动、非规则）。代价是**可预测性下降**：你没法精确控制它调多少次、何时调，只能用 `max_uses` 设上限、用 prompt 引导。对成本极其敏感的场景，这仍是个需要实测校准的黑盒。
3. **最有价值的数字是 Haiku+Opus 那条**：41.2% vs 19.7%，翻倍；同时成本比 Sonnet 单跑还低 85%。这实际上是在说——**在"大量例行 + 少量硬点"的负载上，把 5% 的 token 交给旗舰模型，比把 100% 的 token 交给中端模型更划算。** 这对批处理型 Agent（文档抽取、大规模数据处理）是明确的成本结构重塑。
4. **架构上值得注意的两个细节**：(a) 顾问的 **thinking 被丢弃、只回文本**——意味着推理能力被压缩进一段短指导，信息有损，但换来 token 成本和跨模型兼容性；(b) 结果分**加密 / 明文**两种变体——这暗示 **FRONTIER 模型的思维轨迹被视为需要保护的资产**，未来不同档位顾问的信息可见性可能继续分化，值得持续观察。
5. **移植限制是真实门槛**：服务器端实现 + 仅 Anthropic API，意味着**自建多模型 advisor 要自己实现"完整 transcript → 顾问推理 → 指导回注"这一套**。如果少爷要做类 Advisor 的自研 harness，核心工作量在：上下文裁剪策略、advisor 何时触发的判定、以及 advice 回注时的冲突处理（模型提出建议 vs 本地证据矛盾时如何取舍）。

---

## 10. 参考来源

1. Claude Code 官方文档 — *Escalate hard decisions with the advisor tool*：`https://code.claude.com/docs/en/advisor`
2. Claude Code 官方文档 — *Commands*（`/advisor` 条目）：`https://code.claude.com/docs/en/commands`
3. Claude Platform API 文档 — *Advisor tool*：`https://platform.claude.com/docs/en/agents-and-tools/tool-use/advisor-tool`
4. Anthropic 官方博客 — *The advisor strategy: Give agents an intelligence boost*：`https://claude.com/blog/the-advisor-strategy`
5. claudcod.com — *Claude Code Advisor Tool: How to Set Up /advisor*（2026-09-29）
6. azukiazusa.dev — *Using Claude's Advisor Strategy to Optimize the Balance Between Performance and Cost*（2026-04-11）
7. 中文镜像文档 — `https://claudecode.ac.cn/docs/en/advisor`

> 说明：部分二手资料对 `/advisor` 引入版本号（v2.1.98 / v2.1.101）记载不一，本文已标注差异；接口与模型能力以官方文档为准，**上线前请复核最新版本**。

---

*文档由 YY 整理 ⚡ | 2026-10-08*
