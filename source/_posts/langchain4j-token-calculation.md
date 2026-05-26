---
title: LangChain4j Token 计算踩坑：上下文窗口与多次 HTTP 请求
date: 2026-05-26 16:33:55
tags: [LangChain4j, Token, AI, 故障排查]
---

做项目时用 LangChain4j 计算上下文窗口消耗的 token 量，结果发现数字忽高忽低——第一次对话消耗 4k，第二次对话消耗 2k，根本对不上。

## 现象

按官方文档的写法，在 `tokenStream.onCompleteResponse` 回调里拿 `TokenUsage` 来统计 token：

```java
tokenStream.onCompleteResponse((ChatResponse response) -> {
    TokenUsage usage = response.tokenUsage();
    log.info("input: {}, output: {}, total: {}",
        usage.inputTokenCount(),
        usage.outputTokenCount(),
        usage.totalTokenCount());
});
```

逻辑上没问题，用来算钱也是对的。但把它当作**当前上下文窗口占用量**来用，就出问题了——token 量在对话间忽高忽低，完全没有规律。

## 原因

**真正踩坑点：Agent 一次对话可能触发了多次 HTTP 请求，`onCompleteResponse` 拿到的是所有 HTTP 请求的 token 总和，不是单次请求的 token 量。**

用 LangChain4j 的 Agent（AiService + Tools）时，模型决定调用工具、拿到工具返回后再次请求模型，这一套流程对用户来说是"一次对话"，但实际底层发了 2～N 次 HTTP 请求：

```
用户消息 → HTTP-1 (决定调工具) → 工具执行 → HTTP-2 (整合工具结果回复用户)
```

`onCompleteResponse` 回调里的 `tokenUsage` 是 HTTP-1 + HTTP-2 的**总和**，所以：

- 计费没问题，总消耗确实是这么多。
- **用来追踪上下文窗口就会错**——因为 HTTP-2 才是真正"看到全部历史"的那次请求，HTTP-1 只看了部分上下文。

这就解释了为什么 token 数量忽高忽低：Agent 调工具的次数每次不一样，叠加的 HTTP 请求次数就不同。

## 解决方案

查看 [LangChain4j Observability 文档](https://docs.langchain4j.dev/tutorials/observability)，发现框架提供了 `AiServiceResponseReceivedEvent`，可以拿到每次 HTTP 请求的独立 token 用量。

```java
@Component
public class TokenUsageListener {

    @EventListener
    public void onResponse(AiServiceResponseReceivedEvent event) {
        TokenUsage usage = event.response().tokenUsage();
        if (usage == null) return;
        log.info("单次 HTTP | input: {}, output: {}, total: {}",
            usage.inputTokenCount(),
            usage.outputTokenCount(),
            usage.totalTokenCount());
    }
}
```

**只需要取最后一次 HTTP 请求的 `inputTokenCount`，就是当前上下文窗口的实际占用量。**

> **为什么是最后一次？** 因为每次 HTTP 请求都会把之前所有消息历史（包括工具调用的中间结果）一起发给模型，最后一次请求的 input token 就代表了当前完整上下文的大小。

## 补充说明

- `onCompleteResponse` 里的总 token 量，用于**计费统计**没问题，实际总消耗就是这个数。
- 追踪上下文窗口时，务必用 `AiServiceResponseReceivedEvent` 取最后一次 HTTP 的数据，两者不能混用。
- Spring Boot 项目直接用 `@EventListener` 监听即可，非 Spring 项目需要手动注册 `ChatModelListener`。
- 如果只想监听特定 AiService 的事件，可以检查 `event.service()` 的类型来做区分。

## 参考链接

- [LangChain4j Observability 文档](https://docs.langchain4j.dev/tutorials/observability)
