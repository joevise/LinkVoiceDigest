---
name: link-digest
description: 收到链接后提取正文、结构化总结、闭环逻辑拆解、MiniMax TTS 高品质语音播报（大Joe定制克隆声音，1.2倍速），通过飞书推送（语音+文字摘要同发），实现"听文章"学习闭环。✅ 已定型。
version: 3.0
created: 2026-07-13
updated: 2026-07-13
author: William (JoeVise 分身)
---

# LinkVoiceDigest — 链接消化 Skill

> 收到链接 → 提取正文 → 结构化总结 → 闭环逻辑拆解 → MiniMax TTS 语音播报 → 飞书推送（语音+文字摘要同发）

## 触发条件

大Joe 发来一个 URL（公众号、网页、文章等），或明确说"帮我读一下这个链接""消化一下这篇文章"。

**推荐唤起话术**（大Joe可以这样说）：
- "帮我消化一下这个链接：[URL]"
- "读一下这篇文章：[URL]"
- 或直接发链接 + "这个"

## 完整流程

### Step 1 — 内容提取

优先级：
1. `tavily_extract`（处理 JS 渲染页面、公众号效果好，用 extract_depth=advanced）
2. `web_fetch`（轻量提取，备用）

- 公众号链接（mp.weixin.qq.com）必须用 tavily_extract + advanced
- 如果提取失败，尝试备用方案
- 明确失败原因，不瞎编

### Step 2 — 结构化总结（文字版，必须与语音一起发送）

按以下格式输出：

```
🔑 核心观点
（一句话，不超过30字，说清这篇文章的"心脏"）

📝 内容摘要
（200-500字，把文章核心讲清楚。当书听、当新闻听的体量）

🔄 闭环逻辑
（借鉴 closed-loop-logic-extraction 方法论，精简应用：
 - 原始概念：关键概念是什么
 - 基本关系：概念怎么连接
 - 闭环条件：逻辑怎么跑通
 用最精简方式说清核心逻辑引擎）

📚 学习要点
（3-5条可带走的知识点）

💡 William的观点
（独立判断：同意/不同意/补充思考）
```

**重要**：每次推送必须同时包含 (1) 语音消息 + (2) 这份文字摘要，不能只发语音。大Joe明确要求"推出来的时候要带上大致的摘要总结"。

### Step 3 — 生成 MiniMax TTS 语音

**语音脚本要求**：
- 800-1500字（约2-4分钟，按1.3倍速）
- 口语化，像播客一样自然
- 包含：核心观点 → 内容摘要 → 闭环逻辑 → William的观点
- 不念格式符号，用自然过渡语
- 结尾："以上就是这篇文章的消化，我是William，下次见。"

**MiniMax T2A API 调用**（国内版，使用真实系统 voice_id）：

```python
import requests, os

API_KEY = os.environ["MINIMAX_API_KEY"]
# 注意：coding plan key 使用 api.minimax.chat（国内版）

payload = {
    "model": "speech-02-hd",
    "text": script_text,  # 不超过 10000 字符
    "stream": False,
    "language_boost": "Chinese",
    "voice_setting": {
        "voice_id": "moss_audio_9c223de9-7ce1-11f0-9b9f-463feaa3106a",  # 大Joe定版自定义克隆男声（最终确认）2026-07-13
        "speed": 1.2,  # 大Joe要求：1.2倍速
        "vol": 1.0,
        "pitch": 0
    },
    "audio_setting": {
        "sample_rate": 32000,
        "bitrate": 128000,
        "format": "mp3",
        "channel": 1
    }
}

resp = requests.post(
    "https://api.minimax.chat/v1/t2a_v2",
    headers={
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json"
    },
    json=payload,
    timeout=120
)

data = resp.json()
if data["base_resp"]["status_code"] == 0:
    audio_hex = data["data"]["audio"]
    audio_bytes = bytes.fromhex(audio_hex)
    with open("output.mp3", "wb") as f:
        f.write(audio_bytes)
```

**默认声音配置（已定版，2026-07-13）**：
- `voice_id`: `moss_audio_9c223de9-7ce1-11f0-9b9f-463feaa3106a`（大Joe定制自定义克隆男声，经多次试听确认）
- `speed`: `1.2`

**声音选型试听记录**（供以后参考，不再使用）：
- ~~`male-qn-qingse`~~ — 旧版ID格式已失效
- ~~`Chinese (Mandarin)_Reliable_Executive`~~ — 可靠高管男声，沉稳但非最终选择
- ~~`Chinese (Mandarin)_Mature_Woman`~~ — 大Joe试过后觉得不行
- ✅ `moss_audio_9c223de9-7ce1-11f0-9b9f-463feaa3106a` — **最终定版**（自定义克隆声音）
- 完整系统声音列表（备用）：https://platform.minimax.io/docs/faq/system-voice-id

### Step 4 — 转换 Opus + 飞书发送

**MP3 → Opus 转换**（飞书语音消息要求 opus 格式）：

```bash
FFMPEG="/home/elttilz/.npm-global/lib/node_modules/ffmpeg-static/ffmpeg"
"$FFMPEG" -i voice.mp3 -acodec libopus -ac 1 -ar 16000 voice.opus -y
```

**通过 lark-cli 发送**（William 身份）：

```bash
# 确保使用 william profile
lark-cli profile use william

# 1. 发送语音消息（bot 身份上传 + 发送）
cd /tmp/link-digest && lark-cli im +messages-send \
  --user-id "ou_526f66e2b9bc86ce57a75e71d98b5647" \
  --audio "./voice.opus" \
  --as bot

# 2. 发送文字总结（user 身份，必须同时发送！）
lark-cli im +messages-send \
  --user-id "ou_526f66e2b9bc86ce57a75e71d98b5647" \
  --markdown "文字总结内容" \
  --as user
```

**大Joe 对 William 的 open_id**: `ou_526f66e2b9bc86ce57a75e71d98b5647`

### Step 5 — 双重交付确认

每次任务完成必须同时交付：
1. ✅ 语音消息（MiniMax TTS，沉稳男声，1.3倍速）
2. ✅ 文字结构化摘要（Step 2 的完整格式）

## 闭环逻辑精简应用

借鉴 `closed-loop-logic-extraction` skill，但做精简：
- **不是**拆一个领域的完整闭环（那需要五步组合拳）
- **而是**拆这篇文章的核心论点逻辑链
- 精简三步：
  1. **原始概念**：文章里哪几个概念是不能再拆的底？
  2. **基本关系**：这些概念怎么连？谁导致谁？
  3. **闭环条件**：什么情况下这个逻辑跑通了？

## 注意事项

1. **MiniMax TTS** 使用国内版 API（api.minimax.chat），coding plan key，从环境变量 `MINIMAX_API_KEY` 读取，不要硬编码
2. **飞书语音必须转 opus** 格式，用 ffmpeg-static（npm 安装，路径见上）
3. **公众号**用 tavily_extract advanced 模式
4. **lark-cli** 用 william profile，语音消息用 bot 身份发，文字用 user 身份发
5. **必须双发**：语音 + 文字摘要，不能只发一种
6. 大Joe直接发链接即触发，不需要特定命令词

## 依赖

- `tavily_extract` / `web_fetch` — 内容提取
- `MiniMax T2A API` (speech-02-hd) — 高品质中文 TTS
- `ffmpeg-static` (npm) — MP3 → Opus 转换
- `lark-cli` (william profile) — 飞书消息发送
- `closed-loop-logic-extraction` (方法论借鉴) — 闭环逻辑拆解

## ✅ Skill 已定型（2026-07-13）

经大Joe多轮试听确认，此 Skill 正式定型，默认配置为：
- **声音**: `moss_audio_9c223de9-7ce1-11f0-9b9f-463feaa3106a`（大Joe定制克隆声）
- **语速**: `1.2`倍速
- **交付方式**: 语音+文字摘要同发

## 更新历史

- v3.0（2026-07-13）: ✅ **正式定型**，默认声音改为大Joe定制的自定义克隆声 `moss_audio_9c223de9-7ce1-11f0-9b9f-463feaa3106a`
- v2.2 (2026-07-13): 语速从1.3调整为1.2倍速（大Joe反馈1.3偏快）
- v2.1 (2026-07-13): 声音改为 `Chinese (Mandarin)_Reliable_Executive`（沉稳男声），语速改为1.3倍速；明确要求语音+文字摘要必须同时发送
- v2.0 (2026-07-13): 升级 TTS 从 Edge TTS 到 MiniMax speech-02-hd
- v1.0 (2026-07-13): 初始版本，使用 Edge TTS
