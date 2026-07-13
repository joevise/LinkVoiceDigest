# LinkVoiceDigest 🔗🎙️

链接消化 Skill — 给大Joe 用的"听文章"学习闭环工具。

## 工作流程

收到链接 → 提取正文 → 结构化总结 → 闭环逻辑拆解 → MiniMax TTS 高品质语音 → 飞书推送

## 核心能力

1. **内容提取**：支持公众号、网页、文章（tavily_extract + web_fetch）
2. **结构化总结**：核心观点 + 内容摘要 + 闭环逻辑 + 学习要点 + 独立观点
3. **闭环逻辑拆解**：借鉴 closed-loop-logic-extraction 方法论，精简应用
4. **MiniMax TTS**：speech-02-hd 模型，中文高品质语音合成（3-5分钟播报）
5. **飞书推送**：语音消息 + 文字总结，通过 lark-cli 发送

## 使用方式

在飞书里直接给 William 发一个链接，说"帮我读一下"或直接发 URL 即可触发。

## 技术栈

- MiniMax T2A API (speech-02-hd) — 中文 TTS
- ffmpeg-static — MP3 → Opus 转换
- lark-cli — 飞书消息（William profile）
- tavily_extract — 网页内容提取

## 安装

将 `SKILL.md` 放入 OpenClaw workspace 的 `skills/link-digest/` 目录即可。

---
Created by William 🎩 | 2026-07-13
