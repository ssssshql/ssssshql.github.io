---
title: LangChain4j ONNX 加载重排序模型 DLL 初始化失败
date: 2026-05-15 11:26:44
tags: [LangChain4j, ONNX, Java, 故障排查]
---

运行官方文档中 ONNX 加载重排序模型的示例代码时，程序直接报错，模型怎么都加载不起来。

## 现象

使用 LangChain4j 1.14.1 的 `OnnxScoringModel` 加载 `Xenova/ms-marco-MiniLM-L-6-v2` 模型，JVM 启动后立即抛出异常：

> java.lang.UnsatisfiedLinkError: C:\Users\Administrator\AppData\Local\Temp\onnxruntime-java7456080606033787453\onnxruntime.dll: Dynamic Link Library (DLL) initialization routine failed

完整堆栈：

```
java.lang.UnsatisfiedLinkError: C:\Users\Administrator\AppData\Local\Temp\onnxruntime-java7456080606033787453\onnxruntime.dll: Dynamic Link Library (DLL) initialization routine failed。
    at java.base/jdk.internal.loader.NativeLibraries.load(Native Method)
    at java.base/jdk.internal.loader.NativeLibraries$NativeLibraryImpl.open(NativeLibraries.java:331)
    at java.base/jdk.internal.loader.NativeLibraries.loadLibrary(NativeLibraries.java:197)
    at java.base/jdk.internal.loader.NativeLibraries.loadLibrary(NativeLibraries.java:139)
    at java.base/java.lang.ClassLoader.loadLibrary(ClassLoader.java:2418)
    at java.base/java.lang.Runtime.load0(Runtime.java:852)
    at java.base/java.lang.System.load(System.java:2025)
    at ai.onnxruntime.OnnxRuntime.load(OnnxRuntime.java:387)
    at ai.onnxruntime.OnnxRuntime.init(OnnxRuntime.java:166)
    at ai.onnxruntime.OrtSession$SessionOptions.<clinit>(OrtSession.java:706)
    at dev.langchain4j.model.scoring.onnx.OnnxScoringModel.<init>(OnnxScoringModel.java:14)
    at com.autotoll.skillstore.example.rag.RagPipelineTest.rerank(RagPipelineTest.java:282)
```

复现代码：

```java
@Test
void rerank(){
    String pathToModel = "C:\\Users\\Administrator\\Desktop\\model.onnx";
    String pathToTokenizer = "C:\\Users\\Administrator\\Desktop\\tokenizer.json";
    OnnxScoringModel scoringModel = new OnnxScoringModel(pathToModel, pathToTokenizer);

    Response<Double> response = scoringModel.score("query", "passage");
    Double score = response.content();
    System.out.println("Score: " + score);
}
```

**环境信息：**

- LangChain4j 版本：1.14.1
- 模型：[Xenova/ms-marco-MiniLM-L-6-v2](https://huggingface.co/Xenova/ms-marco-MiniLM-L-6-v2)
- Java 版本：OpenJDK 21
- Spring Boot 版本：3.4.5
- 系统：Windows 11 x64
- 已安装最新版 Microsoft Visual C++ 2015-2022 Redistributable

## 原因

**真正踩坑点：JDK 自带的旧版 VC 运行时 DLL 覆盖了系统新版本。**

ONNX Runtime 编译依赖较新版本的 Visual C++ 运行时（`msvcp140.dll`、`vcruntime140.dll` 等）。而 OpenJDK 21 的安装目录下自带了**旧版本**的同名 DLL 文件。

Windows 加载 DLL 时，会优先使用进程所在目录（即 JDK 的 `bin/` 目录）下的文件，而不是系统目录下已安装的最新版本。结果就是 ONNX Runtime 拿到了旧的 VC 运行时，DLL 初始化失败。

即使你已经安装了最新版的 VC++ Redistributable，只要 JDK 目录里存在旧版 DLL，Windows 就会优先用 JDK 里的那个，导致问题依旧存在。

查阅 [microsoft/onnxruntime/discussions/23971](https://github.com/microsoft/onnxruntime/discussions/23971) 确认了这个原因。

## 解决方案

### 方案一：删除 JDK 中的旧版 VC 运行时 DLL（当前可用）

从 JDK 的 `bin/` 目录中删除以下文件：

- `msvcp140.dll`
- `vcruntime140.dll`
- `vcruntime140_1.dll`

删除后，Windows 会回退到使用系统目录下安装的最新版 VC++ Redistributable，程序即可正常运行。

> **注意：** 删除 JDK 自带的 DLL 可能影响其他依赖这些文件的本地程序，建议先备份。

### 方案二：更换 JDK 发行版

换用不自带 VC 运行时的 JDK 发行版（如 Adoptium/Temurin），从根源上避免 DLL 冲突。OpenJDK 21 部分发行版使用较旧的 VC 编译，自带了旧版运行时文件，换一个编译环境不同的发行版即可。

## 补充说明

- 本质上是 **Windows DLL 搜索顺序** 引发的问题：进程目录优先级高于系统目录，JDK 里带的旧 DLL 抢先加载，导致新版 ONNX Runtime 初始化失败。
- 如果不想动 JDK 目录，也可以通过设置 `PATH` 环境变量或使用 `java.library.path` 来控制 DLL 加载顺序，但操作相对麻烦，不如直接删文件或换 JDK 来得直接。
- Linux / macOS 上一般不会遇到这个问题，因为它们的 C++ 标准库由系统统一管理，不存在 JDK 自带同名文件的情况。

## 参考链接

- [microsoft/onnxruntime/discussions/23971](https://github.com/microsoft/onnxruntime/discussions/23971)
- [LangChain4j ONNX ScoringModel 文档](https://docs.langchain4j.dev/integrations/embedding-stores/in-memory/)
