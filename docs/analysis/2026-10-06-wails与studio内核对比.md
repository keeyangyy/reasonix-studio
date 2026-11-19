# wails-mine 与 studio-mine 内核对比（除前端/外壳）

> ⚠️ **快照已过时（截至 2026-10-08）**：本文写死了分析时的基准 tag，两线此后都已前进（wails `v1.38.45` → `v1.38.49`、studio `v2.29.0-mine.1` → `v2.31.0-mine.1`）。**结构性结论（分层、包级差异、共同祖先）仍可用；具体行数、包数、提交数等数字不可再引用**。要用最新数字就重跑一次对比。

> 分析日期：2026-10-06
> 对比基准：tag `v1.38.45`（wails 线最新发布，2026-10-05）vs tag `v2.29.0-mine.1`（studio 线最新发布，`Latest`，2026-10-06）
> 共同祖先：`23bdf0813`（上游 2026-08-12 分家点）；此后 wails 线约 1500 个提交、studio 线约 3000 个提交
> 方法：`git` 只读对比两个 tag（工作树检出在 `main-v2-mine`，全程未使用工作树文件）+ 5 个并行子代理分域取证 + 主代理抽样核验
> 范围：**不含**前端 UI / Wails-vs-Electron 外壳差异（用户明确不关注）
> 相关：`2026-10-03-三线分支维护方案.md`、skill `reasonix-flow`、上游官方 `docs/MIGRATING.md`、`docs/ROADMAP.md`、`docs/SPEC.md`

## 结论摘要

1. **不是版本差异，是两代内核**：A 的 101 个平铺包在 B 里重组为 12 个领域包（约 200 子包），其中**多数是搬家**（无实质差别）、一部分是**同能力重写**（行为会变）、一部分是**全新能力**。
2. **差别最大的是上下文管理（第②层）**，其次是委派（第④层）与安全审批（第⑤层）。
3. **agent 循环本体没换**（两边 `run_loop.go` 同名同构），差别在循环外围。
4. **技能 / 记忆 / 配置 / 密钥 / 项目指令完全共享**（同一个 `%APPDATA%\reasonix`）；会话不丢、不被改写，但**续写会分叉成同名两份**。
5. **上游 2.x 尚未 GA**：`ROADMAP.md` 的 G1/G3/G4/G5 仍 Open，其中 **G3「会话与配置格式冻结」未达成** → 现在迁移是"并存试用"，不是"终局迁移"。

---

## 0. 量化对比（实测）

| 维度 | A `v1.38.45` | B `v2.29.0-mine.1` |
|---|---|---|
| `internal/` 顶层包 | 101 个平铺包 | 12 个领域包（assembly/base/contract/ext/frontend/model/platform/runtime/safety/session/state/tools），域下约 200 子包 |
| `internal/` 体量 | 2831 文件 / 22.35 MB | 4143 文件 / 27.50 MB |
| agent 核心 | `internal/agent` 560 文件 / 4.42 MB | `internal/runtime/agent` 401 .go；`internal/runtime` 整体 637 文件 / 4.37 MB |
| 循环文件 `run_loop.go` | 551 行 | 753 行 |
| 内置工具实现文件 | `internal/tool/builtin` 38 个 | `internal/tools/builtin` 70 个 |
| provider 可见核心工具 | 14 个 | 19 个 |
| `cmd` / `sdk` | 71 / 20 文件 | 106 / 48 文件 |
| 外壳（供参考，不在对比范围） | Wails+WebView2，`desktop/` 2325 文件 / 22.1 MB | Electron，`desktop/` 933 文件 / 16.7 MB |
| 全仓 diff | — | 10535 文件、+754017 / −1036697 行（重命名检测被 git 跳过，仅量级参考） |

---

## 1. 分层地图

```
⑦ 接口与外壳   CLI / TUI / serve / ACP / desktop      ← 主要是命令与语义变化
⑥ 会话与状态   session 格式 / catalog / 状态域 / 并发保护 ← 换代最彻底
⑤ 安全与审批   权限模式 / 信任文件夹 / 沙箱 / 出网       ← 语义变了
④ 委派         子代理 / fleet / 写声明 / claim 裁决      ← B 强化最多
③ 工具面       工具清单 / 写声明契约 / schema 预算        ← 有增有减
② 上下文       压缩 / 检索 / 记忆                        ← 差别最大
① 循环         turn loop                                ← 骨架同源
```

---

## 2. 第①层：循环（骨架没换）

**A**（官方 `docs/AGENT_CORE_SIMPLIFICATION.md`，A 自己的行为契约）：

```text
build request → provider stream → clean final: 结束
                                → tool call: 执行，下一步
                                → request error: 统一重试
                                → 未处理错误: 显式失败
```

**B**（官方 `docs/SPEC.md` §3.4）：

> `Run(ctx, input)` loop：build `Request`（带工具 schema）→ `provider.Stream` → 实时打印文本增量 → 收集完整工具调用 → 无调用则结束，否则执行工具、追加结果 → 重复，受 `maxSteps` 约束。

两边同名函数都在：`beginRunTurn` / `runToolLoop` / `handleFinalResponse` / `handleToolRound` / `pairUnexecutedGraceCalls` / `unavailableContextualToolCalls`。

**差别只在"流"与"出错恢复"**：

| 项 | A | B |
|---|---|---|
| 流式输出 | 直接 emit | `deferredStreamSink`（reasoning-aware，推理内容显示时机可控） |
| 采样恢复 | `emitProtocolRetry` + `sleepRecovery` | `streamWithSamplingRecovery` + `silentSinceLastToolRound`（针对"模型沉默"） |
| 轮内记录 | 无 | `recordTruncationFact`、`settleRewritesBeforeSampling`、`openUserTurn` |

**双模型协作两边都有**（A 有 `internal/agent/coordinator.go` + `planner_model`；B 有 `internal/runtime/coordinator`），不是新增。B 的 §3.5 描述了同一机制：planner 与 executor 各在自己 session 里跑以保持 prefix 缓存稳定，planner 只用只读研究工具，plan 以结构化文本交给 executor；路由是确定性宿主策略（executor-only / light / full / plan-for-approval / plan-only），不调用分类模型。

---

## 3. 第②层：上下文管理（差别最大）

**A：多阈值 + 投影/裁剪。** 旧机制键都在（`internal/config/config.go`、`internal/agent/projection.go` 命中）：
`soft_compact_ratio`、`tool_result_snip_ratio`、`compact_force_ratio`、`cold_resume_prune`、`context_editing`。

**B：旧键全部移除，只剩一个阈值。**（官方 `docs/SPEC.md` §3.6）

- canonical transcript **append-only，永不重写**（cache-first；任何重写都会破坏 prompt 缓存）
- 唯一自动触发：`agent.compact_ratio`（默认 **0.85**，范围 (0,1)），`triggerTokens = context_window × ratio`
- 触发时执行**一次** summary transaction：`stable prefix + 一份结构化 digest + 最近原文 tail`；候选 ≤ 窗口 50%，实际落 10%~30%
- 巨型工具结果**首次进入模型时**压到 ≤32KB（`Content` 可见 / `RawContent` 原始分离，provider 序列化与缓存哈希不含 RawContent）
- 后续阈值把"旧 digest + 新历史"合并成一份，不做多段合并、不做应用层重试
- 旧键（`soft_compact_ratio` 等）在普通启动时被移除、运行时被忽略；不使用 provider 原生 tool clearing

**B 新增三个面向模型的上下文工具**（A 没有）：`context_budget`（距自动压缩还剩多少）、`recall`（回读被折叠内容）、`history`（BM25 检索历史会话）。

**B 的三层上下文设计规则**（官方 `docs/SESSION_MEMORY_RETRIEVAL.md`）：指令（`AGENTS.md`/`REASONIX.md`，必须每轮在场）/ 背景记忆（会过期的事实）/ 会话历史（原始措辞），三者权限层级不同——事实不会被悄悄升格成命令。记忆侧还有：`subject_key` 冲突模型（一个 scope 一个 subject 只允许一个 active 值）、freshness 分级（reference 14/45 天、project 30/180 天、user+feedback 90/365 天）、自动 recall 上限（默认 ≤4 条 / 2400 字符）、`.migration-backup/` 原文备份。

**影响**：长会话的取舍行为会变；**A 上配的 compact 相关参数在 B 全部无效**，只有 `compact_ratio` 生效。

---

## 4. 第③层：工具面与契约

**provider 可见核心工具面**（两边 golden 快照 `testdata/golden/tool_schemas.json`）：

| | A（14 个） | B（19 个） |
|---|---|---|
| 共有 13 个 | `ask` `bash` `bash_output` `complete_step` `compress` `edit_file` `kill_shell` `read_file` `todo_write` `update_goal` `use_capability` `wait` `write_file` | 同左 |
| B 新增 6 个 | — | `await_user`、`browser_open`、`browser_read`、`browser_act`、`context_budget`、`recall` |
| A 独有 | `view_image` | —（并入 `read_file`） |

两点补充：

- 两边**都**用 `use_capability` 代理可选工具，所以 `glob`/`grep`/`ls`/`web_fetch`/`task`/`skill` 这类不在这张表里（两边都有）。
- B 的完整内置工具表（官方 `docs/TOOL_CONTRACT.md`）共 29 个，比 A 多出 `code_index`、`computer_act`/`computer_read`、`render_chart` 等重新登记的项。

**契约层变化**（对写锁/沙箱体验有直接影响）：

- A：写声明走 `DeclareWriteAccess`，bash 另有 `additional_write_dirs` / `justification` 参数
- B：统一成 `WritePaths` 一个接口，bash 变成"不透明写者、持全锁"
- B 新增 **schema 预算棘轮**（`internal/tools/builtin/schema_budget.json`，`totalTokens = 7050`）+ `Registry.Restrict` + stable/contextual 缓存分段（工具集变化不再打碎 provider 前缀）
- B 的 `docs/SPEC.md` 定义了三档 execution setting（light / balanced / delivery）共用同一精简核心面，可选工具留在 host registry 供 `use_capability` 派发

---

## 5. 第④层：委派（子代理）

**共同点**：`task` / `parallel_tasks` / `read_only_task` / `fleet` 两边都有；`write_paths` 声明两边都有；`runAs: subagent` 档案机制两边都有。

**B 的强化**（官方 `docs/SPEC.md` §3.10~3.17）：

| 机制 | 内容 |
|---|---|
| 子代理**裁决式收尾** | 子代理不能只声称完成，必须以宿主裁决过的 claim 结束；`reasonix run --json` 有 `false_completions`（宿主拒绝背书的声明数）、`criterion_downgrades` |
| 写声明**强制** | "Write claims are enforced, not advisory"；越界记为 `write_scope_violations` |
| 上下文继承**显式** | 子代理拿到什么上下文是显式声明的 |
| `fleet` = 小依赖图 | 支持 `depends_on`、`fail_fast`，路径冲突在 preflight 失败 |
| 统一 child 构造入口 | `internal/runtime/agent/spawn_boundary_test.go` 守着，禁止新旁路 |
| **MCP 并发策略** | "read-only ≠ stateless"；`[[mcp.servers]] concurrency = "serial"｜"parallel"`，浏览器类 server（browser/playwright/puppeteer/chrome/chromium/selenium）默认 serial |
| 并发上限可配 | `agent.max_subagent_concurrency = 6`、`agent.max_parallel_writers = 3` |
| 档案 CLI | `reasonix subagent create/list/edit/delete/try/run`；模型/effort 五级优先级（per-profile 覆盖 > 调用参数 > profile frontmatter > 全局默认 > executor 默认） |
| 委派度量 | `reasonix run --json` 输出 `subagent_runs`、`subagent_nested_runs`、`duplicate_work_paths`、`completion_reports`、`false_completions`、`write_scope_violations` |
| 择优选（新增） | `best_of_n`：多候选各自在独立 worktree 跑，judge 选出落地者（A 全库零命中） |

**B 官方自述的委派实测**：强制委派比不委派贵 **2.4~4.5 倍** token（不同任务形态 3.8x / 1.5x / 2.6x / 2.4x），因此把委派设计成"刹车优先"；33 次可委派运行里只有 15% 真的委派了。

---

## 6. 第⑤层：安全与审批

**权限模式语义整体更换**（官方 `docs/MIGRATING.md`）：

| A（1.38） | B（2.29） |
|---|---|
| `--permission-mode workspace-write` | **Auto** |
| `--permission-mode danger-full-access` | **Yolo** |
| `--permission-mode read-only` | Read only（**任何写都拒绝**） |
| `--yolo` / `bypassPermissions`（实际等于 workspace-write） | yolo = **真的跳过常规审批**（行为不同） |
| 默认 = Workspace | 默认 = **Auto**（信任的文件夹走 Auto，否则 Ask） |
| Shift+Tab 循环 Workspace / YOLO / Plan / Read only | Shift+Tab 循环 Auto / YOLO / Plan / Read only / **Ask** |

**B 新增"信任文件夹"流程**：可信任、写入受沙箱限制且尚无决定记录的文件夹，启动前问 `Trust this folder?`；`reasonix trust` / `reasonix trust --revoke` 管理。**未信任文件夹里的 headless run 会拒绝写与执行**（`permission.untrusted_folder`，退出码 4；`permission_denials` 列出被拒工具）——A 会照做，所以脚本行为会变。Auto 对嵌套/间接 shell（如 `python3 -c '...'`）会先问 → **提示变多**。

**B 新增安全机制**：`egress`（域名级出网 allowlist 代理）、`redirectguard`（重定向限制）、`typesafe`、sandbox 的 `authority`（SSH agent / docker / podman 管控）、`egress_bridge`、`protects`；shellsafe 拆出 `delete` / `git_options` / `inline_code` / `readonly_args` 护栏；权限侧新增只读模式、结构化 Verdict、工具组授权、`write_fence`。

**反向**：A 的 `runtimepolicy`（散文约束引擎，如 `ForbidTests` / `AllowsExternal`）在 B **整个包被移除**（B 全库 grep 零命中），改用 `agentpreset` / `taskpolicy` / `planmode` 表达。

---

## 7. 第⑥层：会话与状态

### 7.1 完全共享的部分（迁移时不用动）

两边共用 home `%APPDATA%\reasonix`：

| 数据 | 路径 | 共享 |
|---|---|---|
| 全局配置 | `<home>/config.toml` | ✅（B 保留它不用的段落如 `[bot]`，原样写回） |
| provider 密钥 | `<home>/.env` | ✅ |
| 斜杠命令 / 技能 / hooks | `<home>/commands/`、`<home>/skills/`、`<home>/settings.json` | ✅ |
| 记忆 | `<home>/memory/`、`<home>/projects/` | ✅ |
| 项目指令 | 项目内 `REASONIX.md` / `AGENTS.md` / `CLAUDE.md` | ✅ |
| 会话 | `<state root>/sessions/`（1.38.8+ 另有 `sessions-v4/`） | ⚠️ 部分，见下 |

### 7.2 会话互操作：官方六条规则

wails 线会话是 `sessions/<id>.jsonl` + `.events.jsonl`（schema1，另写 schema2 DAG 日志），属于"2.x 直接能打开"类别。

| 规则 | 内容 |
|---|---|
| S1 | 2.x **永不写** 1.x 保存的会话文件；打开再离开不改动 |
| S2 | 在 2.x 续写 1.x 的 `sessions/` 会话 → **第一轮运行前**移动成新的同名 2.x 会话并告知；1.x 原件保留 |
| S3 | 1.x 1.38.8+ 打开旧会话时复制到 `sessions-v4`；`sessions/` 下副本原样保留 |
| S4 | `sessions-v4` 会话在 2.x 首次打开时导入为 `sessions/v4-<id>.jsonl`；无 2.x 轮次时下次打开刷新导入，有则两份各自独立 |
| S5 | 2.x 写的 `.wire.jsonl` / `.adjudication.jsonl` / `.execution.jsonl` 会被 1.x 当成多余会话列出（1.x 侧应忽略） |
| S6 | `reasonix session list --json` 只列 2.x transcript |

B 的 `internal/state/sessionv4` 文档自述：**只读**、不加锁、不写缓存、不截断撕裂尾部；根目录每个字节归 1.x 所有。

**影响**：会话不会丢、不会被改写；代价是**续写会分叉成两个同名会话**。

### 7.3 数据面

- **catalog（可丢弃 SQLite 索引）**：代码上 A 有 4 个（`session-catalog/v8.sqlite`、`history-search`、`usage-catalog`、`task-catalog`），B 的 `internal/` 只剩 2 个（`history-search`、`usage-catalog`）。会话列表在 B 改为直接读权威 JSONL + sidecar。
  ⚠️ **B 自己的 `docs/CONFIG_PATHS.md` 仍写着 4 个**（含 `session-catalog/v3.sqlite`、`task-catalog/v1.sqlite`）——文档与代码不一致，不能当"已冻结事实"用。
- **B 新增 8 个状态域**（用途取自各包 `doc.go`）：

| 域 | 用途 |
|---|---|
| `execjournal` | "一次委派执行存在过"的持久记录。理由：transcript 只在**一轮结束时**追加，进程死在轮中会把整轮（请求、派发、每一项）带走，fan-out 子任务对下一进程**不可见** |
| `execgraph` | 从持久事实重建运行图（execjournal + 子代理 store 折叠成与实时图同一套词汇） |
| `observation` | 按宿主策略拍工作区快照（内容寻址目录树；文件按 size/mtime/ctime/inode 的 stamp 识别，不算哈希） |
| `trustedstate` | 可信宿主状态：内容寻址对象 + 每流一条记录链，链尾 head 指名最新记录与代；自述只能保证"相对于已观察的 head，历史没变" |
| `schedule` | 定时运行的持久存储 + 预算策略（一机一份 manifest；只决定槽位能否开，不起进程；策略只能收紧不能放宽） |
| `workgroup` | 把持久轨迹折叠成工具调用、再折叠成"一个生产者的一段工作"（用于测度可折叠单元） |
| `workspacelist` | 侧栏记住的项目列表文件（serve 按请求编辑、存储重定位时合并旧副本） |
| `usagereport` | 用量报表（无包文档） |

- **并发写保护换代**：A 用 DAG-head（`HeadEventForkedConcurrent`）→ B 用 **revision + snapshot 前缀冲突**（`stale_prefix` / `diverged`）。

**要点**：fork 在 wails 线上折腾过的"会话切换重复渲染""另一个窗口在写"属于持久化时序问题；B 对这一层的答案是把委派与执行变成**可持久、可重建、可审计的事实**（execjournal / execgraph / trustedstate）。

---

## 8. 第⑦层：接口与外壳

- **进程模型**：A 有 `reasonix-launcher`、`reasonix-legacy-migrator`；B 换成 `reasonix-studio-host`、`studio-delta`。CLI 主体命令名两边一致（`run` / `serve` / `web` / `acp` / `mcp` / `setup` / `doctor`）。
- **CLI 语义差异**（官方 `docs/MIGRATING.md`）：

| A（1.38） | B（2.29） |
|---|---|
| `reasonix bot`（IM 网关） | **移除**；`[bot]` 配置段落保留 |
| `/recover-context`、`/continue-checks` | **退役** |
| `/web` 是 slash 命令 | 不再是；改用 shell 里 `reasonix web` |
| — | 新增 `/version`、`/feedback`、`/locate`、`/setup`、`/auth` |
| `REASONIX_CHROME` | **不再读**；改 `[browser] executable`（默认找 Chrome/Edge/Chromium） |
| `REASONIX_SESSION_LOG=v1` | **移除**；B 只写一种会话格式 |
| `compact-ratio` 接受 30~85 | 接受 (0,100)，**默认值不同**；状态行改为 `to compaction N%` |
| 未知 `/command` 当普通消息发 | 相同 |

- **TUI**：B 的 `ROADMAP.md` 显示终端接管 T1~T6 全 Done（流式、审批面板、`!` shell、`/resume`、Ctrl+V 贴图），只剩 Windows 手工项 T7（IME 候选窗位置、Alt+V 贴图）未开始。

---

## 9. 升级判断

**方向明确，但现在不是"终局迁移"的窗口。**

支持升级：1.38 线已无上游供血（所有修复自研）；2.x 是上游唯一活跃线，能力面持续扩张（浏览器/电脑操作、best_of_n、出网白名单、插件市场、模型接入扩展）。

不建议一刀切：这不是版本升级而是换内核+换外壳；且上游 `docs/ROADMAP.md` 的 GA 门未全绿：

| GA 门 | 状态 |
|---|---|
| G1 终端接管全部完成 | **Open**（T7/T8/T9 未做） |
| G2 能打开 1.x 会话 | Met |
| G3 **会话与配置格式带版本 + 迁移路径，并在 2.x 线内冻结** | **Open** |
| G4 连续两周无 P0/P1 | Open |
| G5 1.x→2.x porting 清单（W3）逐项完成 | **Open**（W3 = Not started） |
| G6 签名自更新构建 | Met |

配套事实：M2 写着 **1.x 维护在 2.x GA 之后 3 个月结束**（proposed，backstop 日期未定）。

**G3 未达成 = 2.x 的会话/配置格式还会变**。合理路径是**并存试用**（共享技能/记忆/配置，会话不丢不被改写，只需守"同一项目同时只开一版"的纪律），等 G1/G3/G4/G5 全绿后再做一次性迁移。

---

## 10. 未验证 / 存疑

1. **catalog 数量**：代码显示 A 4 个 / B 2 个，但 B 的 `CONFIG_PATHS.md` 仍写 4 个 → 文档与实现不一致；"`task-catalog` 消失说明任务不落库"未穷尽核对（B 是否由 `state/schedule` 承担同类职责未验证）。
2. **`web_search` 工具归属**：子代理称 A 有独立 `web_search` 工具、B 并入 provider 端 server-side search；主代理核验时两边都有该字符串，**存疑**。
3. **`view_image` 的去向**：B 已无该名字，但合并到 `read_file` 的具体路径未确认。
4. **第④/⑤层"重写类"差异的实际行为影响**：全部是代码/文档级判断，**没有实测对比**。
5. **B 新增状态域与 A 的等价物映射**：只核了一部分。
6. **fork 自研修复在 2.29 的覆盖情况未核对**（wails 线的会话切换重复渲染 / 前端会话重构 / 三问题终局）—— 决定迁移人力成本的关键项，尚未做。

## 11. 上轮子代理报告的修正

- ~~B 新增 Coordinator 双模型~~ → **A 本来就有**（`internal/agent/coordinator.go` + `planner_model`），两边同源。
- ~~B 新增 `await_user`（模型可主动提问）~~ → A 已有 `ask` 工具，能力等价；`await_user` 是 B 的另一个变体（两者 B 都有）。
- ~~SQLite catalog 由 4 减到 2~~ → 仅代码层面成立，B 官方文档仍写 4 个（见第 10 节第 1 条）。

---

## 附：证据命令（PowerShell，仓库根 `D:/project/prj-github/DeepSeek-Reasonix`）

```powershell
# 基准与结构
git log --oneline -1 v1.38.45 ; git log --oneline -1 v2.29.0-mine.1
git merge-base wails-mine studio-mine
git ls-tree --name-only v1.38.45:internal ; git ls-tree --name-only v2.29.0-mine.1:internal
git ls-tree -r --name-only <tag> -- internal/<pkg> | Measure-Object -Line

# 体量
git ls-tree -r -l <tag> -- internal   # 文件数 + blob 字节数

# 工具面（golden 快照权威）
git show <tag>:internal/boot/testdata/golden/tool_schemas.json            # A
git show <tag>:internal/assembly/boot/testdata/golden/tool_schemas.json   # B

# 上下文本代
git grep -ln -E 'soft_compact_ratio|tool_result_snip_ratio|compact_force_ratio|cold_resume_prune|context_editing' v1.38.45 -- internal
git show v2.29.0-mine.1:docs/SPEC.md                      # §3.6 上下文管理
git show v2.29.0-mine.1:docs/SESSION_MEMORY_RETRIEVAL.md

# 委派 / 安全 / 工具契约
git show v2.29.0-mine.1:docs/SUBAGENT_PROFILES.md
git show v2.29.0-mine.1:docs/TOOL_CONTRACT.md
git show v2.29.0-mine.1:docs/MIGRATING.md
git grep -ln 'runtimepolicy' v2.29.0-mine.1 -- internal    # 应为空

# 会话与状态
git show v2.29.0-mine.1:internal/state/sessionv4/doc.go
git show v2.29.0-mine.1:docs/CONFIG_PATHS.md
git grep -h -o -E '(session-catalog|history-search|usage-catalog|task-catalog)[a-z0-9/.-]*' <tag> -- internal

# 升级窗口
git show v2.29.0-mine.1:docs/ROADMAP.md
```
