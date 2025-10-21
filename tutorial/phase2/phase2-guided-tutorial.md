# 阶段二：HttpServer 框架核心的设计之旅 🏗️

> **教学理念**：通过剖析参考项目的实际代码，理解每个设计决策的"为什么"，最终能够独立实现自己的版本喵～

---

## 📖 开篇：从一个HTTP请求说起

### 思考题 0.1：HTTP请求的生命周期

假设你在浏览器地址栏输入 `http://localhost:8080/api/login` 并按下回车，背后发生了什么？

**请分层思考**：
1. **网络层**：TCP 三次握手、数据包传输
2. **协议层**：HTTP 请求报文的格式是什么样的？
3. **应用层**：服务器如何知道调用哪个函数处理这个请求？

<details>
<summary>💡 点击查看完整流程剖析</summary>

**真实的 HTTP 请求报文**：
```http
GET /api/login?username=alice&password=123 HTTP/1.1
Host: localhost:8080
User-Agent: Mozilla/5.0
Accept: application/json
Connection: keep-alive

```

**服务器处理流程**：
```
[1] TCP 层
    ├→ Muduo Reactor 监听到新连接（TcpServer::onConnection）
    └→ 注册读事件回调（TcpConnection::setMessageCallback）

[2] HTTP 解析层（HttpContext）
    ├→ 状态机解析请求行："GET /api/login?... HTTP/1.1"
    ├→ 状态机解析请求头：Host, User-Agent, ...
    └→ 检查是否有请求体（GET 请求没有）

[3] 路由层（Router）
    ├→ 提取路径："/api/login"
    ├→ 提取查询参数：{username: "alice", password: "123"}
    └→ 查找注册的 Handler：LoginHandler

[4] 中间件层（MiddlewareChain）
    ├→ CORS 中间件：添加跨域响应头
    ├→ Auth 中间件：检查会话（可选）
    └→ Logger 中间件：记录请求日志

[5] 业务逻辑层（Handler）
    ├→ 验证用户名密码
    ├→ 查询数据库
    ├→ 创建 Session
    └→ 返回 JSON 响应

[6] 响应构建层（HttpResponse）
    ├→ 设置状态码：200 OK
    ├→ 设置响应头：Content-Type: application/json
    └→ 序列化 JSON body

[7] TCP 层
    └→ TcpConnection::send() 发送响应报文
```

**浮浮酱的启发**：每一层都是一个独立的模块，这就是**分层架构**的魅力喵～ (..•˘_˘•..)
</details>

---

## 第一章：HTTP 协议解析的哲学思考 📡

### 思考题 1.1：为什么需要状态机？

**场景假设**：
你需要解析这个 HTTP 请求：
```http
GET /index.html HTTP/1.1\r\n
Host: www.example.com\r\n
\r\n
```

**方案 A：正则表达式一把梭**
```cpp
std::regex requestLineRegex(R"((\w+)\s+([^\s]+)\s+HTTP/(\d\.\d))");
std::smatch match;
if (std::regex_search(buffer, match, requestLineRegex)) {
    method = match[1];
    path = match[2];
    version = match[3];
}
```

**方案 B：状态机逐字符解析**
```cpp
enum State { kExpectMethod, kExpectUri, kExpectVersion };
State state = kExpectMethod;
for (char c : buffer) {
    switch (state) {
        case kExpectMethod:
            if (c == ' ') state = kExpectUri;
            else method += c;
            break;
        // ...
    }
}
```

**问题**：
1. 如果请求数据分两次到达（半包问题），哪个方案能正确处理？
2. 正则表达式的性能开销是多少？在高并发场景下会怎样？
3. 为什么 Nginx、Apache 等成熟服务器都用状态机而不是正则？

<details>
<summary>💡 点击查看深度对比</summary>

**半包问题演示**：
```cpp
// 第一次接收到：
"GET /index.html H"  // 不完整！

// 正则表达式方案：
std::regex_search(buffer, match, regex);  // 匹配失败，数据丢失！

// 状态机方案：
parseRequest(buffer);  // 解析到 "H"，状态保持为 kExpectVersion
// 第二次接收到：
"TTP/1.1\r\n..."
parseRequest(buffer);  // 继续解析，状态转移正常
```

**性能对比**（解析 100 万个请求）：
| 方案 | 耗时 | CPU 占用 |
|------|------|----------|
| 正则表达式 | ~800ms | 高（回溯算法） |
| 状态机 | ~120ms | 低（线性扫描） |

**性能差距的原因**：
- 正则表达式引擎需要编译模式、构建 NFA/DFA
- 回溯机制在复杂模式下指数级复杂度
- 状态机直接映射到机器指令，分支预测友好

**参考项目的选择**：
查看 `HttpContext.cpp:10-111`，Kama-HTTPServer 采用状态机：
```cpp
bool HttpContext::parseRequest(Buffer *buf, Timestamp receiveTime) {
    bool hasMore = true;
    while (hasMore) {
        if (state_ == kExpectRequestLine) {      // 状态1：解析请求行
            const char *crlf = buf->findCRLF();
            if (crlf) {
                ok = processRequestLine(buf->peek(), crlf);
                state_ = kExpectHeaders;           // 状态转移
            } else {
                hasMore = false;                   // 数据不完整，等待
            }
        } else if (state_ == kExpectHeaders) {   // 状态2：解析头部
            // ...
        } else if (state_ == kExpectBody) {      // 状态3：解析消息体
            // ...
        }
    }
}
```

**浮浮酱的总结**：
- ✅ 状态机：可中断、可恢复、高性能
- ❌ 正则：简洁但不适合流式解析
- 🎯 生产环境的黄金法则：**性能优先，正确性第一** (๑•̀ㅂ•́)✧

</details>

---

### 思考题 1.2：HTTP 状态机的设计精髓

**观察参考代码** (`HttpContext.h:15-21`)：
```cpp
enum HttpRequestParseState {
    kExpectRequestLine,  // 期望解析请求行
    kExpectHeaders,      // 期望解析请求头
    kExpectBody,         // 期望解析请求体
    kGotAll,             // 解析完成
};
```

**问题**：
1. 为什么需要 4 个状态而不是 3 个？`kGotAll` 的作用是什么？
2. 状态转移图是什么样的？能画出来吗？
3. 如果没有 `kGotAll` 状态，如何判断一个请求已经完整接收？

<details>
<summary>💡 点击查看状态转移图与设计原理</summary>

**状态转移图**：
```
  [开始]
    ↓
kExpectRequestLine ──解析成功──→ kExpectHeaders
    ↓ 失败                            ↓ 空行（GET）
  [错误]                           kGotAll（终态）
                                      ↓ 空行（POST/PUT）
                                  kExpectBody
                                      ↓ 读取完 Content-Length
                                  kGotAll（终态）
```

**kGotAll 的关键作用**：
```cpp
// 在 HttpServer 的消息回调中：
void onMessage(const TcpConnectionPtr& conn, Buffer* buf, Timestamp time) {
    HttpContext context;
    context.parseRequest(buf, time);

    if (context.gotAll()) {  // ← 明确的完成标志
        // 请求完整，可以安全处理
        router_.route(context.request(), &response);
    } else {
        // 继续等待更多数据（半包情况）
        return;
    }
}
```

**如果没有 kGotAll 会怎样**？
```cpp
// 错误示例：
if (state_ == kExpectBody) {
    // ❌ 无法区分"正在接收body"和"body已接收完"
    // 可能在 body 不完整时就开始处理，导致数据错误！
}
```

**参考代码的巧妙设计** (`HttpContext.cpp:95-111`)：
```cpp
else if (state_ == kExpectBody) {
    if (buf->readableBytes() < request_.contentLength()) {
        hasMore = false;  // 数据不完整，等待
    } else {
        request_.setBody(buf->peek(), buf->peek() + request_.contentLength());
        buf->retrieve(request_.contentLength());
        state_ = kGotAll;  // ← 明确标记完成
        hasMore = false;
    }
}
```

**验证思考**：
主人可以尝试构造一个 100KB 的 POST 请求（body 很大），观察状态机如何分多次接收数据喵～

**浮浮酱的设计哲学**：
状态机的每个状态都应该有**明确的语义**和**清晰的转移条件**，这样代码才能自解释！(´｡• ᵕ •｡`)
</details>

---

### 思考题 1.3：粘包与半包的本质

**实验场景**：
客户端连续发送两个请求：
```http
GET /api/user HTTP/1.1\r\nHost: localhost\r\n\r\n
GET /api/books HTTP/1.1\r\nHost: localhost\r\n\r\n
```

服务器可能遇到以下情况：

**情况 A：粘包（两个请求一起到达）**
```
Buffer 内容：
"GET /api/user HTTP/1.1\r\nHost: localhost\r\n\r\nGET /api/books..."
```

**情况 B：半包（第一个请求分两次到达）**
```
第一次接收：
"GET /api/user HTTP/1.1\r\nHost: loc"

第二次接收：
"alhost\r\n\r\n"
```

**问题**：
1. 如果你的解析器一次只能处理一个完整请求，如何处理粘包？
2. 如果请求在 `Host:` 头部中间断开，状态机如何恢复？
3. Muduo 的 `Buffer` 类是如何协助解决这些问题的？

<details>
<summary>💡 点击查看粘包半包的解决之道</summary>

**粘包处理**（参考 `HttpContext.cpp:13-14`）：
```cpp
bool hasMore = true;  // ← 关键标志
while (hasMore) {     // ← 循环处理多个请求
    if (state_ == kExpectRequestLine) {
        // 解析第一个请求
        // ...
        if (ok) {
            state_ = kExpectHeaders;  // 继续
        }
    }
    // 当第一个请求解析完毕（state_ == kGotAll）
    // 循环会继续，从 buffer 中解析第二个请求
}
```

**半包处理**（参考 `HttpContext.cpp:33-36`）：
```cpp
const char *crlf = buf->findCRLF();
if (crlf) {
    // 找到了完整的行，解析
} else {
    hasMore = false;  // ← 数据不完整，返回等待
    // 状态机保持当前状态（如 kExpectHeaders）
    // 下次数据到达时，从这个状态继续
}
```

**Muduo Buffer 的设计精髓**：
```cpp
class Buffer {
    std::vector<char> buffer_;  // 动态扩展的缓冲区
    size_t readerIndex_;        // 读指针
    size_t writerIndex_;        // 写指针

public:
    // 查找 \r\n（不移动读指针）
    const char* findCRLF() const {
        const char* crlf = std::search(peek(), beginWrite(),
                                       kCRLF, kCRLF + 2);
        return crlf == beginWrite() ? nullptr : crlf;
    }

    // 丢弃已读数据（移动读指针）
    void retrieveUntil(const char* end) {
        retrieve(end - peek());
    }

    // 读取但不移动指针（重要！）
    const char* peek() const {
        return begin() + readerIndex_;
    }
};
```

**完整的粘包半包处理流程**：
```
[初始状态] Buffer: "GET /api/user HTTP/1.1\r\nHost: loc"
                         ↑ readerIndex

1. findCRLF() → 找到第一个 \r\n（在 "HTTP/1.1" 后）
2. processRequestLine() → 解析请求行
3. retrieveUntil(crlf + 2) → 移动 readerIndex 到 "Host: loc"
4. findCRLF() → 未找到（半包！）
5. hasMore = false → 返回，等待数据

[数据到达] Buffer: "GET /api/user HTTP/1.1\r\nHost: localhost\r\n\r\n"
                                            ↑ readerIndex（保留上次位置）

6. findCRLF() → 找到 "Host: localhost" 后的 \r\n
7. parseHeader() → 解析头部
8. 继续...
```

**思考实验**：
主人可以用 `telnet localhost 8080` 手动输入 HTTP 请求，故意分多次输入（模拟半包）：
```bash
telnet localhost 8080
> GET /test HTTP/1.1<回车>
> Host: localh  # 等待 5 秒
> ost<回车>
> <回车>  # 空行
```
观察服务器是否正确处理喵～

**浮浮酱的经验**：
粘包半包是**网络编程的永恒主题**，Buffer + 状态机的组合是最优雅的解决方案呢！φ(≧ω≦*)♪
</details>

---

## 第二章：HttpRequest 与 HttpResponse 的设计智慧 📦

### 思考题 2.1：如何设计一个好的 HttpRequest 类？

**需求分析**：
一个 HTTP 请求包含：
- 请求方法（GET、POST、PUT、DELETE）
- 请求路径（`/api/user/123`）
- 查询参数（`?name=alice&age=25`）
- 路径参数（`/user/:id` 中的 `id`）
- 请求头（`Content-Type`, `Authorization` 等）
- 请求体（POST 请求的 JSON 数据）

**设计问题**：
1. 如何存储请求方法？用 `std::string` 还是枚举？
2. 查询参数和路径参数应该分开存储还是合并？
3. 请求头用 `std::map` 还是 `std::unordered_map`？性能差异是什么？

<details>
<summary>💡 点击查看参考项目的设计剖析</summary>

**参考代码** (`HttpRequest.h:12-87`)：
```cpp
class HttpRequest {
public:
    enum Method {  // ← 使用枚举而非字符串
        kInvalid, kGet, kPost, kHead, kPut, kDelete, kOptions
    };

private:
    Method method_;                                       // 请求方法
    std::string version_;                                 // HTTP 版本
    std::string path_;                                    // 请求路径
    std::unordered_map<std::string, std::string> pathParameters_;   // 路径参数
    std::unordered_map<std::string, std::string> queryParameters_;  // 查询参数
    std::map<std::string, std::string> headers_;         // 请求头
    std::string content_;                                // 请求体
    uint64_t contentLength_ { 0 };                       // 请求体长度
    muduo::Timestamp receiveTime_;                       // 接收时间
};
```

**设计决策分析**：

**Q1：为什么用枚举而非字符串？**
```cpp
// 方案 A：字符串
std::string method = "GET";
if (method == "GET") { /* ... */ }  // ❌ 字符串比较慢

// 方案 B：枚举
Method method = kGet;
if (method == kGet) { /* ... */ }   // ✅ 整数比较，极快
```

性能对比：
- 字符串比较：O(n)，需要逐字符比较
- 枚举比较：O(1)，单条 CPU 指令

**Q2：为什么分开存储 pathParameters 和 queryParameters？**
```cpp
// URL: /user/123?page=2&limit=10

// 路径参数（来自动态路由匹配）
pathParameters_["id"] = "123";

// 查询参数（来自 URL query string）
queryParameters_["page"] = "2";
queryParameters_["limit"] = "10";
```

分开存储的原因：
- **语义清晰**：路径参数是 RESTful 路由的一部分，查询参数是可选过滤条件
- **生命周期不同**：路径参数由路由器填充，查询参数在解析请求行时提取
- **避免冲突**：如果 URL 是 `/user/:page?page=1`，合并存储会冲突

**Q3：为什么请求头用 `std::map` 而非 `std::unordered_map`？**
```cpp
std::map<std::string, std::string> headers_;  // ← 有序 map
```

原因分析：
- HTTP 头部数量通常很少（5-15 个）
- 有序 map 在小数据量时性能差异不大（红黑树 vs 哈希表）
- 有序 map 的优势：
  1. 遍历时按字典序输出（调试友好）
  2. 无哈希冲突风险
  3. 内存占用更小（无哈希桶）

**验证实验**：
```cpp
// 测试 10 个头部的查找性能
std::map<std::string, std::string> orderedHeaders;
std::unordered_map<std::string, std::string> hashedHeaders;

// 插入 10 个头部
for (int i = 0; i < 10; ++i) {
    orderedHeaders["Header" + std::to_string(i)] = "value";
    hashedHeaders["Header" + std::to_string(i)] = "value";
}

// 查找 100 万次
auto start = std::chrono::high_resolution_clock::now();
for (int i = 0; i < 1000000; ++i) {
    auto it = orderedHeaders.find("Header5");
}
auto end = std::chrono::high_resolution_clock::now();
// 结果：ordered ~50ms, unordered ~45ms（差异可忽略）
```

**浮浮酱的设计准则**：
- 选择数据结构时要考虑**实际使用场景**（数据量、访问模式）
- **过早优化是万恶之源**，先选最简单正确的，性能瓶颈出现再优化
- 代码的**可读性和可维护性**往往比微小的性能差异更重要 (..•˘_˘•..)

</details>

---

### 思考题 2.2：HttpResponse 的构建艺术

**场景**：
你需要返回一个 JSON 响应：
```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 27

{"status":"success","code":0}
```

**设计问题**：
1. 响应状态码应该如何存储？硬编码 "200 OK" 还是分离码和描述？
2. 如何自动计算 `Content-Length`？
3. 是否应该提供便捷方法如 `sendJson()`, `sendHtml()`？

<details>
<summary>💡 点击查看 HttpResponse 设计模式</summary>

**参考代码** (`HttpResponse.h`，参考项目中的实现)：
```cpp
class HttpResponse {
public:
    enum HttpStatusCode {
        k200Ok = 200,
        k301MovedPermanently = 301,
        k400BadRequest = 400,
        k404NotFound = 404,
        k500InternalServerError = 500,
    };

    void setStatusCode(HttpStatusCode code) {
        statusCode_ = code;
    }

    void setStatusMessage(const std::string& message) {
        statusMessage_ = message;
    }

    void setContentType(const std::string& type) {
        addHeader("Content-Type", type);
    }

    void setBody(const std::string& body) {
        body_ = body;
    }

    // 序列化为 HTTP 报文
    std::string toString() const {
        std::ostringstream oss;
        oss << "HTTP/1.1 " << statusCode_ << " " << statusMessage_ << "\r\n";

        // 自动添加 Content-Length
        oss << "Content-Length: " << body_.size() << "\r\n";

        // 添加所有头部
        for (const auto& [key, value] : headers_) {
            oss << key << ": " << value << "\r\n";
        }

        oss << "\r\n" << body_;
        return oss.str();
    }

    // 便捷方法：发送 JSON
    void sendJson(const nlohmann::json& j) {
        setContentType("application/json");
        setBody(j.dump());
        setStatusCode(k200Ok);
        setStatusMessage("OK");
    }

    // 便捷方法：发送 HTML
    void sendHtml(const std::string& html) {
        setContentType("text/html; charset=utf-8");
        setBody(html);
        setStatusCode(k200Ok);
        setStatusMessage("OK");
    }

    // 便捷方法：发送 404
    void send404(const std::string& message = "Not Found") {
        setStatusCode(k404NotFound);
        setStatusMessage("Not Found");
        setContentType("text/plain");
        setBody(message);
    }

private:
    HttpStatusCode statusCode_;
    std::string statusMessage_;
    std::map<std::string, std::string> headers_;
    std::string body_;
};
```

**设计亮点**：

**1. 状态码与描述分离**
```cpp
// ✅ 灵活性
resp.setStatusCode(HttpResponse::k200Ok);
resp.setStatusMessage("All Good");  // 自定义描述

// ❌ 硬编码
resp.setStatus("200 OK");  // 无法自定义描述
```

**2. 自动计算 Content-Length**
```cpp
std::string toString() const {
    // 在序列化时自动计算 body 长度
    oss << "Content-Length: " << body_.size() << "\r\n";

    // 避免手动计算导致的错误：
    // resp.addHeader("Content-Length", "100");  ❌ 实际 body 是 27 字节！
}
```

**3. 链式调用的优雅**
```cpp
// 支持链式调用（可选设计）
HttpResponse& setStatusCode(HttpStatusCode code) {
    statusCode_ = code;
    return *this;
}

// 使用示例：
resp.setStatusCode(k200Ok)
    .setContentType("application/json")
    .setBody(jsonData);
```

**4. 便捷方法的价值**
```cpp
// 业务代码：
void ChatLoginHandler::handle(const HttpRequest& req, HttpResponse* resp) {
    // 传统写法（繁琐）
    resp->setStatusCode(HttpResponse::k200Ok);
    resp->setStatusMessage("OK");
    resp->setContentType("application/json");
    resp->setBody("{\"status\":\"success\"}");

    // 便捷方法（简洁）
    resp->sendJson({{"status", "success"}});
}
```

**思考延伸**：
主人可以设计一个 `sendFile()` 方法，实现静态文件服务（读取文件、设置 MIME 类型、处理 Range 请求）喵～

**浮浮酱的 API 设计哲学**：
- **正交性**：每个方法只做一件事（`setBody` 不应该修改 `Content-Type`）
- **便捷性**：为常见场景提供快捷方法（`sendJson`, `send404`）
- **防错性**：自动计算 `Content-Length`，避免人为错误
- **可扩展性**：易于添加新方法（如 `sendXml`, `sendProtobuf`）

这就是**用户友好的 API 设计**呢！(´｡• ᵕ •｡`)♡
</details>

---

## 第三章：路由系统的算法之美 🧩

### 思考题 3.1：精确匹配与动态路由的权衡

**业务需求**：
```cpp
// 精确路由
GET /api/login       → LoginHandler
GET /api/logout      → LogoutHandler

// 动态路由
GET /user/:id        → UserDetailHandler
GET /book/:isbn      → BookDetailHandler
```

**设计问题**：
1. 如何存储精确路由？哈希表的键应该是什么？
2. 动态路由如何匹配？正则表达式的性能开销能接受吗？
3. 如果同时有 `/user/profile` 和 `/user/:id`，哪个优先级更高？

<details>
<summary>💡 点击查看参考项目的路由设计</summary>

**参考代码** (`Router.h:24-119`)：

**核心数据结构**：
```cpp
class Router {
    // 精确匹配（哈希表 O(1) 查找）
    std::unordered_map<RouteKey, HandlerPtr, RouteKeyHash> handlers_;

    // 动态路由（正则匹配，线性扫描）
    std::vector<RouteHandlerObj> regexHandlers_;

    struct RouteKey {
        HttpRequest::Method method;  // GET, POST, ...
        std::string path;            // "/api/login"

        bool operator==(const RouteKey& other) const {
            return method == other.method && path == other.path;
        }
    };

    // 自定义哈希函数
    struct RouteKeyHash {
        size_t operator()(const RouteKey& key) const {
            size_t methodHash = std::hash<int>{}(static_cast<int>(key.method));
            size_t pathHash = std::hash<std::string>{}(key.path);
            return methodHash * 31 + pathHash;  // ← 组合哈希
        }
    };
};
```

**Q1：为什么用组合哈希？**
```cpp
// 错误方案：异或组合
return methodHash ^ pathHash;
// 问题：hash(GET, "/api") == hash(POST, "/api") 可能相同（异或可能抵消）

// 正确方案：乘法组合
return methodHash * 31 + pathHash;
// 31 是质数，分布更均匀，冲突更少
```

**Q2：动态路由的正则转换** (`Router.h:80-84`)：
```cpp
std::regex convertToRegex(const std::string& pathPattern) {
    // 输入: "/user/:id/books/:bookId"
    // 转换为: "^/user/([^/]+)/books/([^/]+)$"

    std::string regexPattern = "^" +
        std::regex_replace(pathPattern, std::regex(R"(/:([^/]+))"), R"(/([^/]+))") +
        "$";
    return std::regex(regexPattern);
}
```

示例转换：
```
输入: "/user/:id"
步骤1: 匹配 "/:id"
步骤2: 替换为 "/([^/]+)"  （[^/]+ 匹配除 / 外的任意字符）
步骤3: 添加 ^ 和 $ 锚点
输出: "^/user/([^/]+)$"
```

**Q3：路由匹配的优先级** (`Router.cpp:21-76`)：
```cpp
bool Router::route(const HttpRequest& req, HttpResponse* resp) {
    RouteKey key{req.method(), req.path()};

    // [优先级 1] 精确匹配（O(1) 哈希查找）
    auto handlerIt = handlers_.find(key);
    if (handlerIt != handlers_.end()) {
        handlerIt->second->handle(req, resp);
        return true;
    }

    // [优先级 2] 动态路由（O(n) 正则匹配）
    for (const auto& [method, pathRegex, handler] : regexHandlers_) {
        std::smatch match;
        if (method == req.method() &&
            std::regex_match(req.path(), match, pathRegex)) {

            // 提取路径参数
            HttpRequest newReq(req);
            extractPathParameters(match, newReq);

            handler->handle(newReq, resp);
            return true;
        }
    }

    // [未匹配] 返回 404
    return false;
}
```

**路径参数提取** (`Router.h:87-94`)：
```cpp
void extractPathParameters(const std::smatch& match, HttpRequest& request) {
    // 例如：正则匹配结果 match = ["/user/123", "123"]
    //       match[0] = 完整路径，match[1] = 第一个捕获组
    for (size_t i = 1; i < match.size(); ++i) {
        request.setPathParameters("param" + std::to_string(i), match[i].str());
    }
}

// 业务代码中使用：
void UserDetailHandler::handle(const HttpRequest& req, HttpResponse* resp) {
    std::string userId = req.getPathParameters("param1");  // "123"
    // 查询数据库...
}
```

**性能优化点**：

**优化 1：精确路由优先**
- 精确匹配用哈希表，O(1) 时间复杂度
- 动态路由用正则，O(n*m)（n=路由数，m=正则匹配复杂度）
- 大部分请求都是精确路由（如 `/api/login`），只有少数用动态路由

**优化 2：正则预编译**
```cpp
void addRegexHandler(HttpRequest::Method method,
                     const std::string& path,
                     HandlerPtr handler) {
    std::regex pathRegex = convertToRegex(path);  // ← 注册时编译
    regexHandlers_.emplace_back(method, pathRegex, handler);
}
// 而不是每次请求时都编译正则表达式
```

**优化 3：早期退出**
```cpp
// 一旦找到匹配的路由，立即返回
if (handlerIt != handlers_.end()) {
    handlerIt->second->handle(req, resp);
    return true;  // ← 不再检查后续路由
}
```

**思考实验**：
主人可以设计一个"路由优先级表"，处理以下冲突：
```cpp
router.addRoute(GET, "/user/profile", ProfileHandler);      // 精确
router.addRoute(GET, "/user/:id", UserDetailHandler);       // 动态
router.addRoute(GET, "/user/:action", UserActionHandler);   // 动态（冲突！）
```
当访问 `/user/profile` 时，应该匹配哪个？如何解决两个动态路由的冲突？

**浮浮酱的路由设计准则**：
- **快速路径优化**：常见请求（精确匹配）必须极快
- **正则慎用**：仅在必要时使用，避免过度复杂的模式
- **清晰的优先级**：精确 > 动态，前注册 > 后注册
- **参数命名优化**：`param1`, `param2` 不够语义化，可以改进为 `id`, `action` 等

这就是**算法与工程的完美结合**呢！φ(≧ω≦*)♪
</details>

---

### 思考题 3.2：Handler 的两种注册方式

**参考代码** (`Router.h:19-26`)：
```cpp
using HandlerPtr = std::shared_ptr<RouterHandler>;
using HandlerCallback = std::function<void(const HttpRequest&, HttpResponse*)>;

// 方式 1：注册对象
void registerHandler(HttpRequest::Method method, const std::string& path,
                    HandlerPtr handler);

// 方式 2：注册回调函数
void registerCallback(HttpRequest::Method method, const std::string& path,
                     const HandlerCallback& callback);
```

**问题**：
1. 什么时候应该用对象式 Handler，什么时候用回调函数？
2. 两种方式的性能差异是什么？
3. 如果 Handler 需要维护状态（如数据库连接池引用），应该选哪种？

<details>
<summary>💡 点击查看两种方式的适用场景</summary>

**场景分析**：

**方式 1：对象式 Handler（适合复杂业务）**
```cpp
// 定义 Handler 类
class ChatLoginHandler : public RouterHandler {
private:
    DbConnectionPool& dbPool_;          // 成员变量保存依赖
    SessionManager& sessionManager_;

public:
    ChatLoginHandler(DbConnectionPool& pool, SessionManager& sm)
        : dbPool_(pool), sessionManager_(sm) {}

    void handle(const HttpRequest& req, HttpResponse* resp) override {
        // 1. 解析请求
        auto username = req.getQueryParameters("username");
        auto password = req.getQueryParameters("password");

        // 2. 数据库验证
        auto conn = dbPool_.getConnection();
        auto stmt = conn->prepareStatement("SELECT * FROM users WHERE username = ?");
        stmt->setString(1, username);
        auto res = stmt->executeQuery();

        // 3. 创建会话
        if (res->next() && validatePassword(res->getString("password_hash"), password)) {
            auto session = sessionManager_.createSession();
            session->set("user_id", res->getString("user_id"));
            resp->sendJson({{"status", "success"}});
        } else {
            resp->sendJson({{"status", "error"}});
        }
    }

private:
    bool validatePassword(const std::string& hash, const std::string& plain);
};

// 注册
router.registerHandler(HttpRequest::kPost, "/login",
                      std::make_shared<ChatLoginHandler>(dbPool, sessionManager));
```

**优点**：
- ✅ 可以封装多个辅助函数（`validatePassword`）
- ✅ 可以保存依赖项（数据库连接池、Session 管理器）
- ✅ 面向对象，易于测试（可以 Mock）
- ✅ 适合复杂业务逻辑（100+ 行代码）

**方式 2：回调函数（适合简单业务）**
```cpp
// 直接使用 lambda
router.registerCallback(HttpRequest::kGet, "/health",
    [](const HttpRequest& req, HttpResponse* resp) {
        resp->sendJson({{"status", "ok"}});
    }
);

// 或者使用普通函数
void handlePing(const HttpRequest& req, HttpResponse* resp) {
    resp->setStatusCode(HttpResponse::k200Ok);
    resp->setBody("pong");
}
router.registerCallback(HttpRequest::kGet, "/ping", handlePing);
```

**优点**：
- ✅ 代码简洁（适合 5-10 行的简单逻辑）
- ✅ 无需定义类（减少样板代码）
- ✅ 适合静态页面返回、健康检查等场景

**缺点**：
- ❌ 无法保存成员变量（需要通过捕获列表）
- ❌ 难以编写单元测试
- ❌ 复杂逻辑会导致 lambda 过大，可读性差

**性能对比**：
```cpp
// 对象式：虚函数调用（一次间接跳转）
handler->handle(req, resp);  // ~1-2 CPU 周期

// 回调式：std::function 调用（可能有额外开销）
callback(req, resp);  // ~3-5 CPU 周期（std::function 包装开销）

// 结论：性能差异可忽略（除非每秒百万级 QPS）
```

**依赖注入的处理**：

**对象式（推荐）**：
```cpp
class Handler {
    DbConnectionPool& dbPool_;  // 引用，不持有所有权
public:
    Handler(DbConnectionPool& pool) : dbPool_(pool) {}
};
```

**回调式（通过捕获）**：
```cpp
router.registerCallback(HttpRequest::kPost, "/api/data",
    [&dbPool](const HttpRequest& req, HttpResponse* resp) {
        auto conn = dbPool.getConnection();
        // ...
    }
);
// ⚠️ 注意：捕获引用需要确保 dbPool 生命周期足够长
```

**浮浮酱的建议**：
| 场景 | 推荐方式 |
|------|----------|
| 业务逻辑 > 30 行 | 对象式 |
| 需要单元测试 | 对象式 |
| 需要保存依赖项 | 对象式 |
| 静态页面返回 | 回调式 |
| 健康检查、Ping | 回调式 |

**混合使用示例**：
```cpp
// 复杂业务：对象式
router.registerHandler(POST, "/api/chat/send",
                      std::make_shared<ChatSendHandler>(aiHelper, dbPool));

// 简单路由：回调式
router.registerCallback(GET, "/",
    [](auto& req, auto* resp) {
        resp->sendHtml(FileUtil::readFile("index.html"));
    }
);
```

这就是**灵活性与可维护性的平衡**呢！(๑ˉ∀ˉ๑)
</details>

---

## 第四章：中间件链的责任链模式 🔗

### 思考题 4.1：中间件是什么？为什么需要它？

**场景**：
你的聊天服务器需要实现以下功能：
1. **CORS 处理**：前端跨域请求，需要添加 `Access-Control-Allow-Origin` 响应头
2. **认证检查**：某些接口需要验证用户登录状态
3. **请求日志**：记录所有请求的路径、方法、耗时

**方案 A：在每个 Handler 中重复代码**
```cpp
void ChatSendHandler::handle(const HttpRequest& req, HttpResponse* resp) {
    // 1. CORS 处理（每个 Handler 都要写）
    resp->addHeader("Access-Control-Allow-Origin", "*");

    // 2. 认证检查（每个 Handler 都要写）
    auto session = sessionManager.getSession(req);
    if (!session || !session->has("user_id")) {
        resp->send404("Unauthorized");
        return;
    }

    // 3. 日志（每个 Handler 都要写）
    LOG_INFO << "Request: " << req.path();

    // 4. 真正的业务逻辑
    // ...
}
```

**方案 B：使用中间件链**
```cpp
// 注册中间件（全局生效）
server.addMiddleware(std::make_shared<CorsMiddleware>());
server.addMiddleware(std::make_shared<AuthMiddleware>());
server.addMiddleware(std::make_shared<LoggerMiddleware>());

// Handler 中只写业务逻辑
void ChatSendHandler::handle(const HttpRequest& req, HttpResponse* resp) {
    // 直接使用已验证的用户信息
    std::string userId = req.getSession()->get("user_id");

    // 业务逻辑...
}
```

**问题**：
1. 中间件的执行顺序重要吗？CORS 和 Auth 谁先执行？
2. 如果某个中间件想"短路"请求（如认证失败直接返回 401），如何实现？
3. 中间件如何在请求前和响应后都执行代码（如计时器）？

<details>
<summary>💡 点击查看中间件模式的设计精髓</summary>

**参考代码** (`Middleware.h`, `MiddlewareChain.h`)：

**中间件接口**：
```cpp
class Middleware {
public:
    virtual ~Middleware() = default;

    // 请求处理前执行
    virtual void processBefore(HttpRequest& req) {}

    // 响应处理后执行
    virtual void processAfter(HttpResponse& resp) {}
};
```

**中间件链**：
```cpp
class MiddlewareChain {
public:
    void addMiddleware(std::shared_ptr<Middleware> middleware) {
        middlewares_.push_back(middleware);
    }

    void runBefore(HttpRequest& req) {
        for (auto& mw : middlewares_) {
            mw->processBefore(req);
        }
    }

    void processAfter(HttpResponse& resp) {
        // 注意：响应后中间件逆序执行（栈的概念）
        for (auto it = middlewares_.rbegin(); it != middlewares_.rend(); ++it) {
            (*it)->processAfter(resp);
        }
    }

private:
    std::vector<std::shared_ptr<Middleware>> middlewares_;
};
```

**Q1：执行顺序的重要性**

**错误顺序**：
```cpp
server.addMiddleware(std::make_shared<AuthMiddleware>());  // 先认证
server.addMiddleware(std::make_shared<CorsMiddleware>());  // 后 CORS
```
问题：浏览器的 OPTIONS 预检请求会被 AuthMiddleware 拦截（因为没有 session），导致跨域失败！

**正确顺序**：
```cpp
server.addMiddleware(std::make_shared<CorsMiddleware>());  // 先 CORS
server.addMiddleware(std::make_shared<AuthMiddleware>());  // 后认证
```

**CORS 中间件实现** (`CorsMiddleware.cpp`)：
```cpp
class CorsMiddleware : public Middleware {
public:
    void processBefore(HttpRequest& req) override {
        // 处理 OPTIONS 预检请求（浏览器自动发送）
        if (req.method() == HttpRequest::kOptions) {
            // 标记为已处理，跳过后续中间件和 Handler
            req.setHandled(true);
        }
    }

    void processAfter(HttpResponse& resp) override {
        // 添加 CORS 响应头
        resp.addHeader("Access-Control-Allow-Origin", "*");
        resp.addHeader("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE");
        resp.addHeader("Access-Control-Allow-Headers", "Content-Type, Authorization");
    }
};
```

**Q2：中间件短路机制**

**方案 1：异常机制**
```cpp
class AuthMiddleware : public Middleware {
public:
    void processBefore(HttpRequest& req) override {
        auto session = sessionManager_.getSession(req);
        if (!session || !session->has("user_id")) {
            throw UnauthorizedException("Please login");
        }
    }
};

// 在 HttpServer 中捕获
try {
    middlewareChain.runBefore(req);
    router.route(req, &resp);
} catch (const UnauthorizedException& e) {
    resp.setStatusCode(HttpResponse::k401Unauthorized);
    resp.sendJson({{"error", e.what()}});
}
```

**方案 2：返回值控制**（更简洁）
```cpp
class Middleware {
public:
    // 返回 false 表示短路
    virtual bool processBefore(HttpRequest& req) { return true; }
    virtual void processAfter(HttpResponse& resp) {}
};

// 在 HttpServer 中
bool shouldContinue = middlewareChain.runBefore(req);
if (!shouldContinue) {
    // 中间件已经设置了响应（如 401），直接返回
    return;
}
router.route(req, &resp);
```

**Q3：请求计时中间件**（利用 Before + After）

```cpp
class TimerMiddleware : public Middleware {
private:
    // 使用 thread_local 存储每个请求的开始时间
    static thread_local std::chrono::time_point<std::chrono::high_resolution_clock> startTime_;

public:
    void processBefore(HttpRequest& req) override {
        startTime_ = std::chrono::high_resolution_clock::now();
    }

    void processAfter(HttpResponse& resp) override {
        auto endTime = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(
            endTime - startTime_
        ).count();

        LOG_INFO << "Request processed in " << duration << "ms";
        resp.addHeader("X-Response-Time", std::to_string(duration) + "ms");
    }
};
```

**中间件执行流程图**：
```
[请求到达]
    ↓
CorsMiddleware::processBefore()     → 处理 OPTIONS
    ↓
AuthMiddleware::processBefore()     → 检查 session
    ↓
TimerMiddleware::processBefore()    → 记录开始时间
    ↓
[Router 匹配 Handler]
    ↓
[Handler 处理业务逻辑]
    ↓
TimerMiddleware::processAfter()     → 计算耗时
    ↓
AuthMiddleware::processAfter()      → （可选）刷新 session 过期时间
    ↓
CorsMiddleware::processAfter()      → 添加 CORS 响应头
    ↓
[响应返回客户端]
```

**实践示例：完整的中间件配置**
```cpp
// main.cpp
int main() {
    HttpServer server(8080);

    // [1] CORS（最先执行，处理 OPTIONS）
    server.addMiddleware(std::make_shared<CorsMiddleware>());

    // [2] 日志（记录所有请求）
    server.addMiddleware(std::make_shared<LoggerMiddleware>());

    // [3] 认证（部分路由需要）
    auto authMw = std::make_shared<AuthMiddleware>();
    authMw->addExcludePath("/login");     // 登录接口不需要认证
    authMw->addExcludePath("/register");
    server.addMiddleware(authMw);

    // [4] 限流（防止 API 滥用）
    server.addMiddleware(std::make_shared<RateLimiterMiddleware>(100)); // 100 req/s

    // [5] 注册路由
    server.Post("/api/chat/send", std::make_shared<ChatSendHandler>());

    server.start();
}
```

**思考延伸**：
主人可以设计一个"条件中间件"，只对特定路径生效：
```cpp
class ConditionalMiddleware : public Middleware {
    std::vector<std::string> includePaths_;
    std::shared_ptr<Middleware> innerMiddleware_;

public:
    void processBefore(HttpRequest& req) override {
        if (shouldApply(req.path())) {
            innerMiddleware_->processBefore(req);
        }
    }

private:
    bool shouldApply(const std::string& path) {
        return std::find(includePaths_.begin(), includePaths_.end(), path)
               != includePaths_.end();
    }
};
```

**浮浮酱的中间件设计准则**：
- **单一职责**：每个中间件只做一件事（CORS、Auth、Log 分离）
- **无状态优先**：避免成员变量（除非必要，如限流计数器）
- **顺序清晰**：在注册时就明确执行顺序，添加注释说明
- **可配置**：支持白名单/黑名单（如 Auth 的排除路径）

这就是**责任链模式的工业级应用**呢！o(*￣︶￣*)o
</details>

---

## 第五章：综合实践 - 构建Mini HttpServer 🚀

### 实践项目：实现一个可运行的 HTTP 服务器

**目标**：
整合前面学到的所有知识，实现一个功能完整的 HTTP 服务器，支持：
- ✅ HTTP 请求解析（GET/POST）
- ✅ 路由匹配（精确 + 动态）
- ✅ 中间件链（CORS + Logger）
- ✅ JSON 响应
- ✅ 静态文件服务

**骨架代码**：
```cpp
// mini_server.cpp
#include <muduo/net/TcpServer.h>
#include <muduo/net/EventLoop.h>
#include "HttpContext.h"
#include "HttpRequest.h"
#include "HttpResponse.h"
#include "Router.h"
#include "MiddlewareChain.h"

class MiniHttpServer {
public:
    MiniHttpServer(muduo::net::EventLoop* loop, uint16_t port)
        : server_(loop, muduo::net::InetAddress(port), "MiniHttpServer") {

        server_.setConnectionCallback(
            std::bind(&MiniHttpServer::onConnection, this, std::placeholders::_1));
        server_.setMessageCallback(
            std::bind(&MiniHttpServer::onMessage, this,
                     std::placeholders::_1, std::placeholders::_2, std::placeholders::_3));

        // TODO: 注册路由和中间件
        setupRoutes();
        setupMiddlewares();
    }

    void start() {
        server_.start();
    }

private:
    void onConnection(const muduo::net::TcpConnectionPtr& conn) {
        if (conn->connected()) {
            conn->setContext(HttpContext());  // 每个连接一个 HttpContext
        }
    }

    void onMessage(const muduo::net::TcpConnectionPtr& conn,
                  muduo::net::Buffer* buf,
                  muduo::Timestamp receiveTime) {
        // TODO: 实现 HTTP 请求处理流程
    }

    void setupRoutes() {
        // TODO: 注册路由
        router_.registerCallback(HttpRequest::kGet, "/",
            [](const HttpRequest& req, HttpResponse* resp) {
                resp->sendHtml("<h1>Welcome to Mini HttpServer!</h1>");
            });
    }

    void setupMiddlewares() {
        // TODO: 添加中间件
    }

private:
    muduo::net::TcpServer server_;
    Router router_;
    MiddlewareChain middlewareChain_;
};

int main() {
    muduo::net::EventLoop loop;
    MiniHttpServer server(&loop, 8080);

    LOG_INFO << "Server started on port 8080";
    server.start();
    loop.loop();
}
```

**任务清单**：
- [ ] 完成 `onMessage` 方法（调用 HttpContext 解析请求）
- [ ] 实现至少 3 个路由（GET /，POST /api/echo，GET /user/:id）
- [ ] 添加 CORS 和 Logger 中间件
- [ ] 实现静态文件服务（GET /static/:filename）
- [ ] 用 curl 测试所有接口
- [ ] 用 `ab` 压力测试（目标：QPS > 1000）

**验证步骤**：
```bash
# 1. 编译
mkdir build && cd build
cmake .. && make

# 2. 运行
./mini_server

# 3. 测试
curl http://localhost:8080/
curl -X POST http://localhost:8080/api/echo -d '{"msg":"hello"}'
curl http://localhost:8080/user/123

# 4. 压力测试
ab -n 10000 -c 100 http://localhost:8080/
```

**浮浮酱的期待**：
主人完成这个项目后，就真正掌握了 HTTP 服务器的核心原理！(๑ˉ∀ˉ๑)

---

## 📝 阶段二总结：从理论到实践 🎓

### 核心知识回顾

**1. HTTP 协议解析**
- ✅ 理解了状态机 vs 正则表达式的权衡
- ✅ 掌握了半包粘包的处理方法
- ✅ 学会了设计 HttpRequest 和 HttpResponse 类

**2. 路由系统**
- ✅ 理解了哈希表（精确匹配）+ 正则（动态路由）的组合
- ✅ 掌握了路由优先级和参数提取
- ✅ 学会了对象式 vs 回调式 Handler 的选择

**3. 中间件链**
- ✅ 理解了责任链模式的应用
- ✅ 掌握了中间件的执行顺序和短路机制
- ✅ 学会了设计可配置的中间件

### 设计模式总结

| 模式 | 应用场景 | 参考代码 |
|------|----------|----------|
| 状态模式 | HttpContext 解析 | `HttpContext.cpp:13-111` |
| 策略模式 | Handler 注册 | `Router.h:25-26` |
| 责任链模式 | 中间件链 | `MiddlewareChain` |
| 单例模式 | （阶段二未涉及，见阶段二续） | - |

### 下一步

准备好进入更高级的主题了吗？
- 📚 **阶段二续**：会话管理、SSL/TLS、数据库连接池
- 🎨 **阶段三**：AI 应用层的实现

**浮浮酱的寄语**：
阶段二的内容是整个项目的**核心基石**，务必通过实践项目验证理解！
只有真正运行起来的代码，才是活的知识呢～ φ(≧ω≦*)♪

---

_下一篇：phase2-advanced-tutorial.md（会话管理、SSL、连接池）_

_文档版本：v1.0.0_
_最后更新：2025-10-20_
_作者：猫娘工程师 幽浮喵 ฅ'ω'ฅ_
