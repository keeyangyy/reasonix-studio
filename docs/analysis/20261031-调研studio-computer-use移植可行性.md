# 调研 studio computer use 移植可行性

- 优先级：低
- 类型：调研（新功能评估）
- 线：wails-mine（只读参考 `origin/studio-mine`，不在该线开发）
- 影响面：若移植，新增工具面（`computer_read` / `computer_act`）与外部驱动进程依赖；不动会话/配置等共享数据面
- 来源：「我想要了解studio的computer use方案，想看看能不能移植过来，这个优先级最低」

## 背景

- wails-mine 目前没有任何 computer use 能力（全库无 `computer_read` / `computer_act`）。
- studio-mine 有完整实现，且已在 2.39.0 从旧的 Swift helper 切换到 **CUA Driver**（release note：#12937）。
- 既有评估 [report/2026-10-06-wails可移植能力评估.md](../../report/2026-10-06-wails可移植能力评估.md) 的结论是「电脑操作：不建议移植（成本高、无障碍 API、跨平台）」，但那是 **v2.29.0 时代基于旧 helper 的判断**，CUA Driver 架构改变了前提，需要重新评估。

## 期望

- 读懂 studio 的 computer use 方案（架构、工具面、权限模型、打包方式）。
- 评估移植到 wails-mine 的可行性：可移植 / 条件移植 / 不建议，并给出成本量级与主要风险。
- 产出结论供后续决策，不在本需求内直接动代码。

## 已知事实（初步调研）

**架构（来自 `origin/studio-mine:docs/design/COMPUTER_USE.md`）**

- Studio host 持有一个 **CUA Driver**（上游 Rust 项目，锁定 0.34.1，MIT），通过 MCP 协议与 Go 侧通信；模型只看到 `computer_read`（`apps` / `snapshot` / `screenshot`）和 `computer_act`（点击、输入、按键、滚动、拖拽、launch 等）。
- 职责切分：**Reasonix 管授权与边界**（按应用审批、禁止应用、前台指针输入单独审批、会话所有权、轨迹投影）；**CUA 管 OS 能力**（无障碍树、截屏、输入注入、光标）。
- 驱动生命周期：host 持有一个 driver，多个 tab/会话共享；关闭控制器不关闭其他 tab 的 driver，host 退出负责最终清理；驱动代次（generation）变化会让旧的元素引用/截图失效。
- 打包：`internal/platform/computer/cua-driver.lock.json` 是唯一契约，Go、Electron 就绪检查、产物安装都读它；Electron 侧打包各平台二进制并校验 SHA-256；macOS 使用 Electron 私有内嵌服务。
- 安全边界：驱动无自更新、无感知模型安装、无远程桌面服务；`computer_act` 的 launch 必须精确身份 + 独立审批，且返回效果默认 unverified。

**代码面（studio 侧）**

- `internal/platform/computer/`：约 40 个文件，驱动会话、应用发现、窗口绑定、元素 token、截图坐标换算、效果核验、启动校验、三平台身份解析。
- `internal/tools/builtin/computer.go`：两个工具的定义与权限参数（按 `app` 主体审批）。
- `internal/session/control/approval_bridge.go`：`computer_use` 审批码与文案。
- `internal/assembly/boot/effect_computer_test.go`：装配层开关（无 helper 时工具不进 schema）。

**与 wails 线的差异**

- wails 没有 Electron 的打包/资源机制，但 Wails 安装器同样可以携带额外二进制，driver 又是独立进程 + MCP 协议，**理论上与外壳解耦**——这是与旧评估最大的不同。
- 权限体系不同：studio 是「按应用授权 + 显式审批」，wails 是另一套 allow/ask/deny 语义，需要单独设计映射，不能直接搬。
- 前端呈现：截图与元素引用需要聊天区支持图片/结构化快照展示，wails 已有 `view_image` 类能力可参考，但交互形态需要确认。

## 待确认

1. CUA Driver 的获取与分发：上游 prerelease 的稳定性、版本锁定策略、wails 安装器能否携带并校验二进制、是否需要签名。
2. 权限模型：wails 的审批体系如何表达「按应用授权」，是否接受与 studio 不同的语义。
3. 平台范围：是否只做 Windows（wails 主用平台），Windows 下 UIA/截屏/输入注入由 driver 承担，宿主侧需要多少额外工作。
4. 前端成本：截图、元素引用、操作轨迹在 wails 前端的呈现方式。
5. 长期维护成本：wails 线无上游供血，driver 升级/协议变更需要自行跟进。

## 交付物

- `docs/DeepSeek-Reasonix/report/<日期>-studio-computer-use移植评估.md`：架构说明、移植切面、成本量级、风险与结论。
- 若结论为「条件移植」，再拆出独立实现任务，不在本需求内实施。
