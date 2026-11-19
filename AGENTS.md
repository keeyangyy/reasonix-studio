# Reasonix Studio

本仓库是 Reasonix 2.x 的接续线：以上游最终版 v2.61.0 为基线继续独立开发（Electron 桌面，版本号沿用上游序列）。上游已归档，本仓库不是官方发行版。

## 定位

- 只发 Windows 桌面端，以 Windows 为主要开发平台。
- 继承上游 2.x 的 Electron 桌面与既有功能，在此基础上新增与修复。
- 与 [Reasonix Wails](https://github.com/keeyangyy/reasonix-wails) 是两条独立接续线：本仓库功能全面，Wails 线是轻量单 exe。

## 工作流

- 主分支 `main`；前期可直接 push，后续按 wails 的方式建立 PR + CI 门禁。
- 文档结构：`docs/archive/`（归档）、`docs/analysis/`（分析）、`docs/decisions/`（决策）。
- 本地环境（机器路径、worktree、SMB 约束）由项目库 `AGENTS.md` 代管。

## 上游规则

@REASONIX.md