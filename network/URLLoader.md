# URLLoader 接口梳理与翻译

## 1. 总体概述与翻译

* **接口名称**：`URLLoader`
* **中文翻译**：URL 加载器

**核心作用**：
`URLLoader` 是一个用于执行 **单次 URL 请求** 的接口。它是客户端（例如浏览器渲染进程）与网络服务（通常在浏览器主进程或独立服务进程中）进行通信的抽象层。

**关键行为**：

* 当 `URLLoader` 对象被销毁时，其发起的关联网络请求会被取消。
* 它通常不直接实例化，而是通过 `URLLoaderFactory` 接口的 `CreateLoaderAndStart` 方法来创建并立即启动一个请求。
* 请求的结果（如响应数据、重定向、错误等）通过另一个名为 `URLLoaderClient` 的接口回调给调用方。

---

## 2. 上下文总结

该接口定义源自 **Chromium 项目网络栈**，通过 **Mojo（IPC 框架）** 定义，用于跨进程通信。

* **URLLoader (Service Side)**：存在于网络服务中，负责实际处理网络请求（如 HTTP 请求）。
* **URLLoaderClient (Client Side)**：存在于客户端（如渲染进程），用于接收来自 `URLLoader` 的通知（如响应、重定向等）。
* **URLLoaderFactory**：创建 `URLLoader` 实例的工厂接口，客户端通过它来发起新的请求。

👉 本定义描述的是 **URLLoader 服务端接口**，客户端通过 Mojo 连接调用其方法。

---

## 3. 逐部分分析与翻译

### 常量定义

```cpp
// If a disconnection is initiated by the client side, it may send the
// following disconnection reason, along with an application-defined string
// description, to notify the service side.
const uint32 kClientDisconnectReason = 1;
```

**翻译：**

```cpp
// 如果由客户端发起了连接断开操作，它可能会发送以下断开连接原因代码，
// 以及一个由应用程序定义的字符串描述，用以通知服务端。
const uint32 kClientDisconnectReason = 1; // 客户端断开连接原因代码
```

**解释**：
这是一个预定义的错误代码常量。当客户端主动关闭与 `URLLoader` 服务的连接时（例如页面跳转后不再需要请求），可使用该代码告知服务端：这是正常行为，而不是错误。

---

### 方法：FollowRedirect

```cpp
// Upon receiving a redirect through |URLLoaderClient::OnReceiveRedirect|,
// |FollowRedirect| may be called to load the URL indicated by the redirect.
//
// |removed_headers| can be used to remove existing headers for the redirect.
// This parameter is before |modified_headers| since removing headers is
// applied first in the URLLoader::FollowRedirect().
//
// |modified_headers| can be used to add or override existing headers for the
// redirect. |modified_cors_exempt_headers| can be used to modify
// |cors_exempt_headers| in the URLRequest. See
// NetworkContextParams::cors_exempt_header_list and
// URLRequest::cors_exempt_headers for details.
//
// If |new_url| is specified, then the request will be made to it instead of
// the redirected URL. |new_url| must be same-origin to the redirected URL.
FollowRedirect(array<string> removed_headers,
               network.mojom.HttpRequestHeaders modified_headers,
               network.mojom.HttpRequestHeaders modified_cors_exempt_headers,
               url.mojom.Url? new_url);
```

**翻译：**

```cpp
// 当通过 |URLLoaderClient::OnReceiveRedirect| 接收到重定向通知后，
// 可以调用 |FollowRedirect| 方法来加载重定向指示的 URL。
//
// |removed_headers| 可用于移除重定向请求中的现有头信息。
// 在 URLLoader::FollowRedirect() 中，移除头信息操作先于修改操作执行。
//
// |modified_headers| 可用于添加或覆盖新的请求头。
// |modified_cors_exempt_headers| 可用于修改 CORS 豁免的请求头。
// 详见 NetworkContextParams::cors_exempt_header_list 和 URLRequest::cors_exempt_headers。
//
// 如果指定了 |new_url|，则请求将发送至该 URL，而不是重定向的 URL。
// 注意：|new_url| 必须与重定向 URL 同源。
FollowRedirect(
  数组<string> removed_headers,                       // 要移除的请求头
  network.mojom.HttpRequestHeaders modified_headers,  // 要修改/新增的请求头
  network.mojom.HttpRequestHeaders modified_cors_exempt_headers, // 要修改的 CORS 豁免请求头
  url.mojom.Url? new_url                              // 可选的新 URL（必须同源）
);
```

**解释**：

1. **触发时机**：收到 `OnReceiveRedirect` 通知时。
2. **客户端决策**：决定是否跟随重定向。
3. **可修改请求**：

   * `removed_headers` → 移除敏感头（如 Cookie）。
   * `modified_headers` → 修改或新增头部。
   * `modified_cors_exempt_headers` → 修改 CORS 豁免头。
   * `new_url` → 可覆盖重定向目标 URL（但必须同源，保证安全）。

---

### 方法：SetPriority

```cpp
// Sets the request priority.
// |intra_priority_value| is a lesser priority which is used to prioritize
// requests within a given priority level. If -1 is passed, the existing
// intra priority value is maintained.
SetPriority(RequestPriority priority, int32 intra_priority_value);
```

**翻译：**

```cpp
// 设置请求的优先级。
// |intra_priority_value| 是次级优先级，用于在相同主优先级下排序。
// 如果传入 -1，则保持当前值不变。
SetPriority(
  RequestPriority priority,  // 主优先级（如 LOW / MEDIUM / HIGH）
  int32 intra_priority_value // 次级优先级（-1 表示不变）
);
```

**解释**：

* **动态调整**：允许在请求发出后改变其优先级。
* **应用场景**：图片懒加载，滚动到视口内时提升优先级。
* **两级优先级**：

  * 主优先级：全局排序依据（LOW / MEDIUM / HIGH）。
  * 次级优先级：同级内进一步排序。

---

## 4. 最终总结

**URLLoader 接口作用**：
它是 **单个网络请求生命周期** 的核心服务端接口。客户端通过 Mojo IPC 调用其方法，主要功能有：

1. **处理重定向**：通过 `FollowRedirect` 控制是否继续请求，并可修改新请求的 HTTP 头信息。
2. **调整调度策略**：通过 `SetPriority` 动态调整请求优先级，优化页面加载性能与用户体验。
3. **生命周期管理**：销毁 `URLLoader` 会立即取消其请求，保证了请求控制的灵活性。

👉 **简而言之**：
`URLLoader` 将网络请求抽象为一个可远程控制的对象，提供了强大的 **请求管理能力**。

