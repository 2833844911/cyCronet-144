# Cronet HTTP Client (Python)

这是一个基于 [Cronet](https://developer.android.com/guide/topics/connectivity/cronet) 的 Python HTTP 客户端封装，提供了类似 `httpx` 和 `requests` 风格的 API。它支持同步和异步请求，并内置了强大的会话管理和代理支持。

## 测试

**使用 https://tls.jsvmp.top:38080 tls检测站点测试（test_tls_verify.py）tls功能是否操作问题，现有的免费使用的库都无法通过,大部分编译的cronet库也在使用代理的情况下也存在问题，当前项目已经修复已知的协议指纹问题。**

##  特性

- **高性能**: 基于 Google Cronet 网络库，支持 HTTP/1.1, HTTP/2, and HTTP/3 (QUIC).
- **多模式**: 支持 **同步 (Sync)** 和 **异步 (Async)** 两种调用方式.
- **会话保持**: 自动管理 Cookie，支持会话级别的 Cookie 保持（类似浏览器的行为）.
- **代理支持**: 支持 HTTP 和 SOCKS5 代理配置，支持代理认证.
- **TLS 配置**: 支持跳过 SSL 证书验证.
- **易用 API**: 设计风格贴近 `requests` 和 `aiohttp`，易于上手.

##  准备工作

在使用本客户端之前，由于其底层依赖于 `cronet-cloak` 服务作为后端，请确保：

1.  **启动后端服务**: 运行项目目录下的 `cronet/cronet-cloak.exe`。
    -   该服务默认监听本地 `3000` 端口。
    -   客户端的所有请求都会转发给这个本地服务处理。
    -   保存请求debug模式启动命令：`cronet-cloak.exe --debug` 查看请求原始格式是否正常

2.  **安装依赖**:
    ```bash
    pip install aiohttp requests
    ```

##  快速开始

### 1. 同步客户端 (`CronetClient`)

适用于习惯 `requests` 库或不需要异步特性的场景。

```python
from cronet_client import CronetClient

# 使用 with 语句自动管理会话（推荐）
with CronetClient() as client:
    # GET 请求
    resp = client.get("https://httpbin.org/get")
    print(resp.status_code)
    print(resp.json())

    # POST 请求 (自动处理 JSON)
    resp = client.post("https://httpbin.org/post", json={"key": "value"})
    print(resp.json())
```

### 2. 异步客户端 (`AsyncCronetClient`)

适用于高并发场景，基于 `asyncio` 和 `aiohttp`。

```python
import asyncio
from cronet_client import AsyncCronetClient

async def main():
    async with AsyncCronetClient() as client:
        # 异步 GET
        resp = await client.get("https://httpbin.org/get")
        print(f"Status: {resp.status_code}, Latency: {resp.duration_ms}ms")

        # 异步并发请求
        tasks = [client.get(f"https://httpbin.org/get?id={i}") for i in range(3)]
        results = await asyncio.gather(*tasks)
        for r in results:
            print(r.status_code)

if __name__ == "__main__":
    asyncio.run(main())
```

##  高级功能

### 代理配置

支持 HTTP 和 SOCKS5 代理。

```python
from cronet_client import CronetClient, Proxy

# 配置代理
my_proxy = Proxy(
    host="127.0.0.1",
    port=7890,
    type=0,  # 0: HTTP, 1: SOCKS5
    username="", #可选
    password=""  #可选
)

with CronetClient(proxy=my_proxy) as client:
    resp = client.get("https://httpbin.org/ip")
    print(resp.text)
```

### 会话与 Cookie 管理

客户端会自动处理 Cookie 的接收和发送，就像浏览器一样。

-   **自动保持**: 服务器返回的 `Set-Cookie` 会被自动存储，并在后续对同一域名的请求中自动携带。
-   **手动设置**: 可以在请求时通过 `cookies` 参数传入额外的 Cookie，它们会与会话 Cookie 合并。

```python
with CronetClient() as client:
    # 第一次请求，服务器设置 Cookie
    client.get("https://httpbin.org/cookies/set/session_id/123456")

    # 第二次请求，会自动带上 session_id=123456
    resp = client.get("https://httpbin.org/cookies")
    print(resp.json()) # {'cookies': {'session_id': '123456'}}

    # 查看当前会话的所有 Cookie
    print(client.cookies)
```

### 跳过证书验证

在开发或测试环境中，如果需要忽略 SSL 证书错误：

```python
client = CronetClient(skip_cert_verify=True)
```

##  API 参考

### `CronetClient` / `AsyncCronetClient`

构造函数参数：
-   `base_url`: 后端服务地址，默认为 `http://127.0.0.1:3000/api/v1`。
-   `proxy`: `Proxy` 对象，可选。
-   `timeout`: 请求超时时间（秒），默认 30.0。
-   `skip_cert_verify`: 是否跳过 SSL 验证，默认 `True`。

主要方法：
-   `get(url, headers=None, cookies=None)`
-   `post(url, headers=None, cookies=None, content=None, data=None, json=None)`
-   `put(...)`
-   `delete(...)`
-   `patch(...)`

### `Response` 对象

请求返回的对象包含以下属性：
-   `status_code`: HTTP 状态码 (int)
-   `headers`: 响应头 (Dict)
-   `content`: 响应原始内容 (bytes)
-   `text`: 响应文本 (str)
-   `json()`: 解析 JSON 响应
-   `duration_ms`: 请求耗时 (ms)

---
更多示例请参考 `demo_cronet_client.py`。
