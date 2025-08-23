
# Out-of-Process iframes (OOPIFs)

## 概述

本文概述了 Chromium 对 **进程外 iframe (OOPIFs)** 的支持。
OOPIFs 允许页面的子 iframe 由与父框架不同的进程进行渲染。

---

## 动机

OOPIFs 的提出主要源于 **安全目标**，尤其是 **站点隔离 (Site Isolation)** 项目。

* **站点隔离**：将不同网站的内容放入不同的渲染进程，以增强安全性。

  * 避免恶意网站通过共享进程漏洞攻击其他网站（如跨站脚本 XSS）。
* **OOPIFs 的作用**：

  * 允许 iframe 拥有独立的渲染进程，即使与父页面属于不同站点。
  * 成为实现站点隔离的 **核心机制**。

虽然安全是主要驱动力，但 OOPIFs 也是一个 **通用机制**：

* 用于 Chrome Apps 中的 `<webview>` 标签。
* 用于 PDF 渲染（MimeHandlerView）等场景。

---

## 架构变革

为了支持 OOPIFs，Chromium 做了重大架构调整：

1. **浏览器进程直接跟踪子框架**：不再完全依赖渲染器进程。
2. **核心功能重构**：绘制、输入、导航等关键模块均更新以支持 OOPIFs。
3. **跨进程信息聚合**：例如辅助功能、页面内查找，需要从多个进程中收集信息。

---

## 当前状态与应用

* **M56**：`--isolate-extensions` 模式，OOPIFs 将网页 iframe 移出扩展进程，保护扩展安全。
* **M67**：桌面端 Chrome 为所有网站启用 OOPIFs，全面支持站点隔离。
* **M77**：Android 上为已登录站点启用 OOPIFs。
* **其他应用**：

  * GuestViews / `<webview>`（替代插件基础设施）。
  * MimeHandlerView（渲染 PDF 和其他数据类型）。

---

## 问题报告

遇到 OOPIFs 相关问题，可在 `Internals>Sandbox>SiteIsolation` 组件提交 Bug。

* 总体跟踪 Bug: [crbug.com/99379](https://crbug.com/99379)
* 发布跟踪 Bug: [crbug.com/545200](https://crbug.com/545200)

---

## 项目资源

* **任务依赖图**（各里程碑任务）
* **已知 Bug 列表**：`Internals>Sandbox>SiteIsolation`
* **功能更新 FAQ**：兼容 OOPIFs 的通用方法
* **受影响功能列表**
* **站点隔离峰会演讲**：架构变更及功能更新资料
* **邮件列表**：[site-isolation-dev@chromium.org](mailto:site-isolation-dev@chromium.org)

---

## 架构详解

### 1. 框架表示 (Frame Representation)

* **从 Tab 到 Frame 中心**：content 模块逻辑以 Frame 为核心。
* **跨站交互**：

  * 同站 (Same-Site) → 必须在同一进程（允许同步访问）。
  * 跨站 (Cross-Site) → 可在不同进程，但需要代理与消息路由（postMessage）。
* **SiteInstance**：定义同站文档的安全边界与进程分配。
* **代理 (Proxy)**：

  * 为跨进程交互维护代理 DOMWindow。
  * 例如：站点 A 的页面想给站点 B 的 iframe postMessage → A 进程里有一个 B 的代理，消息通过浏览器进程转发到 B 的真实文档。

### 2. 浏览器进程 (Browser Process)

* **FrameTreeNode**：浏览器进程维护一个与页面框架树一致的对象树。
* **RenderFrameHostManager**：负责跨进程导航与状态复制。

### 3. 渲染器进程 (Renderer Process)

* **代理 DOMWindow**：轻量级，没有完整 Document 或 V8 上下文。
* **RenderFrame / RenderFrameHost**：替代旧的 RenderView，成为每帧的路由单元。

  * LocalFrame → 对应真实文档和 DOM。
  * RemoteFrame → 轻量代理，仅转发消息。
* **本地框架根 (Local Frame Root)**：布局/渲染的同步单位。
* **内存优化**：RemoteFrame 内存极小，避免因 O(框架数 × 进程数) 的代理数量带来爆炸式开销。

### 4. 导航 (Navigation)

* 所有导航在 **浏览器进程** 拦截，而非渲染器。
* 跨站导航时浏览器进程会执行 **进程切换**。
* **会话历史**：

  * 旧模式 → 渲染器维护 HistoryItem 树。
  * 新模式 → 浏览器进程直接维护“每帧历史 + 联合会话历史 (Joint Session History)”。

### 5. 渲染 (Rendering)

* 通过 **Surfaces** 合成不同进程的图像。
* GPU 进程最终拼合为一张输出图像。

### 6. 输入事件 (Input Events)

* 浏览器进程基于 Surfaces 执行 **命中测试**，事件路由到正确的进程。
* 焦点由浏览器进程管理，键盘事件直达目标进程。

---

## 关键总结

1. **目标**：安全隔离，实现强健的站点隔离模型。
2. **本质**：iframe 可拥有独立渲染进程。
3. **核心机制**：

   * 浏览器进程作为中枢，掌控帧树、导航和进程切换。
   * RemoteFrame 代理，用于跨进程通信。
   * Local Frame Root 保证布局和渲染的同步性。
4. **应用场景**：站点隔离、扩展隔离、`<webview>`、PDF 渲染。
5. **挑战与解决**：

   * 导航 → 浏览器进程主导，历史管理重构。
   * 渲染 → Surfaces 合成。
   * 输入 → 命中测试与直投路由。
   * 内存 → RemoteFrame 优化。
   * 功能兼容 → 大量功能需跨进程聚合信息。

---

## 一句话总结

**OOPIFs 把“页面=多帧=多进程”纳入 Chromium 的核心模型：浏览器进程做中枢调度，渲染进程通过 Local/Remote 帧协作，开发者需基于“代理 + 帧树”而非“单进程假设”来实现功能。**

---

要不要我接下来帮你用 **架构图 (Markdown + 图片)** 直观展示：

* 浏览器进程
* 多个渲染器进程
* LocalFrame / RemoteFrame
* Surfaces 合成流程
  这样比文字更清晰。要不要我画出来？
