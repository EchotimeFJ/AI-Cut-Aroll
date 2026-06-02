# AGENTS.md — AI-Cut A-Roll 项目指南

本项目包含两个 Codex Skill，用于将口播视频自动粗剪为剪映草稿。

## 项目结构

```
skills/
├── aroll-cut/SKILL.md     # 核心粗剪流程：转写→分析→剪映草稿
└── cut-draft/SKILL.md     # cutcli 完整命令参考（草稿/视频/字幕/特效/滤镜/转场等）
```

## 工作原则

1. **先读 SKILL.md**：`aroll-cut/SKILL.md` 是主流程，`cut-draft/SKILL.md` 是 cutcli 参考。
2. **必须过步骤**：aroll-cut 的 9 个步骤必须按顺序执行，不可跳过。
3. **Step 5 确认机制**：编辑决策预览必须展示给用户并等待确认，绝不可跳过。
4. **时间单位**：cutcli 所有时间参数使用**微秒（μs）**，1s = 1,000,000μs。
5. **命令名**：`cutcli`，不是 `cut`（`cut` 是系统命令）。
6. **剪映路径**：首次使用需检查剪映目录（JianyingPro 或 CapCut），配置 `cutcli config set-dir`。
7. **重启剪映**：草稿创建后需提醒用户重启剪映才能看到新草稿。

## 环境检查

在执行任何步骤前，先确保：
- `ffmpeg` 可用
- `whisper` CLI + Python 模块可用
- `cutcli` 可用
- 剪映桌面端已安装

## 增强效果（可选）

在基础的视频+字幕之上，可以添加：
- **转场**：用 `cutcli query transitions` 查询，添加到视频段 JSON 的 `transition` 字段（或直接修改草稿 JSON）
- **特效**：用 `cutcli query effects` 查询画面特效
- **滤镜**：用 `cutcli query filters` 查询，可选 `暗调电影` 等
- **差异化字幕**：每段字幕可用不同的入场动画（渐显/向下滑动/弹入跳动等）
