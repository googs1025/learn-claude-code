[English](./README.md) | [中文](./README-zh.md) | [日本語](./README-ja.md)

# 项目功能详细说明

> 本文件对 **learn-claude-code** 仓库的作用、模块与功能进行完整梳理。

---

## 一、项目定位

**learn-claude-code** 是一个面向开发者的**教学型项目**，目标是从零开始、逐步构建一个类似 Claude Code 的 AI 编程智能体（Agent）。

- **受众**：想深入理解 AI Agent 内部机制的工程师、研究者或学习者
- **语言**：Python（后端 Agent 实现）+ TypeScript/Next.js（交互式学习平台）
- **模型**：使用 Anthropic Claude API（可替换为其他兼容模型）
- **范式**：逐课递进，共 12 个课程（Session），每课只增加一个机制，保持核心循环不变

---

## 二、核心模式：Agent 循环

整个项目围绕同一个最小循环展开：

```
User --> messages[] --> LLM --> response
                                   |
                         stop_reason == "tool_use"?
                        /                          \
                      yes                           no
                       |                             |
                 execute tools                    return text
                 append results
                 loop back -----------------> messages[]
```

```python
def agent_loop(messages):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM,
            messages=messages, tools=TOOLS,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = TOOL_HANDLERS[block.name](**block.input)
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
```

**关键点**：每个课程都在这个循环之上叠加一个新机制，循环本身始终保持不变。

---

## 三、目录结构说明

```
learn-claude-code/
│
├── agents/                  # Python Agent 参考实现（核心代码）
│   ├── s01_agent_loop.py        # 课程1：最简 Agent 循环
│   ├── s02_tool_use.py          # 课程2：多工具 dispatch
│   ├── s03_todo_write.py        # 课程3：TodoWrite 计划机制
│   ├── s04_subagent.py          # 课程4：子智能体（独立上下文）
│   ├── s05_skill_loading.py     # 课程5：按需加载技能知识
│   ├── s06_context_compact.py   # 课程6：三层上下文压缩
│   ├── s07_task_system.py       # 课程7：持久化任务图系统
│   ├── s08_background_tasks.py  # 课程8：后台任务（守护线程）
│   ├── s09_agent_teams.py       # 课程9：多智能体团队
│   ├── s10_team_protocols.py    # 课程10：关机/计划审批协议
│   ├── s11_autonomous_agents.py # 课程11：自治任务认领
│   ├── s12_worktree_task_isolation.py  # 课程12：Worktree 任务隔离
│   ├── s_full.py                # 总纲：全部机制合一的完整实现
│   └── __init__.py
│
├── docs/                    # 文档（3 种语言）
│   ├── en/                      # 英文文档
│   ├── zh/                      # 中文文档
│   └── ja/                      # 日文文档
│
├── web/                     # 交互式学习平台（Next.js）
│   ├── src/
│   │   ├── app/                 # Next.js App Router 页面
│   │   ├── components/          # UI 组件（可视化、模拟器、文档等）
│   │   ├── data/                # 课程数据（scenarios、annotations、生成内容）
│   │   ├── hooks/               # React Hooks（模拟器、可视化步进等）
│   │   ├── i18n/                # 国际化（英/中/日）
│   │   └── lib/                 # 工具函数与常量
│   └── scripts/
│       └── extract-content.ts   # 从 agents/ 和 docs/ 中提取内容生成静态数据
│
├── skills/                  # 按需加载的技能文件（供 s05 使用）
│   ├── agent-builder/           # 设计和构建 AI Agent 的知识
│   ├── code-review/             # 代码审查流程与清单
│   ├── mcp-builder/             # MCP 服务器构建指南
│   └── pdf/                     # PDF 文件处理方法
│
├── .env.example             # 环境变量示例（ANTHROPIC_API_KEY）
├── requirements.txt         # Python 依赖（anthropic, python-dotenv）
├── README.md                # 英文主文档
├── README-zh.md             # 中文主文档
├── README-ja.md             # 日文主文档
└── .github/workflows/       # CI 配置（类型检查 + 构建）
```

---

## 四、各课程功能详解

### 阶段一：循环基础

#### s01 — Agent 循环（The Agent Loop）
- **格言**：*One loop & Bash is all you need*
- **工具数**：1（`bash`）
- **功能**：
  - 实现最简 Agent 主循环（`while stop_reason == "tool_use"`）
  - 提供 `bash` 工具，让 LLM 执行 shell 命令
  - 内置危险命令过滤（防止 `rm -rf /`、`sudo` 等）
  - 超时保护（120 秒）
  - 命令输出截断（50,000 字符）
- **核心洞察**：一个退出条件 + 一个 bash 工具 = 能实际做事的智能体

#### s02 — 工具使用（Tool Use）
- **格言**：*Adding a tool means adding one handler*
- **工具数**：4（`bash`、`read_file`、`write_file`、`edit_file`）
- **功能**：
  - 引入工具分发表（dispatch map）：`{tool_name: handler_function}`
  - 新增 `read_file` 工具：带路径沙箱（sandbox），限制在工作目录内
  - 新增 `write_file` 工具：安全写入文件
  - 新增 `edit_file` 工具：字符串替换式精确编辑
  - 路径安全校验：防止路径穿越（`../../../etc/passwd`）
- **核心洞察**：扩展工具不需要改循环，只需在 dispatch map 中注册新 handler

---

### 阶段二：规划与知识

#### s03 — 待办写入（TodoWrite）
- **格言**：*An agent without a plan drifts*
- **工具数**：5（新增 `todo_write`）
- **功能**：
  - 引入 `TodoManager` 内存状态，管理任务列表
  - 任务状态：`pending`（待做）/ `in_progress`（进行中）/ `completed`（完成）
  - `todo_write` 工具支持创建、更新任务
  - **Nag 提醒机制**：若 3 轮内未更新 TODO，自动在下次工具结果中注入提醒
  - 系统提示要求 Agent 先列计划，再逐步执行
- **核心洞察**：显式计划让完成率翻倍；Nag 机制对抗上下文稀释

#### s04 — 子智能体（Subagents）
- **格言**：*Break big tasks down; each subtask gets a clean context*
- **工具数**：5（新增 `task` 工具）
- **功能**：
  - 父 Agent 通过 `task` 工具派发子任务
  - 子 Agent 拥有独立的、全新的 `messages[]` 数组
  - 子 Agent 完成后只返回摘要文本，所有中间过程不污染父上下文
  - 子 Agent 不能再生成子 Agent（防止递归）
  - 父 Agent 上下文保持整洁
- **核心洞察**：上下文隔离是子 Agent 的核心价值；摘要 ≪ 完整工具调用历史

#### s05 — 技能加载（Skill Loading）
- **格言**：*Load knowledge when you need it, not upfront*
- **工具数**：5（新增 `load_skill`）
- **功能**：
  - 系统提示中仅列出技能名称和简短描述（~100 tokens/技能）
  - `load_skill` 工具按需加载完整 `SKILL.md` 文件（~2000 tokens）
  - 技能内容通过 `tool_result` 注入，不占用永久 system prompt
  - 内置技能：`agent-builder`（Agent 设计）、`code-review`（代码审查）、`mcp-builder`（MCP 服务器）、`pdf`（PDF 处理）
- **核心洞察**：知识按需注入，避免 system prompt 臃肿

#### s06 — 上下文压缩（Context Compact）
- **格言**：*Context will fill up; you need a way to make room*
- **工具数**：5（新增 `compact` 工具）
- **功能**：**三层压缩策略**
  - **第一层（微压缩，每轮静默）**：将 3 轮前的工具调用结果替换为 `[Previous: used {tool_name}]`
  - **第二层（自动压缩，token 超阈值时）**：若上下文超过 50,000 tokens，调用 LLM 生成摘要，替换 messages 历史，仅保留摘要
  - **第三层（手动压缩）**：用户可随时调用 `compact` 工具触发完整上下文总结
  - 压缩时保留系统状态（task list、团队状态等）
- **核心洞察**：三层递进压缩实现无限长会话

---

### 阶段三：持久化

#### s07 — 任务系统（Task System）
- **格言**：*Break big goals into small tasks, order them, persist to disk*
- **工具数**：8（新增 `task_create`、`task_get`、`task_update`、`task_list`）
- **功能**：
  - 将内存中的扁平 TODO 升级为**磁盘持久化任务图（DAG）**
  - 每个任务存为独立 JSON 文件（`.tasks/task_N.json`）
  - 任务字段：`id`、`title`、`description`、`status`、`blockedBy`（前置依赖）、`blocks`（后置依赖）
  - `task_create`：创建任务，支持指定依赖关系
  - `task_get`：获取单个任务详情
  - `task_update`：更新任务状态，完成时自动解锁后续任务
  - `task_list`：列出所有可执行（未阻塞）任务
  - 上下文压缩后任务状态不丢失（磁盘持久化）
- **核心洞察**：任务图回答"现在能做什么"，是多 Agent 协作的基础

#### s08 — 后台任务（Background Tasks）
- **格言**：*Run slow operations in the background; the agent keeps thinking*
- **工具数**：6（新增 `background_run`、`background_check`）
- **功能**：
  - 引入守护线程（daemon thread）运行耗时命令
  - `background_run`：在后台线程中启动 subprocess，立即返回任务 ID
  - 后台线程完成后将结果推入**通知队列**（Queue）
  - 每次 LLM 调用前，自动排空通知队列，将完成通知注入消息
  - `background_check`：主动查询某个后台任务的状态/输出
  - 支持并行运行多个后台任务（`npm install` + `docker build` 同时进行）
- **核心洞察**：非阻塞 I/O 让 Agent 在等待慢操作时继续推进其他工作

---

### 阶段四：团队协作

#### s09 — 智能体团队（Agent Teams）
- **格言**：*When the task is too big for one, delegate to teammates*
- **工具数**：9（新增 `spawn_teammate`、`send_message`、`read_inbox`、`list_teammates`）
- **功能**：
  - 引入**持久化队友**概念：队友有身份、跨调用存活
  - 队友配置存储在 `.team/config.json`（名称、状态、创建时间）
  - **JSONL 邮箱通信**：每个队友有独立的 `.team/inbox/{name}.jsonl` 邮箱
  - `spawn_teammate`：创建新队友（在独立线程中运行独立 Agent 循环）
  - `send_message`：向队友邮箱追加消息
  - `read_inbox`：读取并清空自己的邮箱
  - `list_teammates`：查看所有队友状态（`working` / `idle`）
  - `broadcast`：向所有队友广播消息
  - 队友生命周期：`spawn → WORKING → IDLE → WORKING → ... → SHUTDOWN`
- **核心洞察**：JSONL 追加写邮箱是无锁异步通信的最简实现

#### s10 — 团队协议（Team Protocols）
- **格言**：*Teammates need shared communication rules*
- **工具数**：12（新增 `shutdown_teammate`、`plan_approval_request`、关机响应处理）
- **功能**：**两个结构化握手协议**
  - **关机协议**：
    - 领导发送 `shutdown_request`（携带唯一 `request_id`）
    - 队友收到后收尾工作，发送 `shutdown_response`（引用同一 `request_id`，含 `approve: true/false`）
    - 防止直接杀线程导致的文件损坏和状态不一致
  - **计划审批协议（Plan Approval FSM）**：
    - 队友执行高风险任务前，发送 `plan_approval_request` 给领导
    - 领导审批（`approve: true`）或拒绝（`approve: false`），附带原因
    - 队友根据响应决定继续或调整方案
  - 所有协议消息格式统一：`{type, req_id, from, to, timestamp, ...payload}`
- **核心洞察**：request_id 握手模式驱动所有双方协商

#### s11 — 自治智能体（Autonomous Agents）
- **格言**：*Teammates scan the board and claim tasks themselves*
- **工具数**：14（新增 `idle`、`claim_task`）
- **功能**：
  - 队友**主动扫描任务看板**，无需领导逐个分配
  - `claim_task`：原子性认领一个 `pending` 且未阻塞的任务
  - **空闲循环（Idle Cycle）**：队友无任务时不退出，而是进入轮询等待
    - 调用 `idle` 工具后等待 `POLL_INTERVAL` 秒
    - 再次扫描任务看板，有任务则认领执行，无任务则继续等待
    - 收到关机请求时退出空闲循环
  - **身份重注入**：上下文压缩后，自动在系统提示中重新注入队友身份信息（名称、角色）
  - `IDLE_TIMEOUT`：超时自动退出，防止僵尸进程
- **核心洞察**：自组织任务认领是真正"自治"的关键；身份重注入解决压缩后遗忘问题

#### s12 — Worktree 任务隔离（Worktree + Task Isolation）
- **格言**：*Each works in its own directory, no interference*
- **工具数**：16（新增 `worktree_create`、`worktree_list`、`worktree_merge`）
- **功能**：
  - 将任务管理（`.tasks/`）与执行隔离（`.worktrees/`）关联
  - 每个任务绑定一个独立的 **git worktree** 目录
  - **控制平面**（`.tasks/`）：管理"做什么"
    - 任务 JSON 中新增 `worktree` 字段，记录对应 worktree 名称
  - **执行平面**（`.worktrees/`）：管理"在哪做"
    - 每个 worktree 是独立的 git branch（`wt/{name}`）
    - `index.json`：worktree 注册表（ID、任务绑定、branch、状态）
    - `events.jsonl`：生命周期事件流（append-only，仅用于教学）
  - `worktree_create`：创建 worktree 并与任务 ID 绑定
  - `worktree_list`：列出所有 worktree 及其关联任务
  - `worktree_merge`：完成后合并 worktree 分支
  - 多智能体并行修改不同目录，互不污染，支持独立回滚
- **核心洞察**：任务 ID 是控制平面与执行平面的唯一绑定键

---

### 综合实现

#### s_full — 完整参考实现（Capstone）
- 将 s01–s11 所有机制合并为单一文件
- 所有工具（约 20+）可在同一 Agent 中使用
- REPL 特殊命令：`/compact`（手动压缩）、`/tasks`（查看任务）、`/team`（查看团队）、`/inbox`（查看邮箱）
- 每次 LLM 调用前自动执行：微压缩、排空后台通知、检查邮箱
- 注意：s12（Worktree 隔离）作为独立教学内容单独保留

---

## 五、Web 平台功能

交互式学习平台（Next.js + TypeScript），访问地址 `http://localhost:3000`：

| 页面/组件 | 路径 | 功能说明 |
|-----------|------|----------|
| 首页 | `/` | 课程导航，支持语言切换（英/中/日） |
| 课程页 | `/[locale]/(learn)/[version]` | 每课的文档 + 可视化 + 源码查看 |
| 版本对比 | `/[locale]/(learn)/[version]/diff` | 当前课程与上一课程的代码 diff |
| 课程对比 | `/[locale]/(learn)/compare` | 并排比较两个课程的实现 |
| 时间轴 | `/[locale]/(learn)/timeline` | 12 课程演进时间线 |
| 层次图 | `/[locale]/(learn)/layers` | Agent 机制分层架构图 |

### 核心 UI 组件

| 组件 | 位置 | 功能 |
|------|------|------|
| `AgentLoopSimulator` | `components/simulator/` | 可交互的 Agent 循环模拟器（步进执行） |
| `ArchDiagram` | `components/architecture/` | Agent 架构图（静态可视化） |
| `ExecutionFlow` | `components/architecture/` | 执行流程图 |
| `MessageFlow` | `components/architecture/` | 消息流图 |
| `SourceViewer` | `components/code/` | 课程源代码高亮查看器 |
| `CodeDiff` | `components/diff/` | 基于 `diff` 库的代码对比 |
| `WhatsNew` | `components/diff/` | 每课新增机制说明 |
| `DocRenderer` | `components/docs/` | Markdown 文档渲染（remark/rehype） |
| `Timeline` | `components/timeline/` | 课程时间轴 |
| `s01`–`s12` Visualizations | `components/visualizations/` | 每课专属的动画可视化组件 |

### 数据层

| 文件 | 说明 |
|------|------|
| `data/scenarios/s0N.json` | 每课的模拟场景步骤数据 |
| `data/annotations/s0N.json` | 代码注释/标注数据 |
| `data/generated/docs.json` | 从 `docs/` 提取的文档内容（构建时生成） |
| `data/generated/versions.json` | 从 `agents/` 提取的源码版本（构建时生成） |
| `data/execution-flows.ts` | 执行流程定义（TypeScript） |

### 国际化支持

- 支持语言：**英文**（`en`）、**中文**（`zh`）、**日文**（`ja`）
- URL 格式：`/{locale}/...`（如 `/zh/s01`、`/en/s01`、`/ja/s01`）
- 翻译文件：`src/i18n/messages/{en,zh,ja}.json`

---

## 六、技能系统（Skills）

技能系统为 s05 课程提供可按需加载的专业知识，每个技能是一个 `SKILL.md` 文件：

| 技能 | 目录 | 适用场景 | 主要内容 |
|------|------|----------|----------|
| `agent-builder` | `skills/agent-builder/` | 设计和构建 AI Agent | 核心哲学、三要素（能力/知识/上下文）、复杂度递进、反模式 |
| `code-review` | `skills/code-review/` | 代码审查 | 安全、正确性、性能、可维护性、测试五维度检查清单 |
| `mcp-builder` | `skills/mcp-builder/` | 构建 MCP 服务器 | Python/TypeScript 模板、外部 API 集成、数据库访问 |
| `pdf` | `skills/pdf/` | PDF 文件处理 | 读取、创建、合并、拆分 PDF 的工具与代码示例 |

`agent-builder` 技能还附带参考文件（`skills/agent-builder/references/`）：
- `agent-philosophy.md`：Agent 设计哲学深度解析
- `minimal-agent.py`：~80 行的完整最小 Agent 实现
- `subagent-pattern.py`：子 Agent 上下文隔离模式
- `tool-templates.py`：工具定义模板

以及脚手架脚本（`skills/agent-builder/scripts/init_agent.py`）：自动生成新 Agent 项目结构。

---

## 七、配置与环境

### 环境变量（`.env`）

| 变量 | 说明 | 必填 |
|------|------|------|
| `ANTHROPIC_API_KEY` | Anthropic API 密钥 | 是 |
| `MODEL_ID` | 使用的模型 ID（如 `claude-opus-4-5`） | 是 |
| `ANTHROPIC_BASE_URL` | 自定义 API 地址（可替换为兼容端点） | 否 |

### Python 依赖

```
anthropic>=0.25.0    # Claude API 客户端
python-dotenv>=1.0.0 # 环境变量加载
```

### Web 平台主要依赖

| 包 | 用途 |
|----|------|
| `next` 16 | Web 框架（App Router） |
| `react` 19 | UI 框架 |
| `framer-motion` | 动画（可视化组件） |
| `remark`/`rehype` | Markdown 解析与渲染 |
| `diff` | 代码 diff 对比 |
| `lucide-react` | 图标库 |
| `tailwindcss` | 样式框架 |

---

## 八、CI / 自动化

`.github/workflows/ci.yml`：
- 触发条件：push / pull request
- 执行内容：TypeScript 类型检查（`tsc --noEmit`）+ Next.js 构建（`npm run build`）

`.github/workflows/test.yml`：
- 执行内容：Web 平台测试流程

---

## 九、快速启动

### 运行 Python Agent

```sh
git clone https://github.com/shareAI-lab/learn-claude-code
cd learn-claude-code
pip install -r requirements.txt
cp .env.example .env   # 填入 ANTHROPIC_API_KEY 和 MODEL_ID

python agents/s01_agent_loop.py       # 从第一课开始
python agents/s12_worktree_task_isolation.py  # 最后一课
python agents/s_full.py               # 全机制综合版
```

### 运行 Web 平台

```sh
cd web
npm install
npm run dev   # http://localhost:3000
```

---

## 十、项目边界说明

本项目有意**简化或省略**以下生产级机制（保持教学路径清晰）：

- 完整事件/Hook 总线（如 `PreToolUse`、`SessionStart/End`、`ConfigChange`）
- 基于规则的权限治理与信任流程
- 会话生命周期控制（`resume`/`fork`）与完整 worktree 生命周期
- 完整 MCP 运行时细节（transport/OAuth/资源订阅/轮询）
- 团队 JSONL 邮箱协议是教学实现，不代表任何特定生产系统的内部实现

---

## 十一、学完之后

完成 12 个课程后，可进一步探索：

- **[Kode Agent CLI](https://github.com/shareAI-lab/Kode-cli)**：开源编程 Agent CLI，支持 Skill & LSP，兼容 GLM/MiniMax/DeepSeek
- **[Kode Agent SDK](https://github.com/shareAI-lab/Kode-agent-sdk)**：可嵌入后端/浏览器插件/嵌入式设备的独立 Agent 库
- **[claw0](https://github.com/shareAI-lab/claw0)**：姊妹教学仓库，讲解心跳（Heartbeat）、定时任务（Cron）、IM 多通道等"主动式常驻 AI 助手"机制

---

**核心理念：模型就是智能体。我们的工作就是给它工具，然后让开。**
