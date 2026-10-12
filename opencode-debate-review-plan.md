# OpenCode V2：双 Agent 循环辩论 Review —— 实现方案

> **本文档只写 V2。** V1 的写法一律不采用，只在第 1 节做差异对照，避免再混。
> 部分 API 在 V2 内部仍属 beta，**第 6 节的探测必须先做**再动手。
> 文档来源均为 `opencode.ai/v2/docs/*`，文末列出。

---

## 0. 目标与验收

做一个 `/debate-review` command，运行后：

1. 两个独立 agent（**红方**挑问题、**蓝方**逐条质疑并补漏）就同一份代码改动来回讨论 **N 轮**（默认 3，硬上限 6）；
2. 每一方记得自己前几轮说过什么，**不需要重贴历史**；
3. 过程与状态**落盘**，中断后可 `--resume` 从第 k 轮续跑；
4. 最终产出一份**合并去重、按严重度排序**的报告，每条 finding 标注是"红方提出→蓝方认可"还是"蓝方反驳→红方撤回"。

---

## 1. 先说清楚 V1 / V2 的差异（上一版混用的地方）

V2 不是字段改名，是**三处刻意的破坏性变更**，官方 migrate 页原话：

> 三处有意的破坏：新的 plugin API；新的 server API / client 契约；终端客户端配置从分层的 `tui.json(c)` 合并成单一全局 `cli.json`。

其余 V1 支持的东西官方声明应当继续工作（不工作算兼容 bug）。以下是对照表，**本方案只用右列**：

### 1.1 配置与目录

| V1 | V2 |
|---|---|
| 顶层 `agent` | **`agents`** |
| 顶层 `command` | **`commands`** |
| 顶层 `plugin` | **`plugins`** |
| `subtask`（command 字段） | **`subagent`**（`subtask` 仍接受，但已废弃） |
| `snapshot` / `attachment` / `reference` | **`snapshots` / `media` / `references`** |
| `autoshare: true\|false` | **`share: "manual"\|"auto"\|"disabled"`** |
| `autoupdate` | **`update`**（`false`→`"disable"`，`"notify"`→`"notify"`，`true`→`"auto"`） |
| 顶层 `subagent_depth` | **`experimental.subagent_depth`**（顶层会告警并被忽略） |
| `skills.paths` + `skills.urls` | **一个有序的 `skills` 数组** |
| `small_model` | **`agents.title.model`** |
| `enabled_providers` / `disabled_providers` | **policies**（deny-by-default + allow） |

配置文件路径不变：`~/.config/opencode/opencode.json(c)`、`<project>/opencode.json(c)`、`<project>/.opencode/opencode.json(c)`。

> ⚠️ 顶层 V1/V2 字段可以共存（原生 V2 值胜出），但 `agents` / `providers` / `commands` / `models` **必须整体用一种格式**，不能嵌套混写。只有 `mcp`、`compaction`、`experimental` 内部允许混。

### 1.2 Agent frontmatter

| V1 | V2 |
|---|---|
| `permission`（单数，`"read": "allow"` 简写） | **`permissions`**：有序数组 `{ action, resource, effect }` |
| `tools` | 并入 `permissions` |
| `prompt: <文件路径>` | **`system`**（正文写就是 system prompt） |
| `disable: true` | **`disabled: true`** |
| `maxSteps` | **`steps`** |
| `model` + `variant` 两个字段 | **`model: provider/model#variant`** 合成一个 |
| `temperature` / `top_p` / provider `options` | **挪到 `request.body`** 下 |
| mode 权限 action：`bash` / `task` / `write` / `patch` | **`shell` / `subagent` / `edit` / `edit`** |

（`temperature` / `top_p` / `prompt` / `permission` / `tools` / `disable` / `maxSteps` 在 V2 属于要避免的 legacy 字段。迁移是自动的，但新写就用 V2 形态。）

### 1.3 Command

| V1 | V2 |
|---|---|
| 目录 `command/` | **`commands/`**（单数仍被发现，官方建议用复数） |
| `@路径` 会自动注入文件内容 | **不展开**——在存储的模板里 `@path` 就是普通文本。要附文件得在调用命令时从 composer 加附件 |
| `` !`cmd` `` | 保留。**参数占位符先展开**，所以 `` !`git log --oneline -$1` `` + `/history 5` → `git log --oneline -5`（这条我上一版说"未验证"，现在确认了） |
| 嵌套路径 | `.opencode/commands/team/review.md` → `/team/review` |
| — | 新增 `subagent: true`：强制在后台子会话里跑 |

`$ARGUMENTS`、`$1..$n` 照旧；最高位占位符会吃掉它和它后面的所有词；缺位变空串；没有任何占位符时，非空参数会被追加到末尾。

### 1.4 服务端 / 客户端 / 插件（最要命的一处）

| V1 | V2 |
|---|---|
| `opencode serve` + `POST /session`、`POST /session/:id/message` | **没有面向用户的 serve 命令**。V2 是共享后台 service（`opencode service ...`）。HTTP API 在 **`/api/...`** |
| SDK 参数嵌套 `{ path: {...}, body: {...} }` | **扁平参数**：`session.prompt({ sessionID, text })` |
| 发消息 body 带 **`agent`** 字段 | **prompt body 里没有 agent**。agent 在**建会话时**给，或之后用 `session.switchAgent` 切 |
| `POST /session/:id/message` 发送后等回复返回 | `prompt` 是**投递语义**："durably admit one session input and schedule agent-loop execution"。**它不等回复**，要另外 wait |
| `@opencode-ai/sdk` | **`@opencode/client`**（连服务端）/ **`@opencode/sdk`**（嵌进自己应用）/ **`@opencode-ai/plugin/v2`**（插件） |
| 插件 = 导出工厂函数，收 `{ project, client, $, directory, worktree }` 返回 hooks 对象 | 插件 = **`Plugin.define({ id, setup })`**，hooks 挂在 **`ctx.<domain>.hook(...)`** 上，注册返回 `{ dispose() }` |
| 事件名如 `session.idle`、`message.updated`、`tool.execute.after`，通过 `event` hook 收 | 经 **`ctx.event`** 订阅事件流；事件名变了（如 `session.created`、`session.usage.updated`、`session.step.ended`） |
| 内置工具 `task`（参数 `subagent_type` / `task_id`） | **`subagent`**（参数 **`agent` / `sessionID`**）。v1↔v2 映射：`subagent_type`↔`agent`，`task_id`↔`sessionID` |
| 有 `lsp` 配置和 LSP 工具 | **V2 不跑 language server，没有 LSP 工具，不产 diagnostics**——改用 lint / typecheck / 编译器命令 |
| `instructions` 生效 | **接受但不加载**（会告警）——写 `AGENTS.md` |
| `CLAUDE.md` fallback | **没了**，只认 `AGENTS.md` |
| 内置 `scout` subagent | **V2 没有** |

> 内置 agent（V2）：`build` / `plan`（primary）、`general` / `explore`（subagent）、隐藏的 `compaction` / `title` / `summary`。**`general` 不能再起 subagent。**

### 1.5 一句话：为什么上一版不能直接用

上一版整套"HTTP API 编排"是拿 V1 的 `opencode serve` + `POST /session/:id/message` + body 带 `agent` 写的。V2 里这三件事**全都不成立**：没有那个 serve、路径变成 `/api/session/{id}/prompt`、而且 prompt 根本不吃 `agent`。下面是 V2 的正确写法。

---

## 2. V2 的 Session / 续接模型（核心）

### 2.1 续接 = 复用同一个 sessionID

V2 依然**没有**独立的 resume 端点。续接就是拿着同一个 `sessionID` 再发一次 prompt。session 历史本身就是记忆。

**V2 比 V1 好在一点**：`POST /api/session` 的 body **直接带 `agent`**，所以"建一个绑定到红方 agent 的持久会话"是一步到位的。

### 2.2 V2 的 Session API（`/api/...`，OpenAPI 3.1）

机器可读 spec：**`/v2/openapi.json`**；HTML 版 `/doc`。约 142 个 operation，每个有 Operation ID（如 `session.prompt`）。大部分资源接受 `location` 查询参数（`{ directory }`）。

| 方法 + 路径 | Operation ID | 说明 |
|---|---|---|
| `POST /api/session` | `session.create` | body: `id`, `parentID`, `title`, **`agent`**, `model`, `location`, `metadata`, `permissions`（均要求出现，值可为 null）。**`parentID` 会在父会话的 location 建一个链接的子会话** |
| `POST /api/session/{sessionID}/prompt` | `session.prompt` | body: **`text`（必填）**, `id`, `files`, `agents`, `skills`, `metadata`, `delivery`, `resume`。⚠️ **投递语义，不等回复** |
| `POST /api/session/{sessionID}/agent` | `session.switchAgent` | body: `{ agent }` → **204**。改后续轮次用的 agent |
| `GET /api/session/{sessionID}/context` | `session.context` | 返回**最后一次 compaction 之后**的消息 `Session.Message.Info[]` ← 取对方输出用这个 |
| `GET /api/session/{sessionID}/message` | `session.message.list` | 分页：`limit` / `order` / `cursor` / `type` |
| `GET /api/session?parentID=` | — | 按父会话过滤；`parentID=null` 只返回根会话。**V2 没有独立的 children 端点** |
| `POST /api/session/{sessionID}/fork` | `session.fork` | body `{ before }`，拷贝历史建子会话 |
| 同资源还有 | | `update` / `delete` / `switchModel` / `move` / `command` / `synthetic` / `shell` / `compact` / **`wait`** / `revert` / `diff` / `interrupt` / `generate` / `inbox` 等 |

### 2.3 ⚠️ 两个 V2 特有的坑，务必先处理

**(a) `prompt` 不等回复。**
V1 的 `POST /session/:id/message` 是"发送并等待"。V2 的 `session.prompt` 官方描述是 *"Durably admit one session input and schedule agent-loop execution unless `resume` is false"*——**它只投递、然后调度 agent loop**，返回的是 `Session.Inbox.User`。

所以循环里必须显式等这一轮跑完。V2 session 域里有 **`wait`** 方法（插件 `ctx.session` 的方法列表里也有 `wait`），但**它的确切签名和语义文档没写**。两个候选：

- `await ctx.session.wait({ sessionID })` —— 等会话空闲；
- 或者用 **`ctx.session.generate({ sessionID, prompt })`** —— 插件文档里给了 `await ctx.session.generate({ sessionID, prompt: "Summarize this project" })` 这种写法，看着像同步返回结果，**如果它真是同步的，那它就是循环最需要的原语**。

**这两个必须先探测清楚再写循环**，否则整个循环会在 agent 还没回话的时候就去读 context，拿到的永远是上一轮的旧内容——而且不会报错，静默错。

**(b) 插件 API 还是 beta。**
V2 插件 API 明说是 beta：入口点、hooks、draft 形状、配置在稳定前都可能变；第三方文档也提到"2.x 插件 API 面在 dev build 之间仍在变动，相邻 build 的 `ctx` 形状可能不同"。**不要把方案里每一处 `ctx.*` 调用当成稳定契约**，探测为准。

---

## 3. 推荐架构：外部脚本 + `@opencode/client`

### 3.1 为什么选它（而不是插件）

两条确定性路线：插件（`Plugin.define` + `ctx.session`）和外部脚本（`@opencode/client`）。选后者，理由：

1. **命令挂载方式已确认可用**。V2 commands 的 `` !`...` `` 块会**先展开 `$1` 等占位符**再执行，且在当前项目位置用配置的 shell 跑。这把"命令"和"脚本"稳稳接起来了。
2. **不押注 beta 的插件 API**。`@opencode/client` 是发布出来的包 + 有 OpenAPI spec 兜底；插件 `ctx` 形状还在动。
3. **调试成本低一个量级**。脚本崩了能打栈、能加日志、能单独重跑；插件崩了要去翻 server 日志。
4. **它本身就是"持久化 + 续接"的最小实现**——状态落盘、id 复用，全在脚本里看得见。

插件路线留作第 5 节的升级项。

### 3.2 目录结构

```
.opencode/
├── agents/
│   ├── review-red.md          # 红方
│   ├── review-blue.md         # 蓝方
│   └── review-judge.md        # 裁判
├── commands/
│   └── debate-review.md       # 命令入口 → 调脚本
├── scripts/
│   └── debate-review.mjs      # ★ 编排器
└── reviews/                   # 运行时产物，加 .gitignore
    └── <runId>/
        ├── state.json         # 轮次 + 两个 sessionID + 历史
        ├── context.md         # 共享材料（diff），只写一次
        ├── transcript.md      # 完整辩论记录
        └── report.md          # 最终报告
```

### 3.3 三个 agent（V2 frontmatter）

`.opencode/agents/review-red.md`：

```markdown
---
description: 代码审查辩论的红方。主动发现缺陷并给出可验证证据，对蓝方的反驳逐条回应。
mode: all
model: anthropic/claude-opus-4-5
steps: 24
permissions:
  - action: read
    resource: "*"
    effect: allow
  - action: shell
    resource: "*"
    effect: deny
  - action: shell
    resource: "git diff*"
    effect: allow
  - action: shell
    resource: "git log*"
    effect: allow
  - action: edit
    resource: "*"
    effect: deny
---

你是代码审查的红方。职责是**找出真实存在的问题**，不是找平衡。

规则：
- 每条 finding 必须给出 `文件:行号` 和**具体失败场景**（什么输入 → 什么错误结果）。给不出场景的，别写。
- 不评论格式、命名、import 顺序这类噪音。
- 蓝方反驳你时只有两种合法回应：**撤回**（说明它哪点说服了你）或**坚持**（给出它没考虑到的反例）。不许含糊地"部分同意"。
- 每轮按 prompt 里的 schema 输出 JSON。
```

`.opencode/agents/review-blue.md` 同理，`description` / 正文改成怀疑论者：

```markdown
---
description: 代码审查辩论的蓝方。逐条质疑红方的每条 finding，指出误报、严重度错判与遗漏。
mode: all
model: anthropic/claude-opus-4-5
steps: 24
permissions: [同上]
---

你是蓝方，扮演**怀疑论者**。红方已经给了你它的发现。

对每一条：
1. 先判断是不是**误报**——代码里是不是已经有防护？**去看代码，别凭感觉**。
2. 判断**严重度是否被夸大**。
3. 不是误报就明确写"这是真的"。

然后做红方没做的事：**补充它漏掉的问题**。
禁止为了显得有价值而制造分歧。红方对了就说它对了。
```

`.opencode/agents/review-judge.md`：不挑新问题，只合并去重 + 严重度排序 + 标注双方是否达成一致。

**⚠️ 两个必须注意的点：**

- **`mode` 用 `all`，别用 `subagent`。** 语义上 `subagent` = "只通过 `subagent` 工具在子会话里跑"。我们是自己 `POST /api/session` 建会话再指定 agent，不是通过工具派发。用 `subagent` 可能直接不可用。
- **权限 action 是 V2 的名字**：`shell` / `edit` / `read` / `glob` / `grep` / `subagent` / `webfetch` / `websearch` / `skill`。别写 V1 的 `bash` / `task` / `write` / `patch`。
- **规则是"最后一条匹配的生效"**，所以 `"*": deny` 要写在具体 allow **前面**。跟直觉相反，写反了 agent 什么工具都用不了。

写完 `opencode agent list` 确认三个都在。

### 3.4 命令入口

`.opencode/commands/debate-review.md`：

```markdown
---
description: 两个 agent 循环辩论式 review 代码改动
agent: build
---

下面是编排脚本的完整输出。请原样呈现最终报告，不要改写其中的 finding。

!`node .opencode/scripts/debate-review.mjs $ARGUMENTS`
```

用法：

```
/debate-review
/debate-review --target "HEAD~3..HEAD"
/debate-review --rounds 5
/debate-review --resume 20261012-1430
```

> V2 已确认占位符在 shell 块里先展开，所以 `$ARGUMENTS` 能透传进去。
> 另外官方警告：shell 块在命令求值阶段执行，**不走 agent 工具权限流程**，且"不要把不可信的参数放进 shell 块"。这里参数来自用户自己敲的命令，风险可控，但别把外部输入接进来。

### 3.5 编排器骨架（`debate-review.mjs`）

```js
#!/usr/bin/env node
// .opencode/scripts/debate-review.mjs
import { OpenCode } from "@opencode/client"
import { Service } from "@opencode/client/service"
import { execFileSync } from "node:child_process"
import fs from "node:fs"
import path from "node:path"

// —— 连服务端：不自己拉起进程，能发现就复用，发现不了再 ensure ——
const endpoint = await Service.discover() ?? await Service.ensure()
const client = OpenCode.make({
  baseUrl: endpoint.url,
  headers: Service.headers(endpoint),
})

const DIR = path.join(process.cwd(), ".opencode", "reviews")
const LOCATION = { directory: process.cwd() }

const argv = process.argv.slice(2)
const get = (k, d) => { const i = argv.indexOf(k); return i >= 0 ? argv[i + 1] : d }
const has = (k) => argv.includes(k)

const RUN_ID = get("--run-id", new Date().toISOString().replace(/[-:T]/g, "").slice(0, 12))
const ROUNDS = Math.min(Number(get("--rounds", 3)), 6)          // ★ 硬上限
const TARGET = get("--target", null)
const RESUME = has("--resume")

const runDir  = path.join(DIR, RUN_ID)
const stateAt = path.join(runDir, "state.json")

// —— 1. 共享材料：只写一次，prompt 里只给路径 ——
function buildContext() {
  const p = path.join(runDir, "context.md")
  if (fs.existsSync(p)) return p
  fs.mkdirSync(runDir, { recursive: true })
  const git = (...a) => { try { return execFileSync("git", a, { encoding: "utf8" }) } catch { return "" } }
  const diff  = TARGET ? git("diff", TARGET) : git("diff", "HEAD") + git("diff", "--staged")
  const files = git("diff", "--name-only", TARGET ?? "HEAD").trim().split("\n").filter(Boolean)
  fs.writeFileSync(p, [
    `# Review 目标\n\n${TARGET ?? "工作区未提交改动"}\n`,
    `# 涉及文件\n\n${files.map(f => `- ${f}`).join("\n")}\n`,
    `# Diff\n\n\`\`\`diff\n${diff}\n\`\`\`\n`,
    `# 近期提交\n\n\`\`\`\n${git("log", "--oneline", "-15")}\n\`\`\`\n`,
    `\n> 你可以用 read / grep 工具去读仓库里的完整文件来验证任何一条猜测。\n`,
  ].join("\n"))
  return p
}

// —— 2. 建 / 读两个持久 session ——
async function ensureState() {
  if (RESUME && fs.existsSync(stateAt)) return JSON.parse(fs.readFileSync(stateAt, "utf8"))

  // ★ V2 关键：建会话时就把 agent 绑好（prompt 里没有 agent 字段）
  const root = await client.session.create({ title: `debate-review ${RUN_ID}`, location: LOCATION })
  const red  = await client.session.create({
    title: `[${RUN_ID}] 红方`, agent: "review-red",  parentID: root.id, location: LOCATION,
  })
  const blue = await client.session.create({
    title: `[${RUN_ID}] 蓝方`, agent: "review-blue", parentID: root.id, location: LOCATION,
  })

  const st = {
    runId: RUN_ID, round: 0, rounds: ROUNDS, target: TARGET,
    rootSessionID: root.id, sessionRed: red.id, sessionBlue: blue.id,
    history: [],
  }
  fs.mkdirSync(runDir, { recursive: true })
  fs.writeFileSync(stateAt, JSON.stringify(st, null, 2))
  return st
}

// ★★ 这一对函数是整份方案的核心，必须按第 6 节的探测结果落地 ★★
async function send(sessionID, text) {
  // 投递一轮输入。注意 V2 的 prompt 是"投递 + 调度"，不等回复。
  await client.session.prompt({ sessionID, text })
}
async function readReply(sessionID) {
  // 等到这一轮真的跑完，再取消息。
  // ⚠️ wait 的确切签名未在文档中给出 —— 探测后再定。候选：
  //   await client.session.wait({ sessionID })
  // 若 wait 不可用，退路是轮询 session 状态。
  await client.session.wait?.({ sessionID })
  const res = await client.session.context({ sessionID })   // 最后一次 compaction 之后的消息
  return (res?.data ?? res ?? [])
    .filter(m => m.role === "assistant" || m.type === "assistant")
    .at(-1)?.content ?? ""
}

// —— 3. 主循环 ——
const ctxPath = buildContext()
const st = await ensureState()
const REF = `共享材料：${ctxPath}（含 diff、文件清单、近期提交）。要验证任何猜测，直接读仓库里的真实文件，不要只凭 diff 推断。`

let lastBlue = ""
for (let r = st.round + 1; r <= ROUNDS; r++) {
  const redPrompt = r === 1
    ? `${REF}\n\n第 1 轮。请独立完成审查，按 schema 输出 JSON。`
    : `${REF}\n\n第 ${r} 轮。蓝方上一轮的意见如下，请**逐条**回应（撤回 / 坚持），再给出修订后的完整清单。\n\n<蓝方>\n${lastBlue}\n</蓝方>`

  await send(st.sessionRed, redPrompt)
  const red = await readReply(st.sessionRed)

  const bluePrompt = `${REF}\n\n第 ${r} 轮。红方本轮的结论如下。逐条判断（误报 / 真问题 + 严重度），并补充它漏掉的。最后按 schema 输出 JSON。\n\n<红方>\n${red}\n</红方>`
  await send(st.sessionBlue, bluePrompt)
  const blue = await readReply(st.sessionBlue)

  lastBlue = blue
  st.round = r
  st.history.push({ round: r, at: new Date().toISOString(), redChars: red.length, blueChars: blue.length })
  fs.writeFileSync(stateAt, JSON.stringify(st, null, 2))
  fs.appendFileSync(path.join(runDir, "transcript.md"),
    `\n\n## 第 ${r} 轮 · 红方\n\n${red}\n\n## 第 ${r} 轮 · 蓝方\n\n${blue}\n`)

  console.error(`[round ${r}/${ROUNDS}] red=${red.length}c blue=${blue.length}c`)
  if (isConverged(blue)) { console.error(`[round ${r}] 已收敛，提前结束`); break }
}

// —— 4. 裁判汇总 ——
const judge = await client.session.create({
  title: `[${RUN_ID}] 裁判`, agent: "review-judge", parentID: st.rootSessionID, location: LOCATION,
})
await send(judge.id, `${REF}\n\n完整辩论记录见 ${path.join(runDir, "transcript.md")}。请读它，然后输出最终报告。`)
const report = await readReply(judge.id)
fs.writeFileSync(path.join(runDir, "report.md"), report)
console.log(report)
```

要自己补的部分：

- `isConverged(blue)` —— 判断提前结束。最可靠的做法是让每轮输出**结构化 JSON**（见第 4 节的 schema），数"本轮新增争议项"；没有新增就停。别用正则数关键词。
- `--help`、参数校验、错误处理、`Service.discover()` 返回 null 时的提示。
- `client.session.wait` 是否存在、签名如何 —— **必须先探测**（第 6 节）。

### 3.6 这套设计里三个容易写错的地方

1. **每轮只传对方的增量，不传全史。**
   红方的会话里已经有它自己前几轮说过的话。第 r 轮给红方的 prompt 只需要放**蓝方上一轮输出**。把整个 transcript 每轮重贴一遍，token 按 O(N²) 涨，context 很快爆。
2. **共享材料落盘、只写一次。**
   diff 可能很大。写进 `context.md`，prompt 里只给路径。V2 的命令模板不展开 `@path`，所以这里必须靠 agent 自己 `read`——所以 prompt 里要明确说"去读这个文件"。
3. **循环在代码里，不在 prompt 里。**
   最要紧的一条。任何"让模型自己数着轮次循环"的写法都会失败：它会提前收尾、把 5 轮压成 1 轮。

### 3.7 断点续接

`state.json` 存了 `round` / `sessionRed` / `sessionBlue`，resume 就是：

```js
if (RESUME) return JSON.parse(fs.readFileSync(stateAt, "utf8"))   // round 从 st.round + 1 继续
```

两个会话还在磁盘上活着，直接继续发 prompt，上下文一点不丢。
**这就是"持久化 + 启动 + 续接"的完整闭环，没有用到任何未公开 API。**

---

## 4. 每轮 prompt 骨架

**红方第 1 轮：**

```
共享材料：{context.md 路径}（含 diff、文件清单、近期提交）。
要验证任何猜测，直接去读仓库里的真实文件，不要只凭 diff 推断。

第 1 轮。请独立完成审查，输出 JSON：
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

**红方第 r 轮**（它自己的会话里已有前几轮）：

```
第 {r} 轮。蓝方上一轮的意见如下，请**逐条**回应，每条只能选"撤回"或"坚持"：
撤回 → 说明它哪一点说服了你；
坚持 → 给出它没考虑到的具体反例。

<蓝方>
{blue 上一轮输出}
</蓝方>

然后输出修订后的完整 findings JSON。
```

**蓝方第 r 轮：**

```
第 {r} 轮。红方本轮的结论如下，逐条判断：
- 误报？（去读代码确认，已有防护就指出在哪一行）
- 严重度是否被夸大？
- 真问题 → 明确写"这是真的"

然后补充红方漏掉的问题。**不要为了显得有价值而制造分歧**，红方对了就说它对了。

<红方>
{red 本轮输出}
</红方>

输出 JSON。
```

---

## 5. 备选方案

### 方案 B：纯 Markdown 命令 + 内置 `subagent` 工具

V2 的 `subagent` 工具参数是 **`agent` + `sessionID`**（由 v1↔v2 映射确认：`agent`↔`subagent_type`，`sessionID`↔`task_id`），所以**比 V1 更可行**——理论上可以固定两个 `sessionID` 反复续接。

```markdown
---
description: 双 agent 辩论 review
---
第 1 轮：调 subagent 工具，agent=review-red，记下返回的 sessionID 为 A；
       再调 subagent，agent=review-blue，把红方结论给它，记下 sessionID 为 B。
第 2 轮：再用 subagent，**带 sessionID=A 续接**，把蓝方意见给它……
（重复 N 轮）
最后汇总。
```

**缺陷是结构性的**：循环由模型驱动。它随时可能少跑一轮、或忘了带 `sessionID` 导致每轮从零开始。V1 时代有过对应的 bug（subagent 失败时连 session id 都不返回，父 agent 完全无法恢复）。**只适合轮数少、能容忍不稳定的场景。**

### 方案 C：插件 + `ctx.session`（更原生，但 API 是 beta）

```ts
// .opencode/plugins/debate-review.ts
import { Plugin } from "@opencode-ai/plugin/v2"

export default Plugin.define({
  id: "debate-review",
  async setup(ctx) {
    const reg = ctx.tool.transform((tools) =>
      tools.add("debate_review", {
        description: "两个 agent 循环辩论式 review 代码改动，返回合并后的建议报告。",
        input: {
          type: "object",
          properties: {
            target: { type: "string", description: "git range，如 HEAD~3..HEAD" },
            rounds: { type: "number", description: "轮数，默认 3，上限 6" },
            runId:  { type: "string", description: "续接已有 run" },
          },
        },
        async execute(input, context) {
          const dir = ctx.location.directory
          const red = await ctx.session.create({ title: "红方", agent: "review-red" })
          const blue = await ctx.session.create({ title: "蓝方", agent: "review-blue" })
          // 循环体同 3.5，但用 ctx.session.prompt / ctx.session.context / ctx.session.wait
        },
      })
    )
    return () => reg.dispose()   // hook 注册随插件作用域自动释放
  },
})
```

好处：能被模型自己当工具调用；`ctx.session` 域的方法最全（`create` / `get` / `remove` / `context` / `switchAgent` / `switchModel` / `update` / `move` / `prompt` / `generate` / `command` / `compact` / `synthetic` / `interrupt` / `wait`）；`ctx.options` 能接配置；`ctx.event` 能订阅事件。

代价：**V2 插件 API 是 beta，相邻 build 之间 `ctx` 形状可能变**。建议做成方案 A 之后的增强。

---

## 6. ⚠️ 动手前必须先做的探测

V2 内部仍有 beta 面，且我上面标了若干"未在文档中给出签名"的地方。**别猜，跑这些**：

```bash
# 0. 二进制名与版本（V2 正式版与 V1 共用 opencode；dev build 可能是 opencode2）
opencode --version || opencode2 --version
opencode agent list

# 1. 拿本机真实的 OpenAPI spec —— 最权威的来源
curl -s http://127.0.0.1:4096/v2/openapi.json > /tmp/oc.json
jq -r '.paths | keys[]' /tmp/oc.json                                   # 端点全集
jq '.paths["/api/session"], .paths["/api/session/{sessionID}/prompt"]' /tmp/oc.json   # 建会话 / 发消息的确切 schema
jq -r '.paths | keys[] | select(test("wait|inbox|context"))' /tmp/oc.json             # ★ 找 wait 与 context

# 2. 用 operation ID 直接试调（V2 CLI 提供 api 子命令，走和 TUI 一样的发现/鉴权）
opencode api get /api/health
opencode api session.create -d '{"id":null,"parentID":null,"title":"probe","agent":null,"model":null,"location":{"directory":"'"$PWD"'"},"metadata":null,"permissions":null}'
opencode api session.prompt -d '{"text":"say hi"}'                    # 看返回是不是 Inbox.User（=不等回复）

# 3. 验证 agent frontmatter 是否被正确解析
#    写一个最小 agent 文件，重启后 opencode agent list 里出现，且没有字段无效的告警

# 4. 确认命令模板行为
#    写一个 /probe 命令：!`echo "$1"` ，执行 /probe hello，看输出有没有 hello

# 5. 确认服务端在跑 / 怎么起
opencode debug paths          # db / home / data / config / cache / state / tmp / bin / log / repos
opencode service --help       # 共享后台 service 的管理入口
```

**探测结果决定怎么写：**

| 探测发现 | 应对 |
|---|---|
| `POST /api/session` 确实接受 `agent` | 3.5 的写法直接可用 ✅ |
| 不接受 `agent` | 建完会话立刻调 `POST /api/session/{id}/agent`（`session.switchAgent`）再 prompt |
| 有 `wait` 且签名清楚 | `readReply` 用它 ✅ |
| 没有 `wait` | 退路 1：`ctx.session.generate` / `session.generate` 若同步返回结果，改用它做循环；退路 2：轮询 `GET /api/session/{id}/context`，看最后一条消息是否已从 user 变成 assistant |
| `context` 端点不存在 | 改用 `GET /api/session/{id}/message?order=desc&limit=1` |
| `session.create` 报 `location` 相关错 | `location` 可能要走查询参数而非 body |
| `opencode api` 子命令不存在 | 用 `@opencode/client` 或裸 `fetch` 打 REST |
| 插件 `ctx` 形状与文档不符 | 放弃方案 C，只用方案 A |

---

## 7. 已知的坑

1. **`prompt` 不等回复** —— V2 最容易踩的一个。不显式 wait 就会静默拿到上一轮的旧内容，而且**不报错**。（见 2.3a）
2. **`prompt` body 里没有 `agent`** —— agent 必须建会话时给，或 `switchAgent` 切。（见 1.4）
3. **`mode` 用 `all`**，别用 `subagent`。（见 3.3）
4. **权限 action 是 V2 名字**：`shell`/`edit`/`read`/`glob`/`grep`/`subagent`/`webfetch`/`websearch`/`skill`。用 V1 的 `bash`/`task`/`write`/`patch` 会不生效。
5. **权限规则"最后一条匹配的生效"** —— `"*": deny` 写在具体 allow 前面。
6. **命令模板不展开 `@path`** —— 想让 agent 看文件，得在 prompt 里明确给路径让它自己读。
7. **`` !`...` `` 不走 agent 权限流程**，且官方警告别把不可信参数放进 shell 块。
8. **`agents` / `providers` / `commands` / `models` 不能 V1/V2 嵌套混写** —— 只能顶层共存。
9. **`lsp` 配置被接受但没有效果** —— V2 不跑 language server、没有 LSP 工具、不产 diagnostics。
10. **`instructions` 被接受但不加载** —— 写 `AGENTS.md`。`CLAUDE.md` fallback 也没了。
11. **`general` agent 不能起 subagent**。
12. **轮数上限要设死** —— 两个高 `steps` 的 agent 互相刷，token 是乘法消耗。默认 3，硬上限 6。
13. **别用同一个 agent 起两个会话来"扮演两方"** —— 两个会话共享同一份 system prompt，人格差异全靠 prompt 硬凹，效果远不如两个真正不同的 agent 配置。这就是要建两个 agent 文件的原因。
14. **`experimental.*` 的东西别依赖**（`subagent_depth`、`policies`、`batch_tool` 等）。`policies` 虽然承载了 provider 白/黑名单，但属实验面。
15. **改环境变量要重启**；watched 配置目录下的文件改动会热重载。

---

## 8. 交付物清单

- [ ] `.opencode/agents/review-red.md` / `review-blue.md` / `review-judge.md`（V2 frontmatter）
- [ ] `.opencode/commands/debate-review.md`
- [ ] `.opencode/scripts/debate-review.mjs`（可执行，带 `--help`）
- [ ] `.gitignore` 加 `.opencode/reviews/`
- [ ] 注释或 README 写清：怎么起 service、怎么 resume、参数含义、**`wait` 用的是哪条路径**
- [ ] 一次真实演练：跑 → 中断 → `--resume` → 拿到最终报告

---

## 参考来源（全部为 V2 文档）

- [Intro](https://opencode.ai/v2/docs/) · [Config](https://opencode.ai/v2/docs/config/) · [Migrate from V1](https://opencode.ai/v2/docs/migrate-v1/) · [Troubleshooting](https://opencode.ai/v2/docs/troubleshooting/)
- [Agents](https://opencode.ai/v2/docs/agents/) · [Commands](https://opencode.ai/v2/docs/commands/) · [Permissions](https://opencode.ai/v2/docs/permissions/) · [Policies](https://opencode.ai/v2/docs/policies/) · [Tools](https://opencode.ai/v2/docs/tools/) · [Instructions](https://opencode.ai/v2/docs/instructions/)
- [Plugins](https://opencode.ai/v2/docs/plugins/) · [CLI](https://opencode.ai/v2/docs/cli/) · [CLI Commands](https://opencode.ai/v2/docs/cli/commands/) · [CLI Plugins](https://opencode.ai/v2/docs/cli/plugins/)
- [Build: Plugins](https://opencode.ai/v2/docs/build/plugins/) · [Build: Client (JavaScript)](https://opencode.ai/v2/docs/build/client/) · [Build: SDK](https://opencode.ai/v2/docs/build/sdk/)
- [API Overview](https://opencode.ai/v2/docs/api/)（OpenAPI 3.1，机器可读 spec 在 `/v2/openapi.json`）

---

## 附：一句话总结

> **OpenCode 的"续接"从来不是一个功能，就是"复用同一个 sessionID 再发一次 prompt"。**
> V2 的做法：`POST /api/session` 建两个**各自绑好 agent** 的持久会话（`agent` 字段就在这里给，**不在 prompt 里**）→ 把 id 存进 `state.json` → 用一个普通脚本在 `for` 循环里 `POST /api/session/{id}/prompt`，每轮**显式等这一轮跑完**再读 `context`。
> 轮次、终止、断点续接、报告汇总全在代码里，模型只负责每一轮的内容。
>
> **动手前先跑第 6 节的探测**，尤其是 `wait` 那条——V2 的 `prompt` 是投递语义，不等回复，这一步弄错整个循环会静默失效。
