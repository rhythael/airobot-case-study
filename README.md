<h1 align="center">airobot — trilingual voice assistant for a vending machine</h1>
<p align="center">Case study · <a href="#english">English</a> · <a href="#中文">中文</a> · <a href="https://github.com/rhythael/airobot"><b>Source code (sanitised) →</b></a></p>
<p align="center">
  <img alt="role" src="https://img.shields.io/badge/role-sole%20developer-blue">
  <img alt="period" src="https://img.shields.io/badge/period-Sep%E2%80%93Oct%202026-lightgrey">
  <img alt="stack" src="https://img.shields.io/badge/stack-Python%20%7C%20sherpa--onnx%20%7C%20ESP32%20%7C%20Docker-3776AB">
  <img alt="first audio" src="https://img.shields.io/badge/first%20audio-0.4%E2%80%931.1%20s-brightgreen">
</p>

> This repository is a **case study only**: architecture, decisions and measured results of a project I built for SmartFoodie GmbH.
> 本仓库只是**案例说明**：我为 SmartFoodie GmbH 开发的项目的架构、决策与实测结果。

---

<a id="english"></a>
**Role**: sole developer (requirements, architecture, implementation, testing, deployment, handover)
**Period**: 2026-09-11 to 2026-10-15, about 5 weeks
**Attribution**: Heyao Rong (rhythael@gmail.com) · code lives in the company's private repository (tag `v1.0-handoff`, SSH-signed)
**Ownership**: built for SmartFoodie GmbH, which holds the usage rights; all project-specific code is authored by me (listed per module in the repository's CONTRIBUTIONS file), on top of the open-source xiaozhi-esp32-server (MIT).

## Problem
A customer at a vending machine asks, in Chinese, German or English, "what can I buy", "is there sesame in the dumplings", "I don't eat pork". The machine must answer from the **live inventory** within 1–2 seconds, and allergen answers must never be wrong. The prototype spoke one language, took 7–9 s before the first audio, let the LLM decide when to call the inventory tool, and read internal errors to customers.

## Solution
```
ESP32 → self-hosted server (VAD → SenseVoice/Whisper dual ASR → local answers → turn shaping → any OpenAI-compatible LLM → local first sentence + streamed cloud TTS) → inventory bridge → the company's inventory API
```
- **ASR routing**: SenseVoice decides Chinese first (0.05 s), Whisper only when unsure; treble compensation for the on-board microphone; slot-number mis-hearings normalised (mis-heard slot numbers → canonical form).
- **Per-turn inventory refresh**: instead of letting the model decide when to call a tool, every turn fetches the inventory from the bridge and injects it into the prompt — one model round-trip saved.
- **Local answer layer**: eight question classes (allergens, vegetarian / avoid-meat / spiciness, what's available, slots, prices, greeting, menu overview) are answered from structured inventory and 220 dish records with trilingual templates, never by the LLM. Allergen answers are therefore data-determined, with no room for hallucination.
- **Turn shaping layer**: reply-language detection and stickiness, language-switch requests, mapping of category words to shelf items across turns, "no invented popularity / spiciness" rules — decoupled from the concrete LLM, so swapping the model keeps the behaviour.
- **Speech**: first sentence synthesised locally with sherpa-onnx (audio after 0.1 s), the rest streamed from EdgeTTS and transcoded to Opus on the fly; voice per language, in-sentence language split.
- **Inventory bridge**: separate process, signed calls to the the company's inventory API, returns a text summary for the model plus a structured list for local answers; on failure returns an error code and the server says "inventory temporarily unavailable".
- **Handover**: credentials via .env, Linux/macOS scripts, Docker Compose and systemd, bilingual architecture / deployment / provenance docs, signed tag with hashed archive, handover receipt.

## Results
| Metric | Before | After |
|---|---|---|
| First audio, locally answered questions | 1.9–4.1 s (all via LLM) | **0.4–1.1 s** |
| First audio, LLM questions | 1.9–4.1 s | 1.4–2.5 s |
| Chinese ASR time | 0.59 s | 0.05 s |
| Trilingual WER (48 sentences) | 14.8 % | 9.5 % |
| Allergen answers | small LLM wrong 4/4 | local rules 100 % |
| Regression | — | 334 server, 29 bridge, 37 device-interface tests passing |

## Decisions I made
1. Safety-relevant answers (allergens, vegetarian) go to local rules, not the LLM — a small model got them wrong 4 out of 4 times even with complete data.
2. Turn shaping was extracted from the Claude CLI provider into a provider-neutral module — otherwise the company would lose all dialogue behaviour when switching models.
3. The device-address drift was solved by keeping a fixed address on the server rather than rewriting the device NVS — rewriting needs physical access and the upstream tool writes duplicate keys.
4. Device NVS images contain WiFi credentials; removed from the entire git history with filter-repo before handover and delivered separately.

## Pitfalls
macOS private Wi-Fi address rotation made the device lose the server; upstream's streaming extraction dropped the spaces between words (German read as one long word); a shallow-cloned history cannot be archived with git bundle; an OAuth token without the workflow scope cannot push tags that contain Actions files.

## Stack
Python 3.10/3.11, asyncio, websockets, sherpa-onnx (SenseVoice / Whisper / Piper / Matcha), Silero VAD, EdgeTTS, OpenAI-compatible LLM API, FastMCP, httpx, pytest, Docker, systemd, ESP32 (esptool / NVS). Codex and Claude Code were used as coding assistants; the architecture and every design decision are mine and were validated by measurement.


---

<a id="中文"></a>
**角色**：独立开发者（需求、架构、实现、测试、部署、交付全由本人完成）
**时间**：2026-09-11 至 2026-10-15，约 5 周
**署名**：Heyao Rong（rhythael@gmail.com）· 代码在公司私有仓库（标签 `v1.0-handoff`，SSH 签名）
**权属**：为 SmartFoodie GmbH 开发，使用权归公司；本项目的全部定制代码作者为本人（仓库内 CONTRIBUTIONS 文件逐模块列明），建立在开源项目 xiaozhi-esp32-server（MIT）之上。

## 问题
顾客站在自动售货机前，用中文、德语或英语问"有什么吃的""饺子里有芝麻吗""我不吃猪肉"，机器要结合**实时库存**在 1–2 秒内开口回答，并且过敏原这类答案不能出错。原型只能用一种语言、首包要等 7–9 秒、库存靠大模型猜着调工具，还会把错误信息念给顾客。

## 方案
```
ESP32 → 自建服务端（VAD → SenseVoice/Whisper 双模型识别 → 本地直答 → 每轮整形 → 任一 OpenAI 兼容 LLM → 首句本机合成 + 云端流式 TTS）→ 库存桥 → 公司库存接口
```
- **识别路由**：SenseVoice 先判中文（0.05 s），拿不准再用 Whisper；麦克风高频补偿；货道号听错写法归一（"一号期火道"→"1-7"）。
- **每轮库存刷新**：取消"模型决定何时调工具"，每轮入口直接向桥取库存并注入提示词，省掉一次模型往返。
- **本地直答层**：过敏原、素食/忌口/辣度、货单、货道、价格、招呼、菜单概括等 8 类问题不经过大模型，直接查结构化库存和 220 条菜品资料，三语模板回答。过敏原类答案由资料决定，杜绝模型幻觉。
- **每轮整形层**：回答语言判定与粘性、换语言请求识别、多轮指代把类别词映射到货架商品、"不编造受欢迎/辣"等规则，与具体大模型解耦，换模型不丢行为。
- **语音合成**：首句本机 sherpa-onnx 合成（0.1 s 出声），其余 EdgeTTS 边收边转 Opus；按语言换音色，句内分语言。
- **库存桥**：独立进程，签名调用公司库存接口，输出给模型的文本摘要 + 给本地直答的结构化清单；失败时返回错误码，服务端改说"暂时查不到库存"。
- **交付**：凭据走 .env、脚本 Linux/macOS 通用、Docker Compose 与 systemd、中英双语架构/部署/来源文档、签名标签与哈希归档、交接签收单。

## 结果
| 指标 | 之前 | 之后 |
|---|---|---|
| 首包出声（本地直答类问题） | 1.9–4.1 s（全走 LLM） | **0.4–1.1 s** |
| 首包出声（走 LLM 的问题） | 1.9–4.1 s | 1.4–2.5 s |
| 中文识别耗时 | 0.59 s | 0.05 s |
| 三语识别字错率（48 句实测） | 14.8 % | 9.5 % |
| 过敏原问答 | 小模型 4/4 答错 | 本地规则 100 % 正确 |
| 回归测试 | — | 服务端 334、桥 29、设备接口 37 条通过 |

## 我拍板的几个决定
1. 安全相关答案（过敏原、素食）不交给大模型，改本地规则——因为资料完整时小模型仍 4 次全错。
2. 整形逻辑从 Claude CLI provider 抽成独立模块——否则公司换模型时所有对话行为丢失。
3. 设备地址漂移问题用"让服务器常驻固定地址"而不是改设备 NVS——改设备要拆机刷写，且上游工具会写出重复键。
4. 设备 NVS 镜像含 WiFi 凭据，交付前用 filter-repo 从全部历史移除，单独交付。

## 踩过的坑
Mac 私有 Wi-Fi 地址轮换导致设备找不到服务端；上游 direct_answer 流式提取吃掉词间空格（德语念成一整串）；浅克隆历史不能用 git bundle 归档；OAuth token 缺 workflow scope 推不了含 Actions 的标签。

## 工具
Python 3.10/3.11、asyncio、websockets、sherpa-onnx（SenseVoice / Whisper / Piper / Matcha）、Silero VAD、EdgeTTS、OpenAI 兼容大模型 API、FastMCP、httpx、pytest、Docker、systemd、ESP32（esptool / NVS）。开发过程使用 Codex 与 Claude Code 作为编码辅助，架构与所有设计决策由本人做出并实测验证。
