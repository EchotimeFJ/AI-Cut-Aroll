---
name: "aroll-cut"
description: "自动口播视频 A-roll 粗剪：扫描素材 → FFmpeg 提取音频 → Whisper 转写 → 静音检测 → 智能分析剪辑点 → 用 cutcli 生成剪映草稿（视频片段 + SRT 字幕）。需要 FFmpeg + OpenAI Whisper + cutcli。"
---

# aroll-cut — A-Roll 自动粗剪（剪映版）

自动完成口播/教程类视频的 A-roll 粗剪。核心流程：

```
素材文件夹
  → FFmpeg 提取音频（16kHz mono WAV）
  → Whisper 语音转写（逐词时间戳 JSON）
  → FFmpeg 静音检测
  → LLM 分析：保留/丢弃 take、去重、切静音
  → FFmpeg 预切保留片段
  → cutcli 创建剪映草稿 + 导入视频片段 + 添加字幕
  → 生成 SRT 字幕文件
```

最终产出是一个**可直接在剪映桌面端打开的草稿**，包含剪好的视频片段和同步字幕。

## 首次使用：环境检查

在执行任何步骤之前，先确认环境：

```bash
# 1. FFmpeg
which ffmpeg && ffmpeg -version | head -1

# 2. Whisper（CLI + Python 模块）
which whisper && python3 -c "import whisper; print('whisper', whisper.__version__)"

# 3. cutcli
which cutcli && cutcli --version
```

如果 cutcli 未安装：
```bash
curl -s https://cutcli.com/cli | bash
```

缺少任何依赖时，明确告诉用户缺什么以及怎么安装，不要继续后续步骤。

## 所需参数

任务开始时确认以下参数，用户未提供就询问：

| 参数 | 是否必填 | 默认值 |
|------|---------|--------|
| 素材文件夹路径 | **必填** | — |
| 横屏 or 竖屏 | **必填** | 横屏: 1920×1080 / 竖屏: 1080×1920 |
| 项目名称 | 可选 | 从文件夹名推断 |
| 帧率 | 可选 | 从第一个素材自动检测 |
| 语言 | 可选 | 自动检测 |

用户可用自然语言覆盖任意默认值，如 "用 29.97 帧率"、"我的保留标记是 nice"、"静音阈值 -40dB"。

## 工作流

以下步骤按顺序执行。每步完成后向用户报告进度。

---

### Step 1: 扫描素材

扫描文件夹中的视频文件，支持格式：`.mp4` `.MP4` `.mov` `.MOV` `.mxf` `.MXF` `.avi` `.AVI`。

按文件名排序，列出所有找到的文件，让用户确认后再继续。

---

### Step 2: 提取音频

为每个视频提取 16kHz 单声道 WAV：

```bash
mkdir -p "{素材目录}/aroll_work"
for f in "{素材目录}"/*.{mp4,MP4,mov,MOV,mxf,MXF,avi,AVI}; do
    [ -f "$f" ] || continue
    name=$(basename "$f" | sed 's/\.[^.]*$//')
    ffmpeg -i "$f" -vn -ar 16000 -ac 1 "{素材目录}/aroll_work/${name}.wav" -y
done
```

---

### Step 3: Whisper 转写

```bash
for wav in "{素材目录}/aroll_work"/*.wav; do
    whisper "$wav" --model medium --word_timestamps True \
        --output_format json --output_dir "{素材目录}/aroll_work/"
done
```

- 模型默认 `medium`，可指定 `large` 提高精度
- 如果设置了语言，加入 `--language {LANGUAGE}`
- 输出 JSON 包含逐词时间戳

---

### Step 4: 静音检测

```bash
for wav in "{素材目录}/aroll_work"/*.wav; do
    name=$(basename "$wav" .wav)
    ffmpeg -i "$wav" -af "silencedetect=noise=-50dB:d=0.8" \
        -f null - 2> "{素材目录}/aroll_work/${name}_silence.txt"
done
```

默认参数：`noise=-50dB` `d=0.8s`，用户可自定义。

---

### Step 5: 分析剪辑点

合并转写结果 + 静音数据，判断保留/丢弃：

**标记词识别：**

| 口头标记 | 含义 |
|---------|------|
| OK、好、这条好、keep、good、nice | 前面段落**保留** |
| pass、不要、重来、cut、again、redo | 前面段落**丢弃** |

**规则：**
- 删除语音段之间的静音间隙
- 同一内容多遍录制 → 保留最后一个好 take（离 keep 标记最近的）
- 剪掉每段头尾的无声部分
- 全程无标记词 → 保留所有非静音语音段

**⚡ 必须在本步展示编辑决策预览并等待用户确认。** 预览格式：序号、源文件、出入点时间码、转录前几个词、保留/丢弃原因。绝不可跳过确认直接生成草稿。

---

### Step 6: 预切视频片段

用 ffmpeg 无损切割保留的片段（stream copy，不重编码）：

```bash
mkdir -p "{素材目录}/aroll_work/segments"

# 对每个保留段：
ffmpeg -ss {in_point} -i "{源文件}" -to {out_point} \
    -c copy -avoid_negative_ts make_zero \
    "{素材目录}/aroll_work/segments/seg_{序号}_{源文件名}"
```

记下每个片段文件的路径、分辨率和时长，后续 cutcli 需要使用。

---

### Step 7: 创建剪映草稿

使用 cutcli 创建草稿并添加视频片段和字幕。所有时间单位使用**微秒（μs）**（1 秒 = 1,000,000 μs）。

#### 7.1 创建草稿

```bash
cutcli draft create --width {宽度} --height {高度} --name "{项目名称}"
```

记录返回的 `<draftId>`。

#### 7.2 添加视频片段

将每个预切片段按时间线顺序排列，累加计算时间线位置：

```bash
cutcli videos add <draftId> --video-infos '[
  {"videoUrl":"{seg1文件路径}","width":{宽},"height":{高},"duration":{seg1时长μs},
   "start":0,"end":{seg1时长μs}},
  {"videoUrl":"{seg2文件路径}","width":{宽},"height":{高},"duration":{seg2时长μs},
   "start":{seg1时长μs},"end":{seg1+seg2时长μs}},
  ...
]'
```

- `start`/`end` 是片段在**剪映时间线**上的起止位置（μs），需按顺序累加
- `duration` 是预切片段文件的时长（μs），用 ffprobe 获取
- 同一轨道内片段时间不重叠

#### 7.3 添加字幕

将转写结果转为 cutcli 字幕格式，字幕时间码对齐到剪映时间线（与视频片段相同的累加逻辑）：

```bash
cutcli captions add <draftId> --captions '[
  {"text":"今天我们来聊聊该如何选股","start":0,"end":2780000},
  {"text":"如何在当前形式下选出最适合自己的股票","start":2780000,"end":6760000},
  {"text":"我们来看这张图","start":6760000,"end":7840000}
]' --font-size 8 --text-color "#FFFFFF" --bold \
   --in-animation "渐显" --out-animation "渐隐" \
   --in-animation-duration 500000 --out-animation-duration 500000
```

字幕选项：
- `--font-size`：推荐 6-12，竖屏常用 8
- `--text-color`：默认白色
- `--bold`：加粗
- `--in-animation` / `--out-animation`：入场/出场动画，用 `cutcli query text-animations` 查询
- `--in-animation-duration` / `--out-animation-duration`：动画时长（μs），建议 500000（0.5 秒）

---

### Step 8: 生成 SRT 字幕

同时生成标准 SRT 文件作为备份：

- 基于时间线时间（而非源文件时间）
- 时间码对齐到剪辑后的时间线
- 保存为 `{素材目录}/aroll_work/{项目名称}.srt`

---

### Step 9: 验证结果

执行后验证：

```bash
# 查看草稿信息
cutcli draft info <draftId>

# 列出视频片段
cutcli videos list <draftId>

# 列出字幕
cutcli captions list <draftId>
```

报告结果：
- 草稿 ID 和名称
- 剪映草稿目录路径（macOS 默认 `~/Movies/CapCut/User Data/Projects/com.lveditor.draft/{draftId}/`）
- 视频片段数和总时长
- SRT 文件位置
- 提示用户用剪映桌面端打开草稿

---

## 可调配置表

| 参数 | 默认值 | 说明 |
|------|-------|------|
| 分辨率 | 1920×1080 / 1080×1920 | 横屏/竖屏 |
| 帧率 | 自动检测 | 从素材获取 |
| 语言 | 自动检测 | Whisper 语言参数 |
| Whisper 模型 | medium | tiny / base / small / medium / large |
| 静音阈值 | -50 dB | 低于此电平视为静音 |
| 最短静音 | 0.8s | 最短切分间隙 |
| Keep 标记 | OK, 好, keep, good, nice | 标记好 take |
| Drop 标记 | pass, 不要, cut, again, redo | 标记坏 take |
| 字幕字号 | 8 | 竖屏推荐 8，横屏推荐 10 |
| 字幕动画 | 渐显/渐隐 | 入场/出场动画 |

用户可用自然语言覆盖任意参数。

## 关键规则

- 一个文件夹 = 一个项目 = 一个剪映草稿 + 一个 SRT
- 所有片段按拍摄顺序排布在一条时间线上
- WAV / JSON / 预切片段等中间文件保留至用户要求清理
- 编辑决策必须先展示、确认，再执行 — 绝不跳过
- **cutcli 所有时间参数使用微秒（μs）**，1 秒 = 1,000,000 μs
- **命令名是 `cutcli`，不是 `cut`**（`cut` 是系统命令）
- 预切视频片段使用 ffmpeg stream copy（`-c copy`），速度快且无损
