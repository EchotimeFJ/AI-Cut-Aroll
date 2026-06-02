# AI-Cut A-Roll — 剪映自动口播粗剪

基于 Codex 的 AI 驱动口播/教程视频粗剪工具。**丢素材 → 自动转写 → 智能识别好坏 take → 生成剪映草稿（含字幕、转场、特效）**，在剪映桌面端直接打开编辑。

```
素材文件夹
  → FFmpeg 提取音频（16kHz mono WAV）
  → Whisper 语音转写（逐词时间戳 JSON）
  → FFmpeg 静音检测
  → LLM 分析：保留/丢弃 take、去重、切静音
  → FFmpeg 预切保留片段
  → cutcli 创建剪映草稿 + 导入视频 + 字幕 + 转场 + 特效 + 滤镜
  → 生成 SRT 字幕备份
```

## 环境要求

| 依赖 | 安装方式 | 用途 |
|------|---------|------|
| FFmpeg | `brew install ffmpeg` | 音频提取、静音检测、视频切割 |
| OpenAI Whisper | `pip install openai-whisper` | 语音转写 |
| cutcli | `curl -s https://cutcli.com/cli \| bash` | 操作剪映草稿 |
| 剪映/CapCut | 剪映官网下载 | 打开和编辑草稿 |

## Skills

本项目包含两个 Codex Skill，配合使用：

### 1. `aroll-cut` — 核心粗剪逻辑

**位置**: `skills/aroll-cut/SKILL.md`

负责：素材扫描 → 音频提取 → Whisper 转写 → 静音检测 → LLM 剪辑点分析 → ffmpeg 预切 → cutcli 创建草稿。

### 2. `cut-draft` — cutcli 操作参考

**位置**: `skills/cut-draft/SKILL.md`

cutcli 完整命令参考手册，包括草稿管理、视频/字幕/特效/滤镜/转场/贴纸/关键帧/遮罩等操作。

## 安装（部署到 Codex）

```bash
# 克隆项目
git clone https://github.com/EchotimeFJ/AI-Cut-Aroll.git

# 安装 Skills 到 Codex
cp -r AI-Cut-Aroll/skills/aroll-cut ~/.codex/skills/aroll-cut
cp -r AI-Cut-Aroll/skills/cut-draft ~/.codex/skills/cut-draft

# 安装环境依赖
brew install ffmpeg
pip install openai-whisper
curl -s https://cutcli.com/cli | bash
```

安装后，在 Codex 中直接说「用 aroll-cut 处理 <素材文件夹>」即可。

## 使用方法

在 Codex 中输入：

```
用 aroll-cut 处理 ~/Videos/我的素材 竖屏
```

Codex 会按流程依次执行：

1. **扫描素材** — 列出文件夹内所有视频文件
2. **提取音频** — ffmpeg 批量转 16kHz mono WAV
3. **Whisper 转写** — 逐词时间戳 JSON
4. **静音检测** — 识别静音间隙
5. **分析剪辑点** — 结合标记词（好/OK/pass 等）判断保留/丢弃 → **展示预览等确认**
6. **预切视频** — ffmpeg 精确切割保留片段
7. **创建剪映草稿** — cutcli 一键生成含视频+字幕+转场+特效的完整草稿
8. **生成 SRT** — 标准字幕文件备份

## 保留/丢弃标记

录制时在满意的 take 后说标记词，工具自动识别：

| 标记词 | 作用 |
|--------|------|
| OK、好、keep、good、nice | 前面段落**保留** |
| pass、不要、cut、again、redo | 前面段落**丢弃** |

不用标记词也可以——工具会保留所有语音段，只切掉静音。

## 配置

所有参数都可以用自然语言覆盖：

| 参数 | 默认值 |
|------|--------|
| 分辨率 | 1920×1080（横屏）/ 1080×1920（竖屏） |
| 帧率 | 自动检测 |
| 语言 | 自动检测 |
| Whisper 模型 | medium |
| 静音阈值 | -50 dB |
| 最短静音 | 0.8s |
| 字幕字号 | 8 |
| 字幕动画 | 渐显/渐隐 |
| 转场 | 叠化、闪黑 |
| 滤镜 | 暗调电影 |

例如："用 29.97 帧率"、"静音阈值 -40dB"、"用 whisper large 模型"、"字幕改用向下滑动入场"。

## cutcli 配置

首次使用需配置 cutcli 草稿目录指向实际的剪映目录：

```bash
# macOS 剪映专业版
cutcli config set-dir "~/Movies/JianyingPro/User Data/Projects/com.lveditor.draft"

# macOS CapCut 国际版
cutcli config set-dir "~/Movies/CapCut/User Data/Projects/com.lveditor.draft"

# 验证
cutcli config show
```

## 生成的草稿效果

以一段 3 段口播素材为例，最终剪映草稿包含：

| 轨道 | 内容 |
|------|------|
| 视频轨 | 3 段精确剪好的视频，无缝拼接 |
| 字幕轨 | 3 条同步字幕，逐句差异化样式（渐显/向下滑动/弹入跳动） |
| 转场 | 段间叠化 + 闪黑转场 |
| 特效 | 第 2 段模糊效果 |
| 滤镜 | 全局暗调电影滤镜 |

> **注意**：剪映只在启动时扫描草稿目录。如果在剪映运行时创建草稿，重启剪映后即可看到。

## License

MIT
