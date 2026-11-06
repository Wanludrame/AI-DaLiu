---
name: short-film-maker
description: "AI短片制作引擎：从小说/故事文本自动生成完整短片。全流程自动化：文本分析→分镜脚本→AI图片生成(Seedream 5.0 pro)→AI视频生成(Seedance 2.0 系列)→TTS旁白(edge-tts 分句)→字幕→BGM→最终组装。"
---

# AI 短片制作引擎

从一篇小说或故事文本出发，全流程自动制作一部完整的短片视频。

## 用户输入

用户提供：
1. **文本来源**（必选）：小说/故事的文件路径，或直接粘贴的文本内容
2. **目标时长**（可选）：如"约90秒"、"2分钟"，默认60-120秒
3. **风格偏好**（可选）：如"冷峻科幻"、"温暖文艺"、"悬疑惊悚"，默认根据文本自动推断
4. **旁白声音**（可选）：默认男声低沉克制，可选女声温柔
5. **画面比例**（可选）：默认16:9横版（1920×1080），可选9:16竖版

## 输出

最终产出文件保存在项目目录中：
```
短视频脚本/<标题>_短片脚本.txt          # 分镜脚本
短片素材/<项目名>/                       # 素材根目录
  shot<NN>_<desc>.jpg                   # 概念图
  shot<NN>_<desc>.mp4                   # AI生成视频
  audio/narr<NN>.mp3                    # 旁白音频
  bgm_<name>.mp3                        # 背景音乐
  temp/                                 # 中间处理文件
    proc_<NN>.mp4                       # 处理后的视频片段
    subtitles.srt                       # 字幕文件
    concat_video.mp4                    # 拼接视频（无音频）
    narration_full.wav                  # 完整旁白音轨
  <标题>_完整版.mp4                      # 最终输出
```

---

## 一、技术栈与 API 配置

> **模型版本核对（2026-11-06）**：依据火山方舟《模型下线公告》第十批，`doubao-seedream-5-0-260128`、`doubao-seedance-1-0-pro-250528`、`doubao-seedance-1-0-pro-fast-251015` 于 **2026-11-24 14:00 关停且不自动迁移**，本 skill 已移除。模型每 3–6 个月迭代一次，开工前先查 https://ark.volcengine.com/region:cn-beijing/docs/model-deprecation-notice ，遇到 `ModelNotFound`/模型下线类报错先换成公告里的建议迁移模型。

### 图片生成：火山方舟 Seedream 5.0 pro
- 模型 ID：`doubao-seedream-5-0-pro-260628`（备选：`doubao-seedream-5.0-flash`，更快更便宜）
- API：同步调用
- `size` 用分辨率档位：`1K` / `1.5K` / `2K`（默认 2K），在 prompt 里用自然语言写明宽高比（如 "16:9 widescreen"）；也可直接给像素值，两种写法不可混用
- 返回的图片 URL 有时效，拿到后立即下载到本地

```
POST https://ark.cn-beijing.volces.com/api/v3/images/generations
Headers: Authorization: Bearer $ARK_API_KEY
Body: {
  "model": "doubao-seedream-5-0-pro-260628",
  "prompt": "<英文提示词>, 16:9 widescreen",
  "size": "2K",
  "watermark": false
}
Response: data[0].url → 立即下载图片
```

### 视频生成：火山方舟 Seedance 2.0 系列（逐级降级策略）

**核心原则：优先使用最强模型，额度不足/限流时自动降级到次一级模型。**

| 优先级 | 模型 | Model ID | 分辨率 | 说明 |
|--------|------|----------|--------|------|
| 1（首选） | **Seedance 2.0** | `doubao-seedance-2-0-260128` | 480p/720p/1080p/4k | 画质最好，支持多图参考、首尾帧、视频延长 |
| 2 | **Seedance 2.0 Fast** | `doubao-seedance-2-0-fast-260128` | 480p/720p | 速度快、成本低 |
| 3 | **Seedance 2.0 Mini** | `doubao-seedance-2-0-mini-260615` | 480p/720p | 成本最低，官方指定的 1.0 Pro 替代 |
| 4（兜底） | **Seedance 1.5 Pro** | `doubao-seedance-1-5-pro-251215` | 720p/1080p | 旧一代，2.0 系列全部受限时再用 |

- Seedance 2.0 系列开通条件：账户余额 > 200 元（或购买对应套餐），否则调用会报未开通
- 单次可生成 **4–15 秒**（`duration`），比例用 `ratio`（16:9 / 9:16 等），分辨率用 `resolution`
- 我们自己做旁白和配乐，**必须传 `"generate_audio": false`**，否则模型会自带音轨，和旁白打架
- API：异步调用（提交→轮询→下载）

```
# 文生视频（t2v）
POST https://ark.cn-beijing.volces.com/api/v3/contents/generations/tasks
Body: {
  "model": "doubao-seedance-2-0-260128",
  "content": [{"type": "text", "text": "<英文提示词>"}],
  "ratio": "16:9",
  "resolution": "1080p",
  "duration": <按镜头时长取 4-15 的整数>,
  "generate_audio": false,
  "watermark": false
}

# 图生视频（i2v，人物镜头）—— 定妆照作为参考图，保证人物一致
Body: {
  "model": "<按降级梯队选择>",
  "content": [
    {"type": "image_url", "image_url": {"url": "<公网URL 或 data:image/jpeg;base64,...>"}, "role": "reference_image"},
    {"type": "text", "text": "<英文动作/场景提示词>"}
  ],
  "ratio": "16:9", "resolution": "1080p", "duration": <4-15>, "generate_audio": false
}
# 需要严格以某张图为第一帧时，把 role 改为 "first_frame"（可再加一张 "last_frame"）

# 轮询状态（每20秒一次，最长15分钟）
GET https://ark.cn-beijing.volces.com/api/v3/contents/generations/tasks/<task_id>
Response: status (queued/running/succeeded/failed), content.video_url → 立即下载
```

### 人物形象一致性方案

**核心问题**：纯文生视频每个镜头独立生成，同一人物在不同镜头中外貌差异巨大。

**解决方案：定妆照 + 图生视频（i2v）**

1. **Step 2 中先生成人物定妆照**：为每个主要人物用 Seedream 生成一张标准定妆照（正面/半侧面，清晰面部特征，中性背景）
2. **固定人物外貌描述**：在脚本制作说明中定义人物外貌锚点（如 "a young Asian man, short black hair, thin face, deep-set eyes, pale skin"），所有镜头提示词复用此描述
3. **含人物的镜头使用 i2v**：传入定妆照（`role: reference_image`，Seedance 2.0 系列最多 9 张参考图，可同时放多个人物的定妆照）+ 动作/场景提示词
4. **无人物的镜头**（纯景、太空、物体特写）继续使用 t2v 模型

### TTS 旁白：edge-tts 分句情感控制
- 前置依赖：`pip install -U edge-tts`（7.x 版本）
- 默认男声：`zh-CN-YunxiNeural`（低沉克制）
- 可选女声：`zh-CN-XiaoxiaoNeural`（温柔细腻）
- 其他可选：`zh-CN-YunjianNeural`（浑厚有力）、`zh-CN-YunyangNeural`（沉稳新闻播报风）
- 开工前可用 `edge-tts --list-voices` 确认音色仍可用

**⚠️ 自定义 SSML 已不可用**：edge-tts 已移除自定义 SSML 支持（微软服务端只接受一个 `<voice>` 内一个 `<prosody>`），把多段 `<prosody>`/`<break>` 拼在一起的写法会被拒绝或被当成文字念出来。

**替代做法：按句切分 + 每句单独设参数 + ffmpeg 补停顿**，效果等价：
1. 每段旁白按句（或按情感段）切开
2. 每句用 `Communicate(text, voice, rate=..., pitch=..., volume=...)` 单独生成一个小 mp3
3. 句间停顿用 ffmpeg 生成对应时长的静音片段
4. 按顺序拼接成该镜头的 `narr<NN>.mp3`

参数对照（`rate`/`volume` 用百分比，`pitch` 用 Hz）：
| 情感 | rate | pitch | volume |
|------|------|-------|--------|
| 叙事段 | `-15%` | `-3Hz` | `+0%` |
| 递进段 | `-20%` | `-5Hz` | `-5%` |
| 关键句 / 点题句 | `-35%` | `-8Hz` | `-20%` |
| 转折句句首 | `-10%` | `+0Hz` | `+0%` |

停顿：普通句间 400ms，转折前 800ms，关键句前后 600–1000ms。

**在 Step 1 写分镜脚本时，旁白文本同时附上分句情感标注**（每句标 叙事/递进/关键/转折，句间标停顿毫秒数），Step 4 直接按标注生成。

### 视频组装：FFmpeg
- 要求：系统已安装 FFmpeg（`ffmpeg` 和 `ffprobe` 在 PATH 中）
- 目标格式：H.264 + AAC, 1920×1080, 24fps

### BGM：Incompetech 免费曲库
- 曲目列表：`https://incompetech.com/music/royalty-free/pieces.json`
- 下载地址：`https://incompetech.com/music/royalty-free/mp3-royaltyfree/<filename>`
- 授权：CC BY 4.0（需注明 "Kevin MacLeod (incompetech.com)"）

### API Key 配置
从环境变量 `ARK_API_KEY` 读取火山方舟 API Key：

```python
import os
API_KEY = os.environ["ARK_API_KEY"]
```

如果环境变量不存在，提示用户设置：
```
powershell -Command "[System.Environment]::SetEnvironmentVariable('ARK_API_KEY', '<key>', 'User')"
```

---

## 二、制作流程

### Step 1: 文本分析与分镜脚本

阅读原始文本，执行以下分析：

**1.1 文本分析**
- 提取核心主题、情感弧线、关键转折点
- 识别主要人物及其视觉特征
- 标注最具画面感的段落
- 确定整体视觉基调（色温、明暗、风格）

**1.2 分镜设计原则**
- 每个镜头 4-15 秒（Seedance 2.0 单次生成上限 15 秒），一般 6-10 秒，总镜头数 = 目标时长 / 平均镜头时长
- 开头镜头：建立氛围（通常暗场/特写→展开）
- 中段镜头：信息密度较高，节奏可稍快
- 结尾镜头：节奏放慢，留白/留黑，情感沉淀
- 最后一个镜头可为纯黑屏+文字（点题句）

**1.3 人物视觉锚点定义**
- 为每个主要人物定义一段固定的英文外貌描述（如 "a young Asian man, short black hair, thin face, deep-set eyes, pale skin, wearing a white gown"）
- 此描述在所有包含该人物的镜头提示词中原封不动复用
- 在制作说明中集中列出所有人物的外貌锚点

**1.4 为每个镜头设计**
- 【画面】：详细的视觉描述
- 【字幕】：凝练的短句（不是旁白的复述，是画龙点睛）
- 【旁白】：叙述性文本，从原文提炼或改写
- 【旁白情感标注】：逐句标注 叙事/递进/关键/转折，句间停顿用 `//<毫秒>` 标记（如 `//800`）
- 【音效/BGM】：氛围音效描述
- **AI 生成提示词**：英文，用于 Seedream/Seedance，包含场景、光线、色调、构图、风格关键词
- **镜头类型标注**：标明 `[t2v]`（纯场景/无人物）或 `[i2v]`（含人物，需传入定妆照）

**1.4 脚本格式**

按以下标准格式输出脚本，保存到 `短视频脚本/<标题>_短片脚本.txt`：

```
<标题> —— 短片脚本
时长：约<N>秒
风格：<风格描述>


============================
镜号 01 | 0:00 - 0:06
============================
【画面】
<画面描述>

【字幕】
<字幕文本>

【旁白】（<声音描述>）
<旁白文本>

【音效/BGM】
<音效描述>
```

在脚本末尾附上制作说明，包含：
1. 整体视觉基调（主色调、对比色调）
2. 每个镜头的英文 AI 生成提示词
3. 旁白风格说明
4. BGM 建议
5. 节奏控制说明

**确认点**：脚本完成后展示给用户确认，再进入后续步骤。如用户有修改意见，先修订脚本。

---

### Step 2: AI 图片生成

为需要参考图的镜头生成概念图。

**执行逻辑**：
1. 从脚本制作说明中提取每个镜头的英文 AI 提示词
2. **首先生成人物定妆照**：为每个主要人物单独生成一张标准定妆照（正面或半侧面，清晰面部，中性背景），保存为 `char_<name>.jpg`
3. 再批量生成各镜头的概念图（可并行）
4. 每次调用使用 `size: "2K"`，prompt 末尾写明 `16:9 widescreen`（竖版写 `9:16 vertical`）
5. 保存为 `短片素材/<项目名>/shot<NN>_<desc>.jpg`

**提示词优化规则**：
- 始终包含：风格关键词（cinematic, 4K, sci-fi 等）
- 始终包含：光线描述（warm golden light, cold blue tones 等）
- 始终包含：构图描述（close-up, aerial shot, first-person POV 等）
- 人物描述使用通用特征而非具体人名

**错误处理**：
- 400 错误先检查 size 写法（档位 `2K` 与像素值不可混用）
- 429/限流则等待重试
- 内容审核驳回则调整提示词后重试

---

### Step 3: AI 视频生成

为每个镜头直接生成接近脚本时长的视频片段（不再依赖慢放拉长）。

**执行逻辑**：
1. 提取每个镜头的英文视频提示词及类型标注（`[t2v]` 或 `[i2v]`）
2. `duration` = 脚本镜头时长四舍五入，限制在 4–15 秒之间；旁白较长的镜头按 旁白时长+1 秒取值
3. **[i2v] 镜头**（含人物）：传入定妆照（`role: reference_image`）+ 动作/场景提示词
4. **[t2v] 镜头**（纯场景）：纯文本提示词
5. 所有请求都带 `"generate_audio": false`、`"ratio": "16:9"`、`"resolution"`（2.0 用 `1080p`，Fast/Mini 用 `720p`）
6. 批量提交任务（每批 4-6 个），轮询等待
7. 成功则立即下载视频到 `短片素材/<项目名>/shot<NN>_<desc>.mp4`（结果链接有时效）
8. 保存 task_id 和所用模型到 `video_tasks.json` 以便断点续传

**模型降级策略**（核心逻辑）：
1. 从 `doubao-seedance-2-0-260128`（Seedance 2.0）开始
2. 遇到 `SetLimitExceeded`、额度不足、429 限流或模型未开通，降级到 `doubao-seedance-2-0-fast-260128`
3. 再受限，降级到 `doubao-seedance-2-0-mini-260615`
4. 仍受限，兜底 `doubao-seedance-1-5-pro-251215`
5. 全部受限时提示用户：检查账户余额（2.0 系列需 > 200 元）、到火山方舟控制台关闭"安心体验模式"或调高用量上限，或等待额度刷新
6. **一旦某个模型成功，后续镜头继续使用该模型**（同一批次内不反复尝试更高级模型）
7. 降级到 Fast/Mini 时同步把 `resolution` 改为 `720p`，Step 5 统一放大到 1080p

**视频提示词优化规则**：
- 强调动态元素（camera moving, slowly pushing, pulling back）
- 描述动作/变化过程而非静态场景
- 包含节奏描述（slow, gentle, dramatic）
- 不要在提示词里要求出现文字（字幕由我们后期烧入）

---

### Step 4: TTS 旁白生成

为每个有旁白的镜头生成语音。

**执行逻辑**：
1. 检查 edge-tts 是否安装（`import edge_tts`），未安装则 `pip install -U edge-tts`
2. 从脚本提取每个镜头的旁白文本及分句情感标注
3. 每句单独生成，句间插入静音，拼接为 `短片素材/<项目名>/audio/narr<NN>.mp3`
4. 用 ffprobe 检查每段旁白的时长，记录备用

```python
import asyncio, subprocess, edge_tts

STYLE = {  # 情感 → (rate, pitch, volume)
    "叙事": ("-15%", "-3Hz", "+0%"),
    "递进": ("-20%", "-5Hz", "-5%"),
    "关键": ("-35%", "-8Hz", "-20%"),
    "转折": ("-10%", "+0Hz", "+0%"),
}

async def tts_sentence(text, style, out, voice="zh-CN-YunxiNeural"):
    rate, pitch, volume = STYLE[style]
    await edge_tts.Communicate(text, voice, rate=rate, pitch=pitch, volume=volume).save(out)

def silence(ms, out):
    subprocess.run(["ffmpeg", "-y", "-f", "lavfi", "-i", "anullsrc=r=24000:cl=mono",
                    "-t", f"{ms/1000:.3f}", "-c:a", "libmp3lame", "-b:a", "48k", out], check=True)

# segments 示例：[("她来看过他。","叙事",600), ("她说了两个字：","递进",400), ("谢谢。","关键",0)]
async def build_narration(segments, out, workdir):
    parts = []
    for i, (text, style, pause_ms) in enumerate(segments):
        p = f"{workdir}/s{i:02d}.mp3"; await tts_sentence(text, style, p); parts.append(p)
        if pause_ms:
            s = f"{workdir}/p{i:02d}.mp3"; silence(pause_ms, s); parts.append(s)
    with open(f"{workdir}/list.txt", "w", encoding="utf-8") as f:
        f.writelines(f"file '{x}'\n" for x in parts)
    # 重新编码拼接，避免不同片段参数不一致
    subprocess.run(["ffmpeg", "-y", "-f", "concat", "-safe", "0", "-i", f"{workdir}/list.txt",
                    "-ar", "24000", "-ac", "1", "-c:a", "libmp3lame", "-b:a", "64k", out], check=True)
```

---

### Step 5: 视频处理与统一格式

将所有生成的视频片段处理为统一格式。

**目标规格**：1920×1080, 24fps, H.264, yuv420p

**每个片段的处理**：
1. 获取原始视频的分辨率和时长（ffprobe）
2. 计算目标时长 = max(脚本镜头时长, 旁白时长 + 0.5s)
3. 计算慢放系数 = 目标时长 / 原始时长。Step 3 已按镜头时长生成，系数通常接近 1；只在旁白比画面长时小幅慢放（不超过 1.3 倍，超过就重新生成更长的片段，避免画面发涩）

FFmpeg filter chain：
```
setpts=<慢放系数>*PTS,
scale=1920:1080:force_original_aspect_ratio=decrease,
pad=1920:1080:(ow-iw)/2:(oh-ih)/2:black,
fade=t=in:st=0:d=0.8,
fade=t=out:st=<目标时长-0.8>:d=0.8
```

**纯黑屏+文字镜头**（如结尾点题）：
```
ffmpeg -y -f lavfi -i "color=c=black:s=1920x1080:d=<时长>:r=24"
  -c:v libx264 -preset fast -crf 20 -pix_fmt yuv420p output.mp4
```

**Python subprocess 调用规范**：
- 使用列表参数（不使用 shell=True），避免编码问题
- 不在 print 中输出包含中文的文件路径
- 开头加 `sys.stdout.reconfigure(encoding='utf-8', errors='replace')`

```python
import subprocess
def run_ffmpeg(args, desc=""):
    if desc:
        print(f"  {desc}")
    result = subprocess.run(["ffmpeg"] + args, capture_output=True)
    if result.returncode != 0:
        stderr = result.stderr.decode("utf-8", errors="replace")
        for line in stderr.split("\n"):
            if any(k in line.lower() for k in ["error", "invalid", "failed"]):
                print(f"    ERR: {line.strip()}")
        return False
    return True
```

---

### Step 6: 字幕与拼接

**6.1 生成 SRT 字幕文件**

按时间线生成 SRT，每个镜头一条字幕：
- 开始时间 = 镜头起始 + 0.3s（略迟于画面出现）
- 结束时间 = 镜头结束 - 0.5s（略早于画面消失）

```
1
00:00:00,300 --> 00:00:05,500
字幕文本

2
00:00:07,300 --> 00:00:13,500
字幕文本
...
```

**6.2 拼接所有片段**

使用 FFmpeg concat demuxer：
1. 生成 concat.txt 列表文件
2. `ffmpeg -f concat -safe 0 -i concat.txt -c copy output.mp4`

**6.3 烧入字幕**

使用 subtitles filter。因 Windows 路径含中文/空格会导致 FFmpeg 解析失败，采用以下策略：
1. 将 SRT 文件和视频文件 copy 到用户 home 目录下的简单路径（如 `c:/Users/<username>/`）
2. 在简单路径下执行 FFmpeg subtitles 烧入
3. 将结果 copy 回项目目录
4. 清理临时文件

字幕样式：
```
FontName=Microsoft YaHei,FontSize=22,PrimaryColour=&H00FFFFFF,
OutlineColour=&H80000000,BorderStyle=3,Outline=1,Shadow=0,MarginV=40
```

---

### Step 7: BGM 与最终组装

**7.1 BGM 选择**

从 Incompetech 曲库自动匹配：
1. 获取曲目列表：`GET https://incompetech.com/music/royalty-free/pieces.json`
2. 根据脚本风格关键词匹配 feel/instruments 字段
3. 风格映射表：

| 短片风格 | 匹配关键词 (feel) | 偏好乐器 |
|---------|-------------------|---------|
| 冷峻科幻 | Dark, Mysterious, Somber | Piano, Cello, Strings |
| 温暖文艺 | Calm, Relaxed, Peaceful | Piano, Guitar, Strings |
| 悬疑惊悚 | Dark, Eerie, Unnerving, Tense | Strings, Synth |
| 史诗壮阔 | Epic, Heroic, Intense | Orchestra, Brass, Percussion |
| 忧郁沉静 | Somber, Mystical | Piano, Cello, Choir |

4. 下载匹配度最高的曲目：
   `GET https://incompetech.com/music/royalty-free/mp3-royaltyfree/<filename>`

**7.2 音频混合**

将旁白音频拼接为完整音轨：
1. 每段旁白前加 0.3s 静音（adelay）
2. 用 apad 填充到镜头时长
3. 纯黑屏镜头（无旁白）用静音填充
4. 用 concat 拼接为完整旁白音轨

BGM 处理：
- 音量：15%（`volume=0.15`）
- 开头渐入：3s（`afade=t=in:st=0:d=3`）
- 结尾渐出：5s（`afade=t=out:st=<总时长-5>:d=5`）
- 裁剪到视频时长（`atrim=0:<总时长>`）

最终混合：
```
[narration][bgm]amix=inputs=2:duration=first:dropout_transition=3
```

**7.3 最终输出**

同样使用简单路径策略避免编码问题：
1. copy 素材到简单路径
2. 执行最终合成（视频 + 字幕 + 旁白 + BGM）
3. copy 结果回项目目录，命名为 `<标题>_完整版.mp4`
4. 清理临时文件

```
ffmpeg -y
  -i <concat_video>
  -i <narration_full>
  -i <bgm>
  -filter_complex "[2:a]volume=0.15,afade=...,atrim=...[bgm];[1:a][bgm]amix=...[aout]"
  -vf "subtitles=<srt_path>:force_style='<样式>'"
  -map 0:v -map "[aout]"
  -c:v libx264 -preset fast -crf 20
  -c:a aac -b:a 192k
  -shortest
  <output>
```

---

## 三、错误处理与降级策略

| 错误场景 | 处理方式 |
|---------|---------|
| Seedream API 返回 400（尺寸参数） | 改用档位 `size: "2K"`，不要与像素值混用 |
| 模型不存在 / 已下线 | 查火山方舟《模型下线公告》，换成建议迁移模型 |
| Seedance 2.0 未开通 | 提示用户账户余额需 > 200 元或购买套餐；先降级到 1.5 Pro 继续 |
| Seedance SetLimitExceeded / 429 | 按降级梯队切换：2.0 → 2.0 Fast → 2.0 Mini → 1.5 Pro，全部受限则提示用户 |
| Seedance 任务超时（>15min） | 重试一次，仍失败则跳过该镜头（用图片静帧替代） |
| 生成视频自带音轨 | 检查是否漏传 `generate_audio: false`；组装时只取视频流 `-map 0:v` |
| edge-tts 未安装 | 自动 `pip install -U edge-tts` |
| edge-tts 报 SSML 相关错误 | 说明还在用旧的 SSML 写法，改为 Step 4 的分句生成 |
| FFmpeg subtitles filter 路径错误 | 使用简单路径临时文件策略 |
| FFmpeg 编码错误（cp1252） | subprocess 使用列表参数，不使用 shell=True |
| BGM 下载失败 | 输出无 BGM 版本，提供 SRT 字幕文件供用户自行配乐 |
| 某个镜头视频生成失败 | 用对应图片生成 Ken Burns 效果视频替代 |

---

## 四、风格指南

### 视觉
- 根据原文情感自动推断色调方案
- 默认偏好：冷色主体 + 局部暖色对比（适合大多数叙事类短片）
- AI 提示词始终包含：`cinematic, 4K`，以及具体的光线和色调描述

### 旁白
- 不煽情，不抒情，像在陈述事实——但事实本身就是最重的情感
- 每句之间有明显停顿
- 关键句语速更慢、音量略降（通过分句生成、每句单独设参数实现）

### 字幕
- 简洁有力的短句
- 不是旁白的逐字重复，而是独立的视觉文本层
- 点题句可独占镜头（黑屏+居中文字）

### BGM
- 永远作为背景，不喧宾夺主
- 音量 15%（与旁白混合后旁白清晰可辨）
- 前半段偏冷/暗，后半段可适当变暖
- 结尾一个音符渐弱至无声

---

## 五、执行注意事项

1. **每步确认**：脚本完成后先给用户看，确认后再开始生成素材。素材全部生成后告知用户再开始组装。
2. **断点续传**：保存 `video_tasks.json` 记录任务 ID，如果中途失败可以从上次状态继续。
3. **并行效率**：图片生成（同步）可串行执行；视频生成（异步）可批量提交后统一轮询。
4. **资源管理**：中间文件存放在 `temp/` 目录，最终确认无误后可提示用户清理。
5. **授权声明**：如使用 Incompetech BGM，在输出时提醒用户标注：`"<Track>" Kevin MacLeod (incompetech.com) Licensed under Creative Commons: By Attribution 4.0 License`

---

## 六、自检清单

完成制作后，逐项检查：

- [ ] 脚本是否忠实于原文核心情感和主题？
- [ ] 每个镜头是否有清晰的画面描述和英文 AI 提示词？
- [ ] 所有图片/视频素材是否成功生成？失败的是否有降级方案？
- [ ] 视频片段是否都没有自带音轨（generate_audio=false）？
- [ ] 旁白时长是否与视频时长匹配（视频时长 ≥ 旁白时长 + 0.5s）？
- [ ] 所有视频片段是否统一为 1920×1080, 24fps？
- [ ] SRT 字幕时间线是否与画面正确对齐？
- [ ] BGM 音量是否合适（不盖过旁白）？
- [ ] 淡入淡出过渡是否流畅？
- [ ] 纯黑屏/文字镜头是否正确渲染？
- [ ] 最终 MP4 文件是否可正常播放，无损坏？
- [ ] 总时长是否接近目标时长？
- [ ] 是否告知用户 BGM 授权信息？

---

现在，请根据用户提供的文本，按照以上流程制作短片。每完成一个步骤，简要汇报进度，遇到问题及时告知用户。
