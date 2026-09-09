---
title: astrbot + NapCatQQ + fish.audio：给 QQ 机器人做语音整蛊工具
date: 2026-09-09 14:19:00
tags: [AstrBot, NapCat, OneBot, TTS, fish.audio, QQ机器人]
categories: 折腾
description: 从 NapCatQQ 的 OneBot 原理，到用 fish.audio 的 TTS API 给 AstrBot 机器人加上"文字转语音"能力，实现一个在群里发语音条整蛊群友的小工具
---

之前搭了一个 AstrBot 机器人，研究过程中发现 QQ 消息实际上走的是 [NapCatQQ](https://github.com/NapNeko/NapCatQQ)，顺手把这个项目扒了一遍。最近又刚好在写 [fish.audio](https://fish.audio/) 的反代项目，发现它家的 TTS 合成质量不错、还白嫖起来很方便。两个东西一结合，就做了一个小工具：**在群里让机器人把任意一句话用语音条发出来**，整蛊群友神器。

## 先说结论

一句话版：**NapCatQQ 是跑在 QQNT 里的 OneBot 11 实现，AstrBot 通过 WebSocket 连上它；让机器人发语音，等于往 AstrBot 里塞一个 `Record` 消息段；语音内容用 fish.audio 的 TTS API 现合成。**

## NapCatQQ 是什么

QQ 机器人圈子的老方案是 Go-CQHttp（基于老版本 QQ），但老 QQ 客户端下线后基本废了。NapCat 是社区目前的主流替代：

- 运行在**新版 QQ（QQNT）**框架内，相当于一个"无头"的 QQ 客户端，登录你自己的 QQ 号当机器人
- 对外暴露 **OneBot 11 协议**（HTTP / WebSocket / HTTP SSE 都支持）
- 兼容 Go-CQHttp 时代的生态，NoneBot2、Koishi、AstrBot 这些框架都能直接接

我的 docker-compose 里是这么配的（`MODE=astrbot`）：

```yaml
napcat:
  image: mlikiowa/napcat-docker:latest
  environment:
    - MODE=astrbot
  volumes:
    - ./data:/AstrBot/data
    - ./napcat/config:/app/napcat/config
    - ./ntqq:/app/.config/QQ
astrbot:
  image: soulter/astrbot:latest
  ports:
    - "6185:6185"
  volumes:
    - ./data:/AstrBot/data
```

NapCat 那边把 OneBot 的 WebSocket 客户端连到 AstrBot，见 `napcat/config/onebot11.json`：

```json
{
  "network": {
    "websocketClients": [
      {
        "enable": true,
        "name": "rws",
        "url": "ws://astrbot:6199/ws",
        "messagePostFormat": "array"
      }
    ]
  }
}
```

> 这一步搞明白后其实挺妙的：**QQ 客户端本身只是"壳"，NapCat 把消息转成标准协议，AstrBot 只认协议不认 QQ**——换任何 OneBot 网关（如 Lagrange、LLOneBot）都不用改上层代码。

## fish.audio 的 TTS：正式 API vs 网页内部接口

fish.audio 有两套完全独立的 TTS 接口，计费和鉴权互不相通：

| | 官方开发者 API | 网页内部接口 |
|---|---|---|
| 端点 | `POST /v1/tts` | `POST /task` |
| 鉴权 | API Key（`sk-fish-...`） | Bearer 账号 token + reCAPTCHA |
| 免费模型 | **`s2.1-pro-free`（白嫖）** | 扣平台积分 |
| reCAPTCHA | 不需要 | 需要（可程序化获取） |
| 返回 | 直接音频流 | 任务 id，轮询拿 mp3 直链 |

### 方式一：官方 API（推荐，直接就能用）

```bash
curl -X POST https://api.fish.audio/v1/tts \
  -H "Authorization: Bearer $FISH_API_KEY" \
  -H "Content-Type: application/json" \
  -H "model: s2.1-pro-free" \
  --data-binary @body.json -o out.mp3
```

> **坑**：`body.json` 里如果直接放汉字，可能 400 `invalid unicode`。要么在 JSON 里用 `\uXXXX` 转义，要么 Windows 下用 `curl --data-binary @file.json`（UTF-8 文件）而不是 `-d`。实测英文字符直接可用，中文转义后可用。

### 方式二：网页内部接口（重构/绕过付费用）

前端实际调的是这套，通过 CDP 抓包还原的：

1. `POST https://api.fish.audio/task`，请求体：

```json
{
  "type": "tts",
  "stream": true,
  "model": "<说话人id>",
  "latency": "balanced",
  "parameters": {
    "text": "要合成的文字",
    "model_id": "<说话人id>",
    "format": "mp3",
    "backend": "s2.1-pro",
    "normalize": false
  },
  "recaptcha": "<token>"
}
```

2. 轮询 `GET https://api.fish.audio/task/{taskId}` 直到 `state == "finished"`，`result` 字段就是 mp3 直链。
3. `recaptcha` 是个门槛，但实测可以在登录态页面用一行 JS 程序化获取：

```js
const rc = await window.grecaptcha.enterprise.execute(
  '6LfR4RwqAAAAAEvptRw9zohw7HeDU6NCqtAnJk1i',
  { action: 'tts' }
);
```

**关键实测**：同一个 token 在有效窗口内可**批量**用于多条 `/task` 调用，每次都能拿到正常 mp3；说话人（音色）id 从 `/model/batch/lookup` 查。这给"解一次、批量合成"留了空间。

## 整蛊工具：让机器人发语音条

思路很简单：群里发指令 → 机器人取文本 → 调 fish TTS 合成 mp3 → 用 `Record` 消息段发出去。

给 AstrBot 写个插件，核心代码就这些：

```python
from astrbot.api.event import filter, AstrMessageEvent
from astrbot.api.star import Context, Star, register
from astrbot.api.message_components import Record
import httpx, tempfile, json

FISH_API_KEY = "sk-fish-..."   # 开发者控制台创建
MODEL_ID = "s2.1-pro-free"

@register("fish_tts", "me", "用 fish.audio 合成语音发到群", "1.0.0")
class FishTTS(Star):
    def __init__(self, context: Context):
        super().__init__(context)
        self.headers = {"Authorization": f"Bearer {FISH_API_KEY}",
                        "Content-Type": "application/json",
                        "model": MODEL_ID}

    @filter.command("唱")
    async def sing(self, event: AstrMessageEvent, text: str):
        # 注意：json.dumps 默认 ensure_ascii=True，正好规避中文 invalid unicode 的坑
        payload = json.dumps({"text": text, "format": "mp3"})
        async with httpx.AsyncClient() as client:
            resp = await client.post("https://api.fish.audio/v1/tts",
                                     headers=self.headers, content=payload)
            mp3 = resp.content
        with tempfile.NamedTemporaryFile(suffix=".mp3", delete=False) as f:
            f.write(mp3)
            path = f.name
        yield event.chain_result([Record.fromFileSystem(path)])
```

加进 `data/plugins/astrbot_plugin_fish_tts/`（放 `main.py` + 一份 `metadata.yaml`），重新加载插件就能用。

群里玩法：

```
/唱 群主其实是只猫
/唱 今天谁请客啊
```

机器人会先合成 mp3，然后像正常语音条一样发到群里。

## 验证与注意

- 在群里发 `/唱 大家好`，能收到机器人发的语音条 → **成功**。QQ 语音条需要 silk 编码，NapCat 内置转码，直接给 mp3 就行——这也是它比老框架省事的地方。
- AstrBot 侧发送语音就是 `Comp.Record.fromFileSystem()` / `fromBase64()`，对应 OneBot 里 `record` 消息段。
- **token 或 key 别写死在代码里**，插件里用环境变量/配置文件读取。
- 免费模型 `s2.1-pro-free` 没有 SLA，且官方说免费模型可能排队，整蛊够用；想要更稳就充值 API 额度用 `s2.1-pro`。

## 总结

- NapCatQQ 把 QQ 客户端变成 OneBot 11 网关，上层框架只认协议不认 QQ。
- AstrBot 发语音 = 塞一个 `Record` 消息段，语音内容随意。
- fish.audio 有官方免费模型 `s2.1-pro-free`，一条 curl 就能合成语音；网页内部接口则是"抓包逆向 → reCAPTCHA 程序化 → 批量合成"的另一条路。
- 三者一组合，QQ 语音整蛊工具本体不到 50 行。

## 参考链接

- [NapCatQQ](https://github.com/NapNeko/NapCatQQ)
- [AstrBot](https://github.com/Soulter/AstrBot)
- [fish.audio 官方 TTS 文档](https://docs.fish.audio/features/text-to-speech)
- [OneBot 11 协议](https://github.com/botuniverse/onebot-11)