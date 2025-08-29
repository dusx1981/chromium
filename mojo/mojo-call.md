# mojom 接口跨进程调用（RPC）原理详解

## 一、核心原理概述

mojom 接口的跨进程调用本质上是基于 **消息的异步通信**。
Mojo Bindings API 通过自动生成的代码，将开发者编写的看似“本地”的接口方法调用，透明地转换成在底层消息管道上发送和接收结构化消息的过程。

关键机制包括：

* **代理（Proxy）和存根（Stub）模式**
* **消息管道（Message Pipe）** 作为传输载体

---

## 二、详细流程与原理分析

假设有一个简单的 mojom 接口定义：

```mojom
module example.mojom;

interface Calculator {
    Add(int32 a, int32 b) => (int32 sum);
};
```

进程 A（客户端）调用进程 B（服务端）的 `Add` 方法。

---

### 步骤 1: 代码生成 (The Power of mojom IDL)

1. 编译 `.mojom` 文件：

   * Mojo 的编译工具（如 `mojom_bindings_generator.py`）解析 `Calculator.mojom`。
2. 生成语言特定代码（如 C++）：

   * **Calculator**：纯虚接口类，服务端继承并实现 `Add` 方法。
   * **CalculatorProxy**：客户端持有的对象，调用方法时发起跨进程调用。
   * **CalculatorStub**：内部类，接收管道消息并分派给服务端实现。
   * **序列化/反序列化代码**：自动生成，将方法参数和返回值在内存与二进制消息间转换。

---

### 步骤 2: 建立连接 (Acquiring the Pipe)

1. 创建消息管道：`mojo::MessagePipe`，包含两个端点 `handle0` 和 `handle1`。
2. 端点分配：

   * 服务端 B 获得 `handle1`
   * 客户端 A 获得 `handle0`
3. 绑定端点：

   * 服务端：

     ```cpp
     mojo::MakeSelfOwnedReceiver(
         std::make_unique<CalculatorImpl>(),
         mojo::PendingReceiver<example::mojom::Calculator>(std::move(handle1)));
     ```

     * 创建 `mojo::Receiver<Calculator>`，监听 `handle1` 消息。
     * Receiver 扮演 **Stub**：解析消息，调用 `impl->Add(...)`，序列化结果返回客户端。
   * 客户端：

     ```cpp
     mojo::Remote<example::mojom::Calculator> calculator_remote;
     calculator_remote.Bind(
         mojo::PendingRemote<example::mojom::Calculator>(std::move(handle0)));
     ```

     * `mojo::Remote<Calculator>` 内部管理 `CalculatorProxy` 对象。
     * 调用 `calculator_remote->Add(...)` 实际调用 `CalculatorProxy::Add`。

---

### 步骤 3: 发起调用 (Client Side - Proxy)

1. 客户端调用：

   ```cpp
   calculator_remote->Add(5, 3, base::BindOnce(&OnAddDone));
   ```
2. Proxy 工作：

   * `CalculatorProxy::Add` 被调用
   * 参数序列化为结构化二进制消息，包括：

     * 接口标识符（Calculator）
     * 方法标识符（Add）
     * 参数 a, b
     * 唯一请求 ID
     * 异步回调关联请求 ID
3. 发送消息到管道端点 `handle0`（非阻塞）

---

### 步骤 4: 接收调用 & 执行 (Server Side - Receiver/Stub)

1. 消息到达服务端 `handle1`
2. Receiver 解析消息：

   * 确认接口和方法
   * 获取请求 ID
   * 参数反序列化
3. 调用实现：

   ```cpp
   int32 sum = CalculatorImpl::Add(a, b);
   ```
4. 执行计算得到结果（例如 `sum = 8`）

---

### 步骤 5: 发送响应 (Server Side - Stub)

1. Stub 序列化返回值 `sum`，生成响应消息
2. 写入同一管道端点 `handle1`，请求和响应独立、有序

---

### 步骤 6: 接收响应 (Client Side - Proxy)

1. 响应到达客户端 `handle0`
2. Proxy 解析消息：

   * 解析请求 ID
   * 查找对应回调对象
   * 反序列化返回值
3. 执行回调：

   ```cpp
   OnAddDone(sum); // sum = 8
   ```
4. 客户端处理结果

---

## 三、关键机制与技术点

1. **代理-存根 (Proxy-Stub) 模式**

   * Proxy：拦截方法调用，发送消息请求
   * Stub：接收消息，调用实现并发送响应
   * 自动生成大量重复代码
2. **消息管道 (Message Pipe)**

   * 双向、有序、可靠
   * 可承载多个接口调用，通过接口 ID 区分
3. **序列化与反序列化**

   * 将内存数据转换为平台无关二进制消息
   * IDL 编译器生成高效、安全代码
4. **异步通信**

   * 默认异步调用，立即返回
   * 回调返回结果，避免阻塞 UI 或关键线程
5. **请求 ID**

   * 唯一标识每次异步调用
   * 关联请求与响应
6. **句柄传递**

   * 支持 MessagePipe、DataPipe、SharedBuffer 等句柄
   * 序列化过程处理跨进程传递
7. **关联接口 (Associated Interfaces)**

   * 在主管道上复用逻辑独立的子接口通道
   * 保证消息顺序
   * 共享底层物理管道

---

## 四、总结

mojom 跨进程调用原理核心：

* 利用 **IDL 生成代码**
* 通过 **Proxy-Stub 抽象层**
* 使用 **消息管道传输层**

优势：

* 开发者可像调用本地函数一样 RPC
* 异步特性契合高性能 GUI 应用
* 高级特性（句柄传递、关联接口）支持复杂 IPC 模式
* 在易用性、性能、安全性之间取得平衡
