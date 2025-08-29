# mojom 接口跨进程调用 (RPC) 原理详解

## 一、核心原理概述

mojom 接口的跨进程调用本质上是基于**消息的异步通信**。
Mojo Bindings API 通过自动生成的代码，将开发者编写的“本地方法调用”，转换成底层消息管道上的消息收发。

关键机制：

* **代理 (Proxy)**：客户端调用方法时，序列化参数并发送消息。
* **存根 (Stub)**：服务端接收消息，反序列化参数并调用真实实现，再序列化返回结果。
* **消息管道 (Message Pipe)**：跨进程传输的核心载体。

---

## 二、示例接口定义

```mojom
module example.mojom;

interface Calculator {
    Add(int32 a, int32 b) => (int32 sum);
};
```

---

## 三、详细流程解析

假设进程 A（客户端）调用进程 B（服务端）的 `Add` 方法。

### 步骤 1: 代码生成 (IDL 编译)

1. **编译 `.mojom` 文件**：
   `mojom_bindings_generator.py` 解析 `Calculator.mojom`。
2. **生成语言绑定代码（以 C++ 为例）**：

   * `Calculator`：纯虚接口类，服务端实现。
   * `CalculatorProxy`：客户端使用的代理类（通过 `mojo::Remote` 持有）。
   * `CalculatorStub`：服务端消息分发器，负责解析消息并调用真实实现。
   * 序列化/反序列化代码：参数与返回值转换。

---

### 步骤 2: 建立连接 (Message Pipe)

1. **创建消息管道**：

   ```cpp
   mojo::MessagePipe pipe;
   ```

   * `handle0` 给客户端
   * `handle1` 给服务端

2. **绑定端点**：

   * **服务端**：

     ```cpp
     mojo::MakeSelfOwnedReceiver(
         std::make_unique<CalculatorImpl>(),
         mojo::PendingReceiver<example::mojom::Calculator>(std::move(handle1)));
     ```

     * Receiver 持有 `handle1`，监听消息并调用 `CalculatorImpl`。

   * **客户端**：

     ```cpp
     mojo::Remote<example::mojom::Calculator> calculator_remote;
     calculator_remote.Bind(
         mojo::PendingRemote<example::mojom::Calculator>(std::move(handle0)));
     ```

     * `Remote` 内部持有 `CalculatorProxy`，封装调用逻辑。

---

### 步骤 3: 发起调用 (客户端 Proxy)

1. 客户端代码：

   ```cpp
   calculator_remote->Add(5, 3, base::BindOnce(&OnAddDone));
   ```

2. Proxy 行为：

   * 序列化参数 (5, 3)。
   * 打包消息：接口 ID、方法 ID、请求 ID、参数。
   * 存储回调函数 `OnAddDone`，等待响应。
   * 消息写入管道 (`handle0`)。

---

### 步骤 4: 接收调用 & 执行 (服务端 Receiver/Stub)

1. Receiver 检测到消息到达 (`handle1`)。
2. 解析消息：接口 ID → 方法 ID (`Add`) → 请求 ID → 参数 (5, 3)。
3. 调用 `CalculatorImpl::Add(5, 3)`。
4. 得到结果 `sum = 8`。

---

### 步骤 5: 发送响应 (服务端 Stub)

1. 序列化返回值 `sum = 8`。
2. 包装响应消息：包含原始请求 ID。
3. 写入消息管道 (`handle1`)。

---

### 步骤 6: 接收响应 (客户端 Proxy)

1. Proxy 从 `handle0` 读取响应消息。
2. 解析消息：找到请求 ID → 匹配到回调 `OnAddDone`。
3. 反序列化返回值 `sum = 8`。
4. 调用回调：

   ```cpp
   void OnAddDone(int32_t sum) {
       // sum == 8
   }
   ```

---

## 四、关键机制与技术点

1. **代理-存根 (Proxy-Stub) 模式**

   * Proxy：拦截方法调用，转换为消息。
   * Stub：接收消息，调用实现，并返回结果。

2. **消息管道 (Message Pipe)**

   * 双向、有序、可靠。
   * 一个管道可承载多个接口调用（接口 ID 区分）。

3. **序列化与反序列化**

   * 参数和返回值转换为紧凑的二进制格式。
   * 由 Mojom 编译器自动生成。

4. **异步通信**

   * 默认异步，调用立即返回。
   * 结果通过回调返回，避免线程阻塞。

5. **请求 ID (Request ID)**

   * 唯一标识请求与响应的对应关系。
   * 支持并发调用。

6. **句柄传递 (Handle Passing)**

   * 支持传递其他管道、共享内存等资源。
   * 底层使用平台机制（如 Linux 的 `SCM_RIGHTS`）。

7. **关联接口 (Associated Interfaces)**

   * 在同一管道上复用多个逻辑子接口。
   * 保证严格的消息顺序。

---

## 五、总结

* **核心思想**：
  利用 IDL 自动生成的 Proxy-Stub 框架 + 消息管道，实现跨进程 RPC。

* **开发者视角**：
  像调用本地函数一样调用远程接口，无需关心底层序列化、消息传输和同步。

* **特性优势**：

  * 异步高效，契合事件驱动模型。
  * 支持句柄传递和接口复用，极大增强灵活性。
  * 自动生成代码保证类型安全与性能。

Mojo 通过这种设计，在 **易用性、性能和安全性** 之间取得了很好的平衡。
