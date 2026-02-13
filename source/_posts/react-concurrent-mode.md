---
title: React Concurrent Mode 原理解析：从设计理念到实现细节
date: 2026-02-15 14:30:00
tags: [React, 前端, 性能优化]
categories: [前端技术]
---

React 18 引入的 Concurrent Mode（并发模式）是 React 历史上最重大的架构升级之一。它不仅仅是一组新的 API，更是 React 核心渲染引擎的一次彻底重写。本文将深入剖析 Concurrent Mode 试图解决的问题、其底层的实现原理以及对未来前端开发模式的影响。

<!-- more -->

## 传统的 Stack Reconciler 及其局限性

在 React 16 之前，React 使用的是 **Stack Reconciler**。它的渲染过程是同步且不可中断的。一旦开始渲染组件树，React 就会递归地调用组件的 `render` 方法，直到整棵树遍历完成。

这种“一气呵成”的策略在处理简单的 UI 时表现良好，但在处理大型应用或复杂动画时，就会暴露出问题：

*   **主线程阻塞**：如果组件树很深，渲染时间超过 16ms（一帧的时间），就会导致掉帧，用户感觉到卡顿。
*   **响应迟钝**：在渲染过程中，浏览器无法响应用户的输入（如点击、输入框输入），导致“冻结”感。

## Fiber 架构：并发的基石

为了解决上述问题，React 团队重写了核心算法，引入了 **Fiber 架构**。Fiber 的核心思想是将渲染任务拆分成一个个小的单元（Fiber 节点），使得渲染过程可以被 **中断** 和 **恢复**。

Fiber 节点是一个链表结构，包含了组件的类型、状态、副作用等信息。通过这种数据结构，React 可以在执行完一个 Fiber 单元后，检查剩余时间（`requestIdleCallback` 及其 polyfill）。如果时间不够，就将控制权交还给浏览器，让浏览器去处理优先级更高的任务（如绘制、用户输入）；等浏览器空闲了，再回来继续执行剩下的 Fiber 任务。

## Concurrent Mode 的核心特性

基于 Fiber 架构，React 18 终于完全解锁了 Concurrent Mode 的能力。

### 1. 可中断渲染（Interruptible Rendering）

这是并发模式的灵魂。React 不再一口气把所有更新都提交到 DOM。它可以“在后台”计算新的 UI 状态，同时保持当前的 UI 响应。

例如，当用户在一个搜索框输入时，React 可以优先处理输入框的更新（高优先级），而将下方的搜索结果列表渲染（低优先级）暂时挂起。如果用户继续输入，React 甚至可以丢弃旧的搜索结果渲染任务，直接开始新的。

### 2. 自动批处理（Automatic Batching）

在 React 18 之前，只有在事件处理函数中的状态更新会被批处理。在 Promise、setTimeout 或原生事件回调中的更新是同步发生的，会导致多次重渲染。

React 18 实现了自动批处理，无论在哪里触发状态更新，React 都会尽可能地将它们合并为一次渲染，从而大幅提升性能。

### 3. Transitions API

`useTransition` 和 `startTransition` 是并发模式暴露给开发者的直接控制杆。它们允许开发者将某些更新标记为“非紧急”（Non-urgent）。

```jsx
import { useState, useTransition } from 'react';

function App() {
  const [isPending, startTransition] = useTransition();
  const [count, setCount] = useState(0);

  function handleClick() {
    // 紧急更新：立即反映在 UI 上
    setCount(c => c + 1);

    startTransition(() => {
      // 非紧急更新：可以被中断，可以延迟
      // 比如触发一个昂贵的列表渲染
    });
  }

  return (
    <div>
      {isPending && <Spinner />}
      {/* ... */}
    </div>
  );
}
```

## 深度剖析：双缓存机制（Double Buffering）

Fiber 架构在内存中维护了两棵树：
1.  **Current Tree**：当前屏幕上显示的内容对应的 Fiber 树。
2.  **WorkInProgress Tree**：正在后台构建的新 Fiber 树。

所有的更新都在 `WorkInProgress Tree` 上进行。当整棵树构建完成（即 render 阶段结束），React 只需通过指针交换，将 `WorkInProgress Tree` 变为 `Current Tree`，从而完成 DOM 的更新（commit 阶段）。

这种机制类似于图形学中的双缓存技术，不仅保证了渲染过程中的 UI 一致性（用户不会看到渲染了一半的 UI），也为并发渲染提供了数据结构基础——因为未提交的 `WorkInProgress Tree` 可以随时被丢弃或重新构建，而不会影响用户当前看到的界面。

## 总结

React Concurrent Mode 是前端框架领域的一次大胆尝试。它通过引入操作系统级别的“多任务协作”概念（尽管是在单线程的 JS 中模拟），解决了长期以来困扰前端开发的“CPU 密集型任务阻塞交互”的难题。

对于开发者而言，理解 Concurrent Mode 不仅有助于写出性能更好的 React 应用，更能帮助我们理解现代 UI 库在调度（Scheduling）和资源管理上的演进方向。
