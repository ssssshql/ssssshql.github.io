---
title: 自己账号秒发 TTS 语音条：NapCatQQ + fish.audio 整蛊小工具
date: 2026-09-09 14:19:00
tags: [NapCat, OneBot, TTS, fish.audio, QQ]
categories: 折腾
description: 用 NapCatQQ 登录自己的 QQ 号，脚本调 fish.audio 的 TTS 合成语音，再通过 OneBot send_group_msg 接口以自己账号的身份直接发语音条，聊天中随时整蛊
---

之前搭的东西里有个 [NapCatQQ](https://github.com/NapNeko/NapCatQQ)，研究后发现一个有意思的点：**NapCat 登录的就是我自己的 QQ 号，它把 QQ 客户端变成了一个标准协议网关**——也就是说，我可以正常聊天，然后突然用脚本让自己的号发一条合成的语音条出去。最近刚好又在写 [fish.audio](https://fish.audio/) 的反代项目，TTS 合成质量不错还能白嫖，两个一结合就做了这个小工具：**聊天中随时把任意一句话语音化发出去，整蛊群友神器**。

## 先说结论

一句话版：**NapCatQQ 跑在 QQNT 里、用你自己的 QQ 号登录，对外暴露 OneBot 11 协议；脚本调 fish.audio 的 TTS 接口合成 mp3，再调 OneBot 的 `send_group_msg` 接口带上 `record` 消息段发出去——以你本人身份发出语音条。**

## NapCatQQ 是什么

QQ 机器人圈子的老方案是 Go-CQHttp（基于老版本 QQ），老客户端下线后基本废了。NapCat 是社区目前的主流替代：

- 运行在**新版 QQ（QQNT）**框架内，相当于一个"无头"的 QQ 客户端，**登录你自己的 QQ 号**
- 对外暴露 **OneBot 11 协议**（HTTP / WebSocket / HTTP SSE 都支持），自己给自己发消息、发语音都能走这套 API
- 兼容 Go-CQHttp 时代的生态，AstrBot、NoneBot2、Koishi 这些框架都能直接接（也可以完全不接框架，纯脚本调用）

我这边是 docker 跑的（顺带还起了个 AstrBot）：

```yaml
napcat:
  image: mlikiowa/napcat-docker:latest
  environment:
    - MODE=astrbot
  volumes:
    - ./data:/AstrBot/data
    - ./napcat/config:/app/napcat/config
    - ./ntqq:/app/.config/QQ
  ports:
    - 6099:6099
astrbot:
  image: soulter/astrbot:latest
  ports:
    - "6185:6185"
  volumes:
    - ./data:/AstrBot/data
```

> 关键点：**QQ 客户端只是"壳"，消息收发全走 OneBot 协议**。NapCat 默认暴露 `6099` 端口，WebUI 里可以开启 HTTP API server，设一个 `access_token`——这就是我们发语音要用的接口。

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

## 整蛊工具：脚本秒发语音条

思路：**脚本收到文字 → fish TTS 合成 mp3 → 调 NapCat 的 OneBot HTTP 接口把 mp3 以语音条发出去。**

先准备 NapCat：WebUI（`http://127.0.0.1:6099/webui`）里开启 HTTP API server，记下端口和 `access_token`（没有则不填）。

我写了个 Python 脚本 `tts_send.py`，一条命令搞定：

```python
import httpx, json, sys, os

group_id = int(os.getenv("QQ_GROUP", "123456789"))      # 目标群号
napcat_url = os.getenv("NAPCAT_URL", "http://127.0.0.1:6099")
napcat_token = os.getenv("NAPCAT_TOKEN", "")
fish_key = os.getenv("FISH_API_KEY", "sk-fish-...")

def synth(text: str) -> str:
    # 手动 ensure_ascii=True 转义，规避官方 API 中文 invalid unicode 的坑
    payload = json.dumps({"text": text, "format": "mp3"}, ensure_ascii=True)
    r = httpx.post("https://api.fish.audio/v1/tts",
                   headers={"Authorization": f"Bearer {fish_key}",
                            "Content-Type": "application/json",
                            "model": "s2.1-pro-free"},
                   content=payload)
    r.raise_for_status()
    path = os.path.abspath("tts.mp3")
    open(path, "wb").write(r.content)
    return path

def send(path: str):
    message = [{"type": "record", "data": {"file": path}}]
    headers = {"Authorization": f"Bearer {napcat_token}"} if napcat_token else {}
    r = httpx.post(f"{napcat_url}/send_group_msg",
                   headers=headers,
                   json={"group_id": group_id, "message": message})
    r.raise_for_status()

if __name__ == "__main__":
    text = " ".join(sys.argv[1:])
    print(f"合成: {text}")
    send(synth(text))
    print("已发出")
```

用法（发送前先用环境变量把群号、token 配好）：

```powershell
$env:QQ_GROUP = "123456789"
$env:NAPCAT_TOKEN = "你的token"
$env:FISH_API_KEY = "sk-fish-..."
python tts_send.py 群主其实是只猫
python tts_send.py 今天谁请客啊
```

脚本瞬间合成并发出——你在群里正常聊天，下一秒就冒出一条语音条。想更顺滑，可以把这行命令绑定成快捷方式/输入法短语，点一下就发。

## 验证与注意

- 跑 `python tts_send.py 大家好`，能看到群里以你自己账号发出语音条 → **成功**。QQ 语音条底层是 silk 编码，**NapCat 负责转码，直接给它本地 mp3 路径就行**，这是比老框架省事的地方。
- 第一次用先发一条测试，确认 NapCat HTTP 接口通：`curl -X POST http://127.0.0.1:6099/get_login_info` 看能不能返回你的 QQ 号信息。
- **fish key / napcat token 用环境变量，别写死在脚本里、别发到网上。**
- 免费模型 `s2.1-pro-free` 没有 SLA、可能排队，整蛊够用；想要更稳就充值 API 额度用 `s2.1-pro`。
- 群号可以做成参数：`python tts_send.py --to 群号 文字`，好友私聊用 `send_private_msg` 同理。

## 总结

- NapCatQQ 用你自己的 QQ 号登录，把客户端变成 OneBot 11 网关——**自己给自己发消息完全合法**。
- 发语音 = OneBot `send_group_msg` 带 `record` 消息段，mp3 直接给，silk 转码 NapCat 干。
- fish.audio 有官方免费模型 `s2.1-pro-free`，一条 curl 就能合成语音；网页内部接口则是"抓包逆向 → reCAPTCHA 程序化 → 批量合成"的另一条路。
- 整个工具就一个脚本、`python tts_send.py 一句话` 完事，聊天中随时整蛊。

## 参考链接

- [NapCatQQ](https://github.com/NapNeko/NapCatQQ)
- [fish.audio 官方 TTS 文档](https://docs.fish.audio/features/text-to-speech)
- [OneBot 11 协议](https://github.com/botuniverse/onebot-11)