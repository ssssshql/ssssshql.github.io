---
title: LocalAI qwen3-embedding 模型返回相同结果的 Bug 记录
date: 2026-06-15 16:45:00
tags: [LocalAI, Docker, AI, 嵌入模型, Qwen3]
categories: AI
description: 记录 LocalAI Docker 镜像 latest-cpu 版本中 qwen3-embedding-0.6b 模型返回相同嵌入结果的 Bug，并提供解决方案
---

在使用 LocalAI 部署 Qwen3 嵌入模型时，遇到了一个奇怪的问题：无论输入什么文本，模型都返回完全相同的嵌入向量。经过排查，发现这是一个已知的 Bug，需要升级到特定版本才能解决。

## 问题现象

使用 LocalAI 的 `latest-cpu` Docker 镜像部署 `qwen3-embedding-0.6b` 模型时，发现：

- 无论输入什么文本，模型返回的嵌入向量完全相同
- 所有维度的值都是一样的
- 这个问题在 GitHub Issues 中有记录：[https://github.com/mudler/LocalAI/issues/6257](https://github.com/mudler/LocalAI/issues/6257)

## 原因分析

经过查看 GitHub Issues，发现这是一个已知的 Bug：

1. **特定版本问题**：`latest-cpu` 镜像版本在当时还未修复这个 Bug
2. **模型处理问题**：LocalAI 在处理 Qwen3 系列嵌入模型时存在缺陷，导致无法正确区分不同的输入
3. **上游修复**：这个 Bug 已经在后续版本中修复

## 解决方案

### 方案一：升级到修复版本（推荐）

使用 **v3.12.1-aio-cpu** 或更高版本的镜像：

```yaml
# docker-compose.yml
version: '3.8'

services:
  localai:
    image: localai/localai:v3.12.1-aio-cpu
    # 或者使用更新的版本
    # image: localai/localai:v3.13.0-aio-cpu
    container_name: localai
    ports:
      - "8080:8080"
    volumes:
      - ./models:/models
    environment:
      - DEBUG=true
      - THREADS=4
      - CONTEXT_SIZE=512
    restart: always
```

**推荐版本**：
- `v3.12.1-aio-cpu` - 修复了该 Bug 的稳定版本
- `v3.13.0-aio-cpu` - 更新的版本，包含更多改进
- 最新的稳定版本（修复了所有已知问题）

### 方案二：使用其他嵌入模型

如果暂时不想升级 LocalAI，可以考虑使用其他嵌入模型：

```yaml
models:
  - name: text-embedding
    backend: llama
    parameters:
      model: models/bge-base-zh-v1.5  # 使用 BGE 系列模型
      embedding: true
      type: embedding
```

**推荐的替代模型**：
- `bge-base-zh-v1.5`
- `bge-large-zh-v1.5`
- `nomic-embed-text`
- `all-MiniLM-L6-v2`

## 验证修复

升级后，使用以下代码验证嵌入结果是否正确：

```python
import requests
import json

def get_embedding(text):
    url = "http://localhost:8080/v1/embeddings"
    payload = {
        "model": "qwen3-embedding-0.6b",
        "input": text
    }
    response = requests.post(url, json=payload)
    return response.json()['data'][0]['embedding']

# 测试不同的输入
text1 = "今天天气真好"
text2 = "我喜欢编程"
text3 = "今天天气真好"

embedding1 = get_embedding(text1)
embedding2 = get_embedding(text2)
embedding3 = get_embedding(text3)

# 验证
print(f"文本1 和 文本2 的相似度: {sum(a*b for a,b in zip(embedding1, embedding2)):.4f}")
print(f"文本1 和 文本3 的相似度: {sum(a*b for a,b in zip(embedding1, embedding3)):.4f}")
print(f"文本2 和 文本3 的相似度: {sum(a*b for a,b in zip(embedding2, embedding3)):.4f}")

# 预期结果：
# - 文本1 和 文本2 的相似度应该较低（不同内容）
# - 文本1 和 文本3 的相似度应该很高（相同内容）
# - 文本2 和 文本3 的相似度应该较低（不同内容）
```

**正确的结果应该是**：
- 相同文本的相似度接近 1.0
- 不同文本的相似度明显较低

## 完整的 Docker Compose 配置示例

```yaml
version: '3.8'

services:
  localai:
    image: localai/localai:v3.12.1-aio-cpu
    container_name: localai-embedding
    ports:
      - "8080:8080"
    volumes:
      - ./models:/models
      - ./config:/config
    environment:
      - DEBUG=true
      - THREADS=4
      - CONTEXT_SIZE=512
      - CORS=true
      - CORS_ALLOW_ORIGINS=*
    restart: always
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3
```

模型配置文件 `config/qwen3-embedding.yaml`：

```yaml
name: qwen3-embedding-0.6b
backend: llama
parameters:
  model: models/qwen3-embedding-0.6b.gguf
  embedding: true
  type: embedding
context_size: 512
threads: 4
gpu_layers: 0
f16: false
```

## 总结

这个问题的根本原因是 LocalAI 的 `latest-cpu` 镜像在处理 Qwen3 嵌入模型时存在 Bug。解决方案很简单：

1. **升级镜像版本**：使用 `v3.12.1-aio-cpu` 或更高版本
2. **验证修复**：通过相似度计算确认模型正常工作
3. **选择合适版本**：建议使用最新的稳定版本以获得最佳体验和安全性

## 参考链接

- [LocalAI GitHub Issue #6257](https://github.com/mudler/LocalAI/issues/6257)
- [LocalAI Docker Hub](https://hub.docker.com/r/localai/localai)
- [Qwen3 模型官方文档](https://qwen.readthedocs.io/)
- [LocalAI 官方文档](https://localai.io/)
