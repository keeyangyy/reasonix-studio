# 本线发版链与 CI 基础设施

| 项 | 值 |
|---|---|
| 分支 | `feat-studio/ci-release`（重建后新建；旧线对应主题的分支是 `ci/fork-release-pipeline`） |
| 起点提交 | `5d784ed6d`（基线 `studio-v2.41.0`） |
| 最新提交 | `b410f19e2`（重建后该分支共 4 个提交：`fd60763f2` → `d8808a37e` → `f9c9169aa` → `b410f19e2`） |
| PR |  |
| 状态 | 生效 |

## 改动

为 `studio-mine` 自带一套触发 / 打包 / 发版流程（`.github/workflows/` 三件套 + `ci-rerun.yml`）。

- **自建三件套，不动上游 `release-studio.yml`**：本线自带 `release-manual.yml` + `release-windows.yml`，并把 `ci.yml` / `studio.yml` 的触发分支改成本线——三线各带同名 workflow，靠 tag 归属断言与 `-r <分支>` 分开，互不串线。
- **只留 Windows**：本 fork 无 macOS runner，上游 macOS legs 直接移除。
- **打包串行**（`8ac6db6a8`）：冷缓存下两条 electron-builder 并行会撞 7-Zip 解压竞争，改串行。
- **自动重跑独立成 workflow**（`38305b5d5`、`d11143951`）：改为 `workflow_run` 触发的独立 `ci-rerun.yml`，且**只在 `run_attempt == 1` 时动作**；重跑条件改为逐个失败 job 匹配已知环境性 flake 特征。
- **发版兜底区间只取本线**（`499b0b30d`）：`--merged HEAD` 限定本线 + `grep -vx` 排除当前 tag + 按创建时间取最近发布点。
- **catalog 自覆盖修复**（`1c5fc5aff`）：写临时文件再 `mv`。

## 动机

fork 要发的是自己的 `v2.26.0-mine.N`，而上游发布链只认 `studio-v*` tag，本线必须自带一套流程，否则无从出包。

## 取舍

- **自动重跑必须独立成 workflow**：run 不能在运行期间 rerun 自己（GitHub 直接拒绝）；且 `workflow_run` 只认默认分支上的定义，故这份文件实际是仓库级、跟默认分支走。
- **重跑要有条件**：初期无条件重跑会把确定性失败也重跑一遍，白烧 runner、还把真回归伪装成抖动；改为匹配已知 flake 特征，任一失败不带特征就保持红并在 step summary 写明"该在本地复现修复"。
- **发版区间只取本线**：原先按版本号排序会混进另两条线的 tag（`v1.38.41` 的说明列过 60 条不相干提交；`v2.26.0-mine.2` 取到刚打的自己、留下空的「变更总览」）。
- **catalog 更新要先写临时文件再覆盖**：`Update the fork's own catalog` 把已有清单同时当输入（`existing=`）与输出（`>`）——重定向会在脚本执行**前**截断该文件，脚本随即读到空文件并报 `existing index ... is not a versions index`（信息误导，且只会红在发版最后一步）。首次发版与 wails 线之所以没暴露：它们的 anchor 还没有清单，走 `existing=-` 分支、根本不读文件；**只要 anchor 里已有清单（第二次发版起）必挂**。清单本身语义是对的（追加 → 同版本去重 → semver 倒序只留最新 `keep` 条，工作流传 3），故"自动维护、只留最新 3 个"不需改。

## 关键文件

`.github/workflows/` 下五个文件：`ci.yml`、`studio.yml`、`release-manual.yml`、`release-windows.yml`、`ci-rerun.yml`；另 `tools/repolint/baseline.json` 与本 fork 自己的发布目录。

## 备注

- **验证**：`v2.26.0-mine.1` 起各版本均由该链产出 zip + installer；双绿 gate 与 tag 归属断言在实测中拦下过不合规的发版；重跑机制在 flake 上跑通过、确定性失败保持红。
- **回退**：反向 patch 或 `git revert`；这些 workflow 是独立新增的，删掉即回到上游默认行为（代价是本线没有发版链）。
- 相关：skill `reasonix-flow`（发版章节）与 `references/ci-pipeline.md`；[report/2026-10-03-三线分支维护方案.md](../../report/2026-10-03-三线分支维护方案.md) 第 5、12、13 节。

## 重做记录（2026-10-16，基线 `studio-v2.41.0`）

> 注：本节上方正文里括注的提交号（`8ac6db6a8`、`38305b5d5`、`d11143951`、`499b0b30d`、`1c5fc5aff`、`1ab39fbc6`）属**旧线**（重建前的对应实现）——重建后同样的东西是重做的，重建后提交见下。

旧线起点 `1ab39fbc6`、最新 `1c5fc5aff`（基线上游 2.26）。本轮在 `studio-v2.41.0` 上重建同一套流程，两个提交：

- `fd60763f2`：本线发版链三件套（`release-manual.yml` / `release-windows.yml` / `ci-rerun.yml`）+ `tools/catalogfilter/`（每上游 base 保留最新 mine 与最近 N 条）+ `.gitignore` 排本地产物。
- `d8808a37e`：`ci.yml` / `studio.yml` 的 fork 化裁剪（去 macOS 腿与多平台），以及上游 `release-studio.yml` 的 `Build the Windows delta` 步骤加 `HAS_R2` 门——fork 没有 R2，上游原样会白算 delta。

与旧线的差异：上游 2.41 的 `release-studio.yml` 结构已改，`if: HAS_R2` 的落点须跟着上游结构走；本线不再改 `tools/repolint/baseline.json`。旧的 catalog 自覆盖缺陷（`1c5fc5aff`）由 `tools/catalogfilter` 自己写临时文件规避。
