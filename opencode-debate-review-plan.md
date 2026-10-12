# OpenCode 2：双 Agent 循环辩论 Review —— 实现方案

> 目标读者：实现这个功能的编码 agent。
> 本文档基于 OpenCode 官方文档 + 相关 issue/PR 整理。部分 API 在 V1/V2 之间有差异，**第 5 节的探测必须先做**，然后再动手写代码。

---

## 0. 目标与验收标准

做一个 OpenCode 的 command（暂定 `/debate-review`），运行后：

1. 两个独立的 review agent（下称 **红方**、**蓝方**）就同一份代码改动来回讨论 **N 轮**（N 默认 3）；
2. 每一方都记得自己上一轮说过什么，**不需要把历史重贴一遍**；
3. 讨论过程与中间状态**落到磁盘**，进程中断/会话退出后可以 `--resume` 从第 k 轮接着跑；
4. 最后产出一份**合并去重、按严重度排序**的 review 建议（而不是两份独立报告并排）。

验收：跑一次真实 diff，中断，再 resume，最终输出的报告里每一条 finding 都能追溯到是"红方提出→蓝方认可"还是"蓝方反驳→红方撤回"。

---

## 1. 事实基础：OpenCode 2 的能力边界

### 1.1 Agent 的定义方式

Agent 是"system prompt + 模型偏好 + 权限 + 展示信息"的组合。Markdown 定义，正文就是 system prompt：

- 全局：`~/.config/opencode/agents/<name>.md`
- 项目：`.opencode/agents/<name>.md`（会从当前目录向上扫描到项目根）
- 子目录参与 ID：`.opencode/agents/team/reviewer.md` → agent ID 是 `team/reviewer`

frontmatter 关键字段（**V2 形态**）：

| 字段 | 说明 |
|---|---|
| `description` | 强烈建议写，模型在选择 subagent 时就看这个 |
| `mode` | `primary` / `subagent` / `all` |
| `model` | `provider/model`，可带变体 `#high` |
| `system` | 非空时替换 provider 的基础 prompt（正文写 prompt 即可，不用这个键） |
| `permissions` | 有序规则数组，每条 `{ action, resource, effect }`，`effect` ∈ `allow`/`ask`/`deny`，**最后一条匹配的规则生效**，所以宽规则放前面、例外放后面 |
| `steps` | 最大步数；到头时工具被撤掉并要求模型总结 |
| `hidden` / `color` / `disabled` | 展示与开关 |

`action` 的取值包含：`shell`、`edit`、`subagent`、`read`/`glob`/`grep`、`webfetch`/`websearch`、`skill`。

> ⚠️ **V2 明确说了要避免的 legacy 顶层字段**：`temperature`、`top_p`、`prompt`、`permission`（单数）、`tools`、`disable`、`maxSteps`。
> V1 用的是 `permission`（单数，`"read": "allow"` 这种简写）和 `prompt`。**这两个版本写法不一样，先探测再写。**

内置 agent：`build` / `plan`（primary），`general` / `explore`（subagent），外加隐藏的 `compaction` / `title` / `summary`。V2 里 `general` **不能**再起子 agent。

### 1.2 Command 的定义方式

- 路径：`~/.config/opencode/commands/<name>.md` 或 `.opencode/commands/<name>.md`（**复数是 V2 主用，单数兼容 V1**）；文件名即命令名
- 也可以在 `opencode.json` 的 `command` 键里写
- frontmatter：`description`、`agent`、`model`、`subtask`（强制以 subagent 身份运行）
- 正文是 prompt 模板，支持四种插值：
  - `$ARGUMENTS` —— 命令后面的全部内容
  - `$1` `$2` `$3` —— 按位置取参
  - `` !`bash 命令` `` —— 在项目根执行，**stdout 会注入到 prompt 里**
  - `@路径` —— 把文件内容注入 prompt

> 最后一条 `` !`...` `` 是这个方案的关键抓手：**它让我们可以用一个外部脚本干完所有活，把结果当 prompt 喂给模型**。

### 1.3 Session 与子 session —— 持久化和续接的真正基石

这是整个方案里最重要的一节，请重点看：

- **Session 本身就是持久化的**。OpenCode 把会话存在本地，`opencode session list` 能列出来，`opencode export <id>` 能导出
- **子 session 通过 `parentID` 挂在父 session 下**，`GET /session/:id/children` 可以列出来
- **续接 = 复用同一个 session ID 再发一次 prompt**。官方没有单独的 "resume" 端点/方法——所谓续接，就是拿着同一个 `:id` 继续 `POST /session/:id/message`
- **session 的历史就是记忆**。这是本方案的核心杠杆：你不需要把前几轮的对话重新塞进 prompt，因为那个 session 里本来就有

服务端 HTTP API（`opencode serve`，默认 `127.0.0.1:4096`，OpenAPI 3.1 spec 挂在 `/doc`）：

| 方法 | 路径 | 用途 |
|---|---|---|
| `POST` | `/session` | 建 session，body `{ parentID?, title? }` |
| `GET` | `/session` | 列出所有 session |
| `GET` | `/session/:id` | 取 session |
| `GET` | `/session/:id/children` | 子 session |
| `GET` | `/session/:id/message` | 列消息（`?limit=`） |
| `POST` | `/session/:id/message` | **发消息并等回复** |
| `POST` | `/session/:id/prompt_async` | 发消息不等（204） |
| `POST` | `/session/:id/abort` | 打断 |
| `GET` | `/agent` | 列出所有可用 agent |

发消息的 body 接受：`{ messageID?, model?, agent?, noReply?, system?, tools?, parts }`。

> ★ **`agent?` 是 body 字段**——选 agent 没有单独的接口，就是在发消息时指定。这正是"同一个 server 上跑多个不同人格的 agent"的实现方式。

JS/TS SDK：`npm i @opencode-ai/sdk`。`createOpencodeClient({ baseUrl })` 连已运行的实例，`createOpencode()` 连服务端一起拉起来。方法名基本是 API 路径的驼峰版：`session.create/get/list/children/messages/message/prompt/abort/revert`，`app.agents()`。默认 `responseStyle: "fields"`，返回值在 `.data` 上，且**不会自动抛错**（`throwOnError` 默认 false），要自己判。

Plugin：放 `.opencode/plugins/*.js`（或 `~/.config/opencode/plugins/`），启动时自动加载。每个插件导出函数，收到 context 对象 `{ project, directory, worktree, client, $ }`——**`client` 就是上面那个 SDK client**，`$` 是 Bun 的 shell。可用事件/hook 名：`command.executed`、`session.created`、`session.idle`、`session.updated`、`session.error`、`message.updated`、`message.part.updated`、`tool.execute.before`、`tool.execute.after`、`permission.asked`、`tui.prompt.append`、`tui.command.execute`、`tui.toast.show`、`shell.env` 等。
> 注意：官方文档**只列了事件名，没写 payload 字段**。要在 hook 里用到 session id 之类，先 `console.log(JSON.stringify(event))` 打一遍看实际结构，别猜。

CLI 侧（如果不想起 server）：

```
opencode run [message..]          # 非交互模式
  --agent <name>                  # 指定 agent
  --session / -s <id>             # ★ 续接指定 session
  --continue / -c                 # 续接最后一个 session
  --fork                          # 续接时 fork（要和上面两个一起用）
  --model / -m <provider/model>
  --format json                   # 原始事件流
  --attach <url>                  # 挂到已运行的 serve 上，省掉 MCP 冷启动
opencode session list --format json -n 20
opencode serve [--port 4096]
```

### 1.4 三条"启动 / 续接子 agent"的路径 —— 对比

| | 路径 A：自己建 session 走 HTTP/SDK | 路径 B：内置 `subagent` 工具的续接参数 | 路径 C：`opencode run --session` |
|---|---|---|---|
| 启动 | `POST /session {parentID}` | 让模型调 `subagent` 工具 | 起一个新进程 |
| 续接 | 复用同一个 session id 再 `POST /message` | 传 `sessionID` / `task_id` 参数 | `--session <id>` |
| 可控性 | ★★★ 完全可控，轮次/终止/并发自己写 | ★ 由模型决定，可能提前收尾 | ★★ 可控但要反复起进程 |
| 依赖未确认 API | 无 | **有**（见下） | 无 |
| 适用 | **推荐主路线** | 轻量场景 | 兜底 / 调试 |

**关于路径 B 的不确定性（务必先探测）：**

- V1 的工具叫 `task`，**它确实支持续接**：返回值里带 `task_id: ${session.id} (for resuming to continue this task if needed)`，把它传回 `task` 就能带着完整上下文续跑。还有个 PR 让 `task_id` 支持可读 slug（如 `explore-auth`），ID 由 `sha256(slug + 根 session id)` 派生，同一个根 session 下同名 slug 会命中同一个子 session——这对"固定两个 agent 反复讨论"简直量身定做。
- **但 V2 把 `task` 工具换成了 `subagent` 工具**，官方 V2 agents 文档明确说 subagent"在子 session 中以全新上下文运行"，**整页都没有提 subagent 工具的参数，也没有提任何续接机制**。
- 有兼容层把 V2 的 `subagent` 调用映射回 V1 的 `task` 管线：`agent` → `subagent_type`，`sessionID` → `task_id`。也就是说 **V2 的 `subagent` 工具很可能接受 `sessionID` 参数来续接**，但这是第三方库的映射，不是官方文档。
- 另外 issue #6584 是在**请求**加 `resumeSessionId` 参数、把子 session 暴露给父级；PR #34947 想给 task 工具加 `resume: true` 门禁（防止误续接闲置 session）但**被关闭了**（维护者说"这条分支已经并入 V2，不再接受 V1 的 PR"）。

**结论：如果你要实现的是"稳定可复现的 N 轮循环"，不要押注路径 B。走路径 A。**

---

## 2. 推荐架构（方案 A）：外部编排脚本 + Server API

### 2.1 为什么是它

三个理由：

1. **循环必须是确定性的。** 如果你在 markdown command 里写"请重复 5 轮"，那是**让模型自己循环**——它会提前收尾、会偷懒、会把 5 轮压成 1 轮。循环必须在代码里（`for` 循环），模型只负责每一轮的内容。
2. **持久化天然由 session 承担。** 每个 agent 一个长期存活的 session，历史自动累加，不用手写滑动窗口、不用重贴上下文。
3. **断点续接只是复用 ID。** 把两个 session id 写进 `state.json`，resume 就是读回来接着发消息。

### 2.2 目录结构

```
.opencode/
├── agents/
│   ├── review-red.md          # 红方：主动挑问题
│   ├── review-blue.md         # 蓝方：逐条反驳 + 补漏
│   └── review-judge.md        # 裁判：汇总（可选，也可以让主 agent 干）
├── commands/
│   └── debate-review.md       # 命令入口
├── scripts/
│   └── debate-review.mjs      # ★ 编排器（真正干活的地方）
└── reviews/                   # 运行时产物，建议加进 .gitignore
    └── <runId>/
        ├── state.json         # 轮次、两个 session id、历史摘要
        ├── context.md         # 共享材料（diff + 文件清单），只写一次
        ├── transcript.md      # 人类可读的完整辩论记录
        └── report.md          # 最终报告
```

### 2.3 两个 agent 的定义

`.opencode/agents/review-red.md`：

```markdown
---
description: 代码审查辩论的红方。负责主动发现缺陷并给出可验证证据，对蓝方的反驳逐条回应。
mode: all
model: anthropic/claude-opus-4-5
steps: 24
permissions:
  - action: read
    resource: "*"
    effect: allow
  - action: edit
    resource: "*"
    effect: deny
  - action: shell
    resource: "*"
    effect: deny
  - action: shell
    resource: "git diff*"
    effect: allow
  - action: shell
    resource: "git log*"
    effect: allow
---

你是代码审查的红方。你的职责是**找出真实存在的问题**，不是找平衡。

规则：
- 每条 finding 必须给出 `文件:行号` 和**具体的失败场景**（什么输入 → 什么错误结果）。给不出场景的，别写。
- 不要评论格式、命名、import 顺序这类噪音。
- 当蓝方反驳你时，只有两种合法回应：**撤回**（并说明为什么它是对的）或**坚持**（并给出它没考虑到的反例）。不许含糊地"部分同意"。
- 每轮结束输出 JSON，见 prompt 中给的 schema。
```

`.opencode/agents/review-blue.md`：

```markdown
---
description: 代码审查辩论的蓝方。逐条质疑红方的每条 finding，指出误报、严重度错判与遗漏。
mode: all
model: anthropic/claude-opus-4-5
steps: 24
permissions: [同上]
---

你是代码审查的蓝方，扮演**怀疑论者**。红方已经给了你它的发现。

对每一条：
1. 先判断是不是**误报**——代码里是不是已经有防护？（去看代码，别凭感觉）
2. 判断**严重度是否被夸大**。
3. 如果不是误报，明确写"这是真的"。

然后做红方没做的事：**补充它漏掉的问题**。

禁止为了显得有价值而制造分歧。红方对了就说它对了。
每轮结束输出 JSON，见 prompt 中给的 schema。
```

`.opencode/agents/review-judge.md`：不挑新问题，只做合并去重 + 严重度排序 + 标注"双方是否达成一致"。

**⚠️ 一个坑**：这里 `mode` 必须是 `all`（或 `primary`），**不要用 `subagent`**。因为 `mode: subagent` 的语义是"只能通过 subagent 工具在子 session 里跑"，而我们是自己 `POST /session` 建 session 再指定 agent——不是通过工具派的。用 `subagent` 可能导致 agent 不可用。写完 `opencode agent list` 确认一下三个都在。

### 2.4 命令入口

`.opencode/commands/debate-review.md`：

```markdown
---
description: 两个 agent 循环辩论式 review 代码改动
agent: build
---

下面是编排脚本的完整输出。请原样转述其中的关键结论，并优先展示最终报告。

!`node .opencode/scripts/debate-review.mjs $ARGUMENTS`
```

用法：

```
/debate-review                          # 默认 review 工作区改动
/debate-review --target "HEAD~3..HEAD"  # review 指定区间
/debate-review --rounds 5
/debate-review --resume 20261012-1430   # 续接上次
```

> ⚠️ **要探测**：`$ARGUMENTS` 是否在 `` !`...` `` 块内部也会被替换。文档说明前者是"prompt 模板的占位符替换"、后者是"执行 bash 并把输出注入"——两者叠加时哪个先发生没写。**保险做法**：脚本同时支持从环境变量读参数，命令模板里写成
> `` !`node .opencode/scripts/debate-review.mjs $ARGUMENTS 2>&1 || node .opencode/scripts/debate-review.mjs` ``
> 不太优雅。**更稳的做法是让脚本无参运行、用一个 `.opencode/reviews/current.json` 存上次参数**，或者干脆让用户直接在终端跑脚本、command 只负责读结果。先试出来再定。

### 2.5 编排器核心骨架（`debate-review.mjs`）

用原生 `fetch` 打 HTTP API，**零依赖**（比引 SDK 少一层版本不确定性；想要类型安全再换 SDK）：

```js
#!/usr/bin/env node
// .opencode/scripts/debate-review.mjs
import { execFileSync } from "node:child_process"
import fs from "node:fs"
import path from "node:path"

const BASE = process.env.OPENCODE_URL ?? "http://127.0.0.1:4096"
const AUTH = process.env.OPENCODE_SERVER_PASSWORD
  ? "Basic " + Buffer.from(`${process.env.OPENCODE_SERVER_USERNAME ?? "opencode"}:${process.env.OPENCODE_SERVER_PASSWORD}`).toString("base64")
  : undefined

async function api(method, p, body) {
  const res = await fetch(BASE + p, {
    method,
    headers: { "content-type": "application/json", ...(AUTH ? { authorization: AUTH } : {}) },
    body: body ? JSON.stringify(body) : undefined,
  })
  if (!res.ok) throw new Error(`${method} ${p} -> ${res.status} ${await res.text()}`)
  return res.status === 204 ? null : res.json()
}

// —— 参数 ——
const argv = process.argv.slice(2)
const get = (k, d) => { const i = argv.indexOf(k); return i >= 0 ? argv[i + 1] : d }
const has = (k) => argv.includes(k)

const RUN_ID  = get("--run-id", new Date().toISOString().replace(/[-:T]/g, "").slice(0, 12))
const ROUNDS  = Number(get("--rounds", 3))
const TARGET  = get("--target", null)          // 如 "HEAD~3..HEAD"，null = 工作区改动
const RESUME  = has("--resume") ? get("--resume", RUN_ID) : null
const MODEL   = { providerID: "anthropic", modelID: "claude-opus-4-5" }

const DIR = path.join(process.cwd(), ".opencode", "reviews", RUN_ID)
const STATE = path.join(DIR, "state.json")

// —— 1. 准备共享材料（只做一次）——
function buildContext() {
  const p = path.join(DIR, "context.md")
  if (fs.existsSync(p)) return p
  fs.mkdirSync(DIR, { recursive: true })
  const git = (...a) => { try { return execFileSync("git", a, { encoding: "utf8" }) } catch { return "" } }
  const diff = TARGET ? git("diff", TARGET) : git("diff", "HEAD") + git("diff", "--staged")
  const files = git("diff", "--name-only", TARGET ?? "HEAD").trim().split("\n").filter(Boolean)

  fs.writeFileSync(p, [
    `# Review 目标\n\n${TARGET ?? "工作区未提交改动"}\n`,
    `# 涉及文件\n\n${files.map(f => `- ${f}`).join("\n")}\n`,
    `# Diff\n\n\`\`\`diff\n${diff}\n\`\`\`\n`,
    `# 近期提交\n\n\`\`\`\n${git("log", "--oneline", "-15")}\n\`\`\`\n`,
    `\n> 你可以用 read/grep 工具去看仓库里的完整文件来验证任何一条猜测。\n`,
  ].join("\n"))
  return p
}

// —— 2. 建 / 读两个持久 session ——
async function ensureState() {
  if (RESUME && fs.existsSync(STATE)) return JSON.parse(fs.readFileSync(STATE, "utf8"))

  // 一个"根 session"把这次 review 的所有子 session 串起来，方便在 TUI 里导航
  const root = await api("POST", "/session", { title: `debate-review ${RUN_ID}` })
  const a    = await api("POST", "/session", { parentID: root.id, title: `[${RUN_ID}] 红方` })
  const b    = await api("POST", "/session", { parentID: root.id, title: `[${RUN_ID}] 蓝方` })

  const st = {
    runId: RUN_ID, round: 0, rootSession: root.id,
    sessionA: a.id, sessionB: b.id,
    rounds: ROUNDS, target: TARGET,
    history: [],          // [{ round, speaker, converged, disputed: n }]
  }
  fs.mkdirSync(DIR, { recursive: true })
  fs.writeFileSync(STATE, JSON.stringify(st, null, 2))
  return st
}

// —— 3. 发一轮话 ——
// ★ 续接的全部秘密就在这里：path 里的 id 是老的，消息就追加到同一个 session 上
async function ask(sessionID, agent, text) {
  const res = await api("POST", `/session/${sessionID}/message`, {
    agent,
    model: MODEL,
    parts: [{ type: "text", text }],
  })
  // 返回值是 { info, parts:[...] }，把 text 类型的 part 拼起来
  return (res.parts ?? [])
    .filter(p => p.type === "text")
    .map(p => p.text)
    .join("\n")
}

// —— 4. 主循环 ——
const ctxPath = buildContext()
const st = await ensureState()
const REF = `共享材料：${ctxPath}（含 diff、文件清单、近期提交）。需要验证任何猜测时，直接去读仓库里的真实文件，不要只凭 diff 推断。`

let lastBlue = ""
for (let r = st.round + 1; r <= st.rounds; r++) {
  const redPrompt = r === 1
    ? `${REF}\n\n第 1 轮。请独立完成审查，按 schema 输出 JSON。`
    : `${REF}\n\n第 ${r} 轮。蓝方上一轮对你提出了以下意见，请**逐条**回应（撤回 / 坚持），然后给出修订后的完整清单。\n\n<蓝方>\n${lastBlue}\n</蓝方>`

  const red = await ask(st.sessionA, "review-red", redPrompt)

  const bluePrompt = `${REF}\n\n第 ${r} 轮。红方本轮的结论如下。请逐条判断（误报 / 真问题 + 严重度），并补充它漏掉的。最后按 schema 输出 JSON。\n\n<红方>\n${red}\n</红方>`
  const blue = await ask(st.sessionB, "review-blue", bluePrompt)

  lastBlue = blue
  st.round = r
  st.history.push({ round: r, at: new Date().toISOString(), redChars: red.length, blueChars: blue.length })
  fs.writeFileSync(STATE, JSON.stringify(st, null, 2))
  fs.appendFileSync(path.join(DIR, "transcript.md"),
    `\n\n## 第 ${r} 轮 · 红方\n\n${red}\n\n## 第 ${r} 轮 · 蓝方\n\n${blue}\n`)

  console.error(`[round ${r}/${st.rounds}] red=${red.length}c blue=${blue.length}c`)
  if (isConverged(blue)) { console.error(`[round ${r}] 已收敛，提前结束`); break }
}

// —— 5. 裁判汇总 ——
const judge = await api("POST", "/session", { parentID: st.rootSession, title: `[${RUN_ID}] 裁判` })
const report = await ask(judge.id, "review-judge",
  `${REF}\n\n完整辩论记录见 ${path.join(DIR, "transcript.md")}。请读它，然后输出最终报告。`)
fs.writeFileSync(path.join(DIR, "report.md"), report)
console.log(report)
```

需要你自己补两个函数：

- `isConverged(blueText)` —— 判断是否该提前结束。两个做法：
  1. 便宜的：正则数蓝方回复里"这是真的"和"误报"的条数，若本轮没有新增争议项就返回 `true`；
  2. 可靠的：给 `POST /session/:id/message` 的 body 加 `format: { type: "json_schema", schema: {...} }` 走**结构化输出**，模型会通过一个 StructuredOutput 工具返回校验过的 JSON，从 `info.structured_output` 里读。SDK 里还可以用 `retryCount` 控制校验重试（默认 2），失败会返回 `StructuredOutputError`。**推荐这个**——顺便还解决了"怎么稳定解析每轮结果"的问题。
- 参数解析 / 错误处理 / `--help`。

### 2.6 会话启动方式

脚本假设 server 已经在跑。三种接法：

1. **用户先开**：另开一个终端 `opencode serve`，再跑脚本（最简单，调试友好）；
2. **脚本自拉**：`spawn("opencode", ["serve", "--port", "4096"])`，等 `/doc` 能不通再继续，退出时杀掉——**注意别和用户已经开着的 TUI 抢端口**，先探测 4096 是否已被占用，占用了就直接复用；
3. **SDK 的 `createOpencode()`**：一行把 server + client 都拉起来，但会多一个 npm 依赖。

> ⚠️ 如果设了 `OPENCODE_SERVER_PASSWORD`，所有请求都要带 HTTP Basic（用户名默认 `opencode`）。上面骨架里处理了。

### 2.7 为什么这套设计是对的（三个容易写错的地方）

1. **每轮只传对方的增量，不传全史。**
   红方的 session 里已经有它自己前几轮说过的话了。所以第 r 轮给红方的 prompt 里只需要放**蓝方上一轮的输出**。如果你把整个 transcript 每轮重贴一遍，token 会按 O(N²) 涨，context 还容易爆。
2. **共享材料落盘、只写一次。**
   diff 可能很大。写进 `context.md`，prompt 里只给路径，让 agent 需要时自己 `read`。这样每轮的 prompt 都很小。
3. **循环在代码里，不在 prompt 里。**
   这是整份文档里最重要的一条。任何"让模型自己数着轮次循环"的写法都会失败。

### 2.8 断点续接

`state.json` 里存了 `round` / `sessionA` / `sessionB`，所以 resume 就是：

```js
if (RESUME) { st = JSON.parse(read(STATE)); /* 从 st.round + 1 继续 */ }
```

两个 session 还在磁盘上活着，直接继续发消息即可，上下文一点不丢。这就是"持久化 + 续接"的完整闭环——**没有用到任何未公开或未确认的 API**。

---

## 3. 备选方案

### 方案 B：纯 Markdown，用内置 `subagent` 工具的续接参数

最轻，没有脚本、没有 server。命令正文大致是：

```markdown
---
description: 双 agent 辩论 review
---
第 1 轮：调用 subagent 工具，agent=review-red，得到 sessionID 记为 A。
       再调用 subagent，agent=review-blue，把红方结论给它，得到 sessionID 记为 B。
第 2 轮：再用 subagent 工具，**带上 sessionID=A 续接**，把蓝方意见给它……
（重复 N 轮）
最后汇总。
```

**只有当第 5 节的探测确认你的版本支持 `sessionID` / `task_id` 续接时才用这条路。**
即使支持，它还有个结构性缺陷：**循环由模型驱动**，它随时可能少跑一轮、或者忘了带 `sessionID` 导致每轮从零开始（issue #13910 就是这类问题的记录：subagent 失败时连 session id 都不返回，父 agent 完全无法恢复）。所以它只适合"轮数少、可以容忍不稳定"的场景。

### 方案 C：Plugin hook 驱动

`.opencode/plugins/debate-review.js`，context 里直接有 `client`（就是那个 SDK client），可以：

- 监听 `command.executed`，识别 `/debate-review` 后自己触发编排；
- 或者通过 `tui.prompt.append` / `tui.toast.show` 把进度反馈到界面上；
- 或者用 `tool.execute.after` 观测每次子 agent 调用的返回。

**注意**：官方文档没写这些事件的 payload 字段，也没给"从插件里创建 session"的示例。要上手先 `console.log(JSON.stringify(event))` 摸清结构。
这条路比方案 A 更"原生"（能接进 TUI 交互），但**不确定性更高**，建议做成方案 A 之后的增强，而不是起点。

---

## 4. 提给 agent 的每一轮 prompt 骨架

红方第 1 轮：

```
共享材料：{context.md 路径}（含 diff、文件清单、近期提交）。
需要验证任何猜测时，直接去读仓库里的真实文件，不要只凭 diff 推断。

第 1 轮。请独立完成审查，按 schema 输出 JSON：
{
  "findings": [
    {
      "id": "R1-1",
      "file": "src/foo.ts",
      "line": 42,
      "severity": "high|medium|low",
      "claim": "一句话说清问题",
      "failure_scenario": "什么输入 → 什么错误结果",
      "evidence": "你实际读了哪段代码",
      "confidence": 0.0-1.0
    }
  ]
}
只写你能给出 failure_scenario 的条目。宁少勿滥。
```

红方第 r 轮（自己 session 里已有前几轮）：

```
第 {r} 轮。蓝方上一轮的意见如下。请**逐条**回应，每条只能选"撤回"或"坚持"：
撤回 → 说明它哪一点说服了你；
坚持 → 给出它没考虑到的具体反例。

<蓝方>
{blue 上一轮输出}
</蓝方>

然后输出修订后的完整 findings JSON。
```

蓝方第 r 轮：

```
第 {r} 轮。红方本轮的结论如下。逐条判断：
- 误报？（去读代码确认，已有防护就说明防护在哪一行）
- 严重度是否被夸大？
- 真问题 → 明确写"这是真的"

然后补充红方漏掉的问题。**不要为了显得有价值而制造分歧**，红方对了就说它对了。

<红方>
{red 本轮输出}
</红方>

输出 JSON。
```

---

## 5. ⚠️ 动手前必须先做的探测

V1/V2 的 API 有差异，网上文档互相打架。**别猜，跑这五条**：

```bash
# 1. 版本与 agent 目录约定（V2 用复数 permissions / 复数目录）
opencode --version
opencode agent list

# 2. 拿本机真实的 OpenAPI spec —— 这是最权威的来源
opencode serve --port 4096 &
curl -s http://127.0.0.1:4096/doc | jq -r '.paths | keys[]'      # 端点全集
curl -s http://127.0.0.1:4096/doc | jq '.paths["/session/{id}/message"]'   # 发消息的 body schema
curl -s http://127.0.0.1:4096/agent | jq                              # agent 列表

# 3. CLI 是否有续接 flag
opencode run --help | grep -E 'session|continue|agent|attach'

# 4. 写一个最小 agent 文件，重启后确认 agent list 里出现了，且没报 frontmatter 字段无效
#    （重点验证 permissions 还是 permission、prompt 还是 system）

# 5. 路径 B 是否可用：在 TUI 里手动触发一次 subagent 调用，看返回值里
#    有没有 session id 字段、以及工具描述里有没有 sessionID / task_id 参数
```

**探测结果决定：**

| 探测发现 | 应对 |
|---|---|
| `/session/{id}/message` 接受 `agent` 字段 | 方案 A 直接可用 ✅ |
| 不接受 `agent` | 退路：`POST /session/:id/command`，body `{ agent, model, command, arguments }` 走 slash command 机制；或建 session 前用 `session.update` 绑定 agent |
| `permissions`（复数）报错 | 换 V1 写法：`permission:` + `"read": "allow"` 简写；`prompt:` 指定 system prompt 文件 |
| `subagent` 工具描述里有 `sessionID` / `task_id` | 方案 B 可行，可作为轻量版一起做 |
| `/session/{id}/children` 不存在 | 不影响主流程，只是 TUI 里导航不到子会话 |

---

## 6. 已知的坑

1. **`mode: subagent` 的 agent 可能无法在自建 session 里用** —— 用 `all`。（见 2.3）
2. **`$ARGUMENTS` 在 `` !`...` `` 内部的行为未验证** —— 见 2.4，先实测。
3. **SDK 默认不抛异常** —— `throwOnError` 默认 `false`，`client.session.get()` 失败不会 throw，别以为 try/catch 能兜住。
4. **V2 的 `cost` 字段目前硬编码为 0** —— token 数是对的，成本统计别信。
5. **`general` agent 不能起子 agent**，`subagent` 权限被 deny 也是同理。
6. **`experimental.policies`、`subagent_depth` 这些是实验性配置**，行为不稳定，别依赖。
7. **环境变量改了要重启**，配置文件改了（V2 支持 live reload）不用。
8. **V2 读 `AGENTS.md`，不读 `CLAUDE.md`** —— 项目约定放对文件名。
9. **权限规则是"最后一条匹配的生效"** —— 所以 `"*": deny` 要写在具体 allow 规则的**前面**，跟直觉相反，很容易写反导致 agent 什么工具都用不了。
10. **轮数上限要设死**——两个高 `steps` 的 agent 互相刷起来，token 消耗是乘法关系。建议默认 3 轮，并在脚本里硬性 `rounds <= 6`。
11. **不要用同一个 agent 起两个 session 来"扮演两方"** —— 两个 session 会共享同一份 system prompt，人格差异靠 prompt 硬凹，效果远不如两个真正不同的 agent 配置。这也是本方案坚持建两个 agent 文件的原因。

---

## 7. 交付物清单

做完之后应该长这样：

- [ ] `.opencode/agents/review-red.md` / `review-blue.md` / `review-judge.md`
- [ ] `.opencode/commands/debate-review.md`
- [ ] `.opencode/scripts/debate-review.mjs`（可执行，带 `--help`）
- [ ] `README` 或注释里写清：怎么起 server、怎么 resume、参数含义
- [ ] `.gitignore` 加 `.opencode/reviews/`
- [ ] 一次真实演练：跑 → 中断 → `--resume` → 拿到最终报告

---

## 参考来源

- [Agents — OpenCode](https://opencode.ai/docs/agents/)
- [Agents — OpenCode V2](https://dev.opencode.ai/v2/docs/agents/)
- [Commands — OpenCode](https://opencode.ai/docs/commands/)
- [SDK — OpenCode](https://opencode.ai/docs/sdk/)
- [Server — OpenCode](https://opencode.ai/docs/server/)
- [Plugins — OpenCode](https://opencode.ai/docs/plugins/)
- [Config — OpenCode](https://opencode.ai/docs/config/)
- [CLI — OpenCode](https://opencode.ai/docs/cli/)
- [Issue #6584：Agent-level resume for subagents (resumeSessionId)](https://github.com/anomalyco/opencode/issues/6584)
- [Issue #13910：Task tool loses session ID when sub-agent fails](https://github.com/anomalyco/opencode/issues/13910)
- [PR #32122：allow human-readable slugs as task_id](https://github.com/anomalyco/opencode/pull/32122)
- [PR #34947：add dispatch controls to the task tool（已关闭）](https://github.com/anomalyco/opencode/pull/34947)
- [oh-my-opencode-slim — v2 Delegation & Interview Bridge](https://deepwiki.com/alvinunreal/oh-my-opencode-slim/11.2-v2-delegation-and-interview-bridge)
- [opencode-v2-delegate-tool](https://socket.dev/npm/package/opencode-v2-delegate-tool)

---

## 附：一句话总结

> **OpenCode 的"续接"不是一个特殊功能，就是"复用同一个 session id 再发一次消息"。**
> 所以不要去找什么 resume API —— `POST /session` 建两（三）个持久 session，把它们的 id 存进 `state.json`，然后用一个普通脚本在 `for` 循环里轮流 `POST /session/:id/message`，每次 `body.agent` 指到不同的 agent 上。轮次、终止、断点续接、报告汇总全部由脚本控制，模型只负责每一轮的内容。
> 这条路不依赖任何未确认的 API，是唯一能保证稳定跑满 N 轮的方案。
