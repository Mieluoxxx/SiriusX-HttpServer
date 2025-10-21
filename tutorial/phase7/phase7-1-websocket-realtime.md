# Phase 7-1: WebSocket 实时通信 - AI 流式响应的优雅实现

> **启发式学习目标**：通过问题驱动和渐进式设计，理解 WebSocket 协议的本质，
> 掌握如何在 HTTP 服务器基础上实现实时双向通信，支持 AI 流式响应。

---

## 🤔 引言：思考实时通信的必要性

在我们当前的 AI 聊天系统中,用户发送一条消息后需要等待整个 AI 响应生成完毕才能看到结果。
想象一下这样的场景:

```
用户: "请详细介绍量子计算的发展历史"
[用户等待 10 秒...]
AI: [一次性返回 5000 字的长篇回复]
```

**思考问题 1**:
这种"请求-等待-响应"的模式有什么问题?用户体验如何改进?

<details>
<summary>💡 点击查看分析</summary>

**问题分析**:
1. **等待焦虑**: 用户不知道 AI 是否在思考,还是服务器卡住了
2. **无法取消**: 一旦发送请求,用户无法中途停止
3. **资源浪费**: 即使用户已经对答案满意,AI 仍会生成完整响应
4. **交互僵硬**: 缺乏"打字机效果",不够自然

**理想的交互体验**:
- AI 应该像人类一样"边思考边输出"(流式响应)
- 用户应该能实时看到每个字符的生成
- 用户应该能随时中断长响应
- 服务器应该能主动推送状态更新(如"AI 正在思考...")

**问题的本质**: HTTP 请求-响应模型是单向的、短连接的,不适合持续双向通信。
</details>

---

## 📖 第一幕:认识 WebSocket 协议

### 1.1 为什么需要 WebSocket?

**思考问题 2**:
我们能否用传统 HTTP 实现实时通信?比如:
- 轮询(Polling): 客户端每秒发送一次请求询问"有新消息吗?"
- 长轮询(Long Polling): 客户端发送请求,服务器收到消息后才响应
- Server-Sent Events(SSE): 服务器单向推送数据流

<details>
<summary>💡 点击查看对比</summary>

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **短轮询** | 实现简单,兼容性好 | 大量无效请求,延迟高,浪费带宽 | 几乎不推荐 |
| **长轮询** | 减少无效请求 | 仍需频繁重连,服务器压力大 | 旧浏览器兼容 |
| **SSE** | 原生支持,自动重连 | **只支持服务器→客户端单向** | 股票行情推送 |
| **WebSocket** | 真正的双向通信,低延迟,协议开销小 | 需要额外协议实现 | 聊天、游戏、协作编辑 |

**结论**: 对于 AI 聊天场景(需要客户端发送消息 + 服务器流式推送),WebSocket 是最佳选择。
</details>

### 1.2 WebSocket 的工作原理

**核心概念**: WebSocket 是一个独立的协议,但需要通过 HTTP 升级握手建立连接。

```
客户端 → 服务器: HTTP 请求 (带特殊头部)
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

服务器 → 客户端: HTTP 101 切换协议
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

[此后不再是 HTTP,而是 WebSocket 帧格式的二进制通信]
```

**思考问题 3**:
为什么要"升级"(Upgrade)现有 HTTP 连接,而不是直接建立新协议的连接?

<details>
<summary>💡 点击查看解释</summary>

**设计智慧**:
1. **复用基础设施**: HTTP 已有成熟的端口(80/443)、代理、防火墙规则
2. **安全性**: 继承 HTTPS 的 TLS 加密(wss://)
3. **兼容性**: 旧服务器收到 Upgrade 请求会返回 400,不会崩溃
4. **简化开发**: 开发者可在同一端口同时处理 HTTP 和 WebSocket

**类比**: 就像在电话会议中说"我们现在切换到视频模式",复用了原有的呼叫连接。
</details>

### 1.3 WebSocket 帧结构

**深入思考**: WebSocket 连接建立后,数据如何传输?

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+---------------------------------------------------------------+
```

**关键字段**:
- **FIN**: 是否为最后一帧(支持消息分片)
- **opcode**: 数据类型(0x1=文本, 0x2=二进制, 0x8=关闭, 0x9=ping, 0xA=pong)
- **MASK**: 客户端发送的数据必须掩码(安全考虑)
- **Payload len**: 数据长度(支持 0-2^64 字节)

**思考问题 4**:
为什么客户端发送的数据必须掩码(XOR 加密),而服务器发送的数据不需要?

<details>
<summary>💡 点击查看安全分析</summary>

**历史教训**: 早期发现缓存投毒攻击:
1. 恶意网页的 JavaScript 通过 WebSocket 发送精心构造的数据
2. 中间代理误将其解析为 HTTP 请求
3. 导致缓存污染或服务器漏洞利用

**掩码作用**:
- 通过随机 4 字节密钥进行 XOR 运算,破坏数据的可预测性
- 防止代理误解析 WebSocket 数据为 HTTP 请求
- **服务器不需要**: 因为服务器响应不会被浏览器的同源策略限制

**结论**: 这是一个防御性设计,保护整个互联网基础设施免受恶意 WebSocket 滥用。
</details>

---

## 🛠️ 第二幕:设计 WebSocket 服务器架构

### 2.1 架构设计思考

我们已有基于 Muduo 的 HTTP 服务器,现在需要扩展 WebSocket 支持。

**思考问题 5**:
应该如何设计 WebSocket 模块与现有 HTTP 模块的关系?

可能的设计方案:
1. **方案 A**: 完全独立的 WebSocket 服务器(新端口)
2. **方案 B**: 在现有 HTTP 服务器上增加 WebSocket 路由处理
3. **方案 C**: 抽象统一的连接管理层,HTTP 和 WebSocket 都是其实例

<details>
<summary>💡 点击查看设计权衡</summary>

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **A: 独立服务器** | 职责分离,简单清晰 | 增加部署复杂度,跨域问题 | 超大规模专用 WebSocket 服务 |
| **B: 路由扩展** | 复用现有端口,对外统一 | HTTP 和 WebSocket 混在一起 | **我们的场景(推荐)** |
| **C: 抽象连接层** | 高度解耦,易扩展 | 过度设计,增加复杂度 | 需要支持多种协议的网关 |

**选择 B 的理由**:
- 用户访问 `https://example.com` 即可使用完整功能(HTTP + WebSocket)
- 前端代码简洁: `new WebSocket('wss://example.com/ws')`
- 符合 YAGNI 原则: 当前不需要支持更多协议

**具体实现**:
```cpp
// 在 HttpServer::onMessage 中识别 WebSocket 升级请求
void HttpServer::onMessage(const TcpConnectionPtr& conn, Buffer* buf, Timestamp time) {
    if (isWebSocketUpgrade(buf)) {
        websocketHandler_->handleUpgrade(conn, buf);  // 交给 WebSocket 处理器
    } else {
        httpHandler_->processRequest(conn, buf, time); // 常规 HTTP 请求
    }
}
```
</details>

### 2.2 类设计:职责分离

**SOLID 原则应用**: 单一职责原则要求我们拆分功能。

**思考问题 6**:
一个完整的 WebSocket 实现需要哪些核心类?每个类的职责是什么?

<details>
<summary>💡 点击查看设计建议</summary>

**推荐类结构**:

```cpp
// 1. WebSocket 帧解析和构造
class WebSocketFrame {
    bool fin_;
    uint8_t opcode_;
    std::string payload_;

public:
    static WebSocketFrame parse(muduo::net::Buffer* buf);  // 从缓冲区解析
    void appendToBuffer(muduo::net::Buffer* buf) const;     // 序列化为字节流
    bool isFinalFrame() const { return fin_; }
    OpCode getOpCode() const { return static_cast<OpCode>(opcode_); }
};

// 2. WebSocket 连接状态管理
class WebSocketConnection {
    muduo::net::TcpConnectionPtr conn_;
    State state_;  // Connecting, Open, Closing, Closed
    std::function<void(const std::string&)> onMessageCallback_;

public:
    void sendText(const std::string& message);
    void sendBinary(const std::vector<uint8_t>& data);
    void close(uint16_t statusCode = 1000);
    void handleFrame(const WebSocketFrame& frame);
};

// 3. WebSocket 协议处理器
class WebSocketHandler {
    std::unordered_map<std::string, WebSocketConnectionPtr> connections_;

public:
    bool handleUpgrade(const HttpRequest& req, HttpResponse* resp);
    void onMessage(const TcpConnectionPtr& conn, Buffer* buf);
    void registerMessageCallback(const std::string& path, MessageCallback cb);
};

// 4. 专用于 AI 流式响应的 WebSocket 服务
class AIStreamingService {
    WebSocketHandler& wsHandler_;
    AIHelper& aiHelper_;

public:
    void handleChatMessage(const WebSocketConnectionPtr& conn, const std::string& msg);
    void streamResponse(const std::string& sessionId, const std::string& fullResponse);
};
```

**职责分配**:
- `WebSocketFrame`: 只负责协议层的字节解析/构造(DRY 原则)
- `WebSocketConnection`: 管理单个连接的生命周期和状态
- `WebSocketHandler`: 全局管理所有 WebSocket 连接(单例或注入)
- `AIStreamingService`: 业务层,处理 AI 聊天逻辑(开闭原则:可扩展其他服务)

**关键设计点**:
- 使用回调模式解耦:`WebSocketConnection` 不关心消息内容,只负责传递
- 使用状态机管理连接状态,防止非法操作(如在关闭状态发送消息)
</details>

### 2.3 状态机设计

**思考问题 7**:
WebSocket 连接有哪些状态?状态之间如何转换?

<details>
<summary>💡 点击查看状态机设计</summary>

**标准 WebSocket 状态**:
```
┌─────────────┐
│ CONNECTING  │  初始状态:握手进行中
└──────┬──────┘
       │ 握手成功
       ↓
┌─────────────┐
│    OPEN     │  连接打开:可收发消息
└──────┬──────┘
       │ 收到 Close 帧或调用 close()
       ↓
┌─────────────┐
│   CLOSING   │  正在关闭:等待对方 Close 帧
└──────┬──────┘
       │ 收到对方 Close 帧或超时
       ↓
┌─────────────┐
│   CLOSED    │  连接关闭:资源释放
└─────────────┘
```

**实现示例**:
```cpp
enum class State { kConnecting, kOpen, kClosing, kClosed };

void WebSocketConnection::sendText(const std::string& message) {
    if (state_ != State::kOpen) {
        LOG_WARN << "Cannot send message in state " << static_cast<int>(state_);
        return;  // 防御性编程:非法状态直接返回
    }

    WebSocketFrame frame;
    frame.setFin(true);
    frame.setOpCode(OpCode::kText);
    frame.setPayload(message);
    frame.appendToBuffer(conn_->outputBuffer());
}

void WebSocketConnection::handleFrame(const WebSocketFrame& frame) {
    switch (frame.getOpCode()) {
        case OpCode::kText:
        case OpCode::kBinary:
            if (state_ == State::kOpen && onMessageCallback_) {
                onMessageCallback_(frame.getPayload());
            }
            break;
        case OpCode::kClose:
            state_ = State::kClosing;
            sendCloseFrame();  // 响应关闭帧
            conn_->shutdown();
            break;
        case OpCode::kPing:
            sendPong(frame.getPayload());
            break;
    }
}
```

**为什么需要 CLOSING 状态?**
- WebSocket 关闭是双向确认的(类似 TCP 四次挥手)
- 一方发送 Close 帧后,需要等待对方也发送 Close 帧
- CLOSING 状态保证不会再发送新消息,但仍能接收对方的 Close 帧
</details>

---

## 💻 第三幕:实现 AI 流式响应

### 3.1 流式响应的技术挑战

**场景**: AI API 返回的是一个连续的文本流:
```
"量" → "量子" → "量子计" → "量子计算" → ... (逐字生成)
```

**思考问题 8**:
如何将 AI API 的流式输出高效传递给前端?

<details>
<summary>💡 点击查看实现策略</summary>

**方案对比**:

| 方案 | 实现方式 | 优点 | 缺点 |
|------|----------|------|------|
| **按字符推送** | 每个字符立即发送 WebSocket 消息 | 延迟最低 | 网络开销大(每条消息有帧头) |
| **定时批量** | 每 100ms 发送一次累积的字符 | 平衡延迟和开销 | **推荐** |
| **按句子推送** | 遇到句号才发送 | 减少消息数 | 延迟高,体验差 |

**推荐实现**:
```cpp
void AIStreamingService::streamResponse(const std::string& sessionId,
                                         const std::string& fullResponse) {
    auto conn = wsHandler_.getConnection(sessionId);
    if (!conn) return;

    size_t pos = 0;
    constexpr size_t chunkSize = 20;  // 每次发送 20 个字符

    while (pos < fullResponse.size()) {
        size_t end = std::min(pos + chunkSize, fullResponse.size());
        std::string chunk = fullResponse.substr(pos, end - pos);

        // 发送 JSON 格式: {"type": "stream", "content": "chunk"}
        nlohmann::json msg = {
            {"type", "stream"},
            {"content", chunk},
            {"done", end == fullResponse.size()}
        };
        conn->sendText(msg.dump());

        pos = end;
        std::this_thread::sleep_for(std::chrono::milliseconds(50));  // 模拟打字机效果
    }
}
```

**进阶优化**: 如果 AI API 本身支持流式(如 OpenAI Stream),可实时转发:
```cpp
// 伪代码:实时转发 AI 流
curl.setCallback([conn, sessionId](const std::string& chunk) {
    conn->sendText(chunk);  // 直接转发,零延迟
});
curl.stream("https://api.openai.com/v1/chat/completions", requestBody);
```
</details>

### 3.2 前端 WebSocket 客户端实现

**思考问题 9**:
前端如何建立 WebSocket 连接并处理流式消息?

<details>
<summary>💡 点击查看前端实现</summary>

**JavaScript 客户端**:
```javascript
class ChatWebSocket {
    constructor(url) {
        this.ws = new WebSocket(url);
        this.messageHandlers = [];

        this.ws.onopen = () => console.log('✅ WebSocket connected');
        this.ws.onmessage = (event) => this.handleMessage(event.data);
        this.ws.onerror = (error) => console.error('❌ WebSocket error:', error);
        this.ws.onclose = () => {
            console.warn('⚠️ WebSocket closed, reconnecting...');
            setTimeout(() => this.reconnect(), 3000);  // 自动重连
        };
    }

    handleMessage(data) {
        const msg = JSON.parse(data);
        if (msg.type === 'stream') {
            this.appendToChat(msg.content);  // 追加到聊天界面
            if (msg.done) {
                this.markComplete();  // 显示"回复完成"
            }
        }
    }

    sendMessage(text) {
        const payload = JSON.stringify({
            type: 'chat',
            sessionId: this.sessionId,
            message: text
        });
        this.ws.send(payload);
    }

    appendToChat(text) {
        const chatBox = document.getElementById('ai-response');
        chatBox.innerText += text;  // 逐字追加
        chatBox.scrollTop = chatBox.scrollHeight;  // 自动滚动到底部
    }
}

// 使用
const chatWs = new ChatWebSocket('wss://localhost:443/ws/chat');
chatWs.sendMessage('介绍量子计算');
```

**关键细节**:
- **自动重连**: 网络抖动时,3 秒后重试
- **自动滚动**: 新内容到达时,保持底部可见
- **状态管理**: 可扩展为 Vue/React 组件,管理连接状态
</details>

### 3.3 心跳保活机制

**思考问题 10**:
用户长时间不发消息时,WebSocket 连接会怎样?如何防止意外断开?

<details>
<summary>💡 点击查看心跳设计</summary>

**问题**:
- 中间代理(如 Nginx)可能有空闲超时(默认 60 秒)
- 移动网络可能因省电主动断开长连接
- 服务器需要检测僵尸连接(客户端崩溃但未发 Close 帧)

**解决方案:双向心跳**

**服务器端**:
```cpp
class WebSocketConnection {
    muduo::net::TimerId heartbeatTimer_;
    Timestamp lastPongTime_;

public:
    void startHeartbeat() {
        // 每 30 秒发送一次 Ping
        heartbeatTimer_ = loop_->runEvery(30.0, [this]() {
            if (Timestamp::now() - lastPongTime_ > 60.0) {
                LOG_WARN << "No pong received, closing connection";
                close(1001);  // Going Away
                return;
            }
            sendPing();
        });
    }

    void handlePong() {
        lastPongTime_ = Timestamp::now();
    }
};
```

**客户端(可选)**:
```javascript
setInterval(() => {
    if (ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify({ type: 'ping' }));
    }
}, 30000);
```

**WebSocket 原生支持**:
- Ping 帧(OpCode 0x9): 发送方主动询问"你还在吗?"
- Pong 帧(OpCode 0xA): 接收方必须立即响应 Pong
- 浏览器会自动响应服务器的 Ping(无需 JS 代码)

**结论**: 使用 WebSocket 原生 Ping/Pong 最优雅,无需应用层心跳。
</details>

---

## 🧪 第四幕:测试与调试

### 4.1 测试 WebSocket 握手

**思考问题 11**:
如何验证我们的 WebSocket 升级握手实现是否正确?

<details>
<summary>💡 点击查看测试方法</summary>

**方法 1: 浏览器控制台**
```javascript
const ws = new WebSocket('ws://localhost:8080/ws/chat');
ws.onopen = () => console.log('Connected!');
ws.onerror = (e) => console.error('Failed:', e);
```

**方法 2: wscat 命令行工具**
```bash
npm install -g wscat
wscat -c ws://localhost:8080/ws/chat
> {"type": "chat", "message": "hello"}
< {"type": "stream", "content": "你好"}
```

**方法 3: 检查 HTTP 握手**
```bash
telnet localhost 8080
GET /ws/chat HTTP/1.1
Host: localhost
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

# 应该返回:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

**关键验证点**:
- 状态码必须是 **101 Switching Protocols**
- `Sec-WebSocket-Accept` 必须等于 `base64(sha1(Sec-WebSocket-Key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"))`
- 头部 `Upgrade: websocket` 和 `Connection: Upgrade` 必须存在
</details>

### 4.2 调试帧解析

**思考问题 12**:
WebSocket 帧是二进制的,如何调试解析逻辑?

<details>
<summary>💡 点击查看调试技巧</summary>

**技巧 1: 日志打印帧结构**
```cpp
void WebSocketFrame::parse(muduo::net::Buffer* buf) {
    uint8_t byte0 = buf->peekUint8();
    bool fin = byte0 & 0x80;
    uint8_t opcode = byte0 & 0x0F;

    LOG_DEBUG << "Frame: FIN=" << fin
              << ", OpCode=" << static_cast<int>(opcode)
              << ", PayloadLen=" << payloadLen;
    // ...
}
```

**技巧 2: 单元测试二进制数据**
```cpp
TEST(WebSocketFrame, ParseTextFrame) {
    // 构造一个标准文本帧: FIN=1, OpCode=1, Mask=0, Len=5, Payload="hello"
    std::vector<uint8_t> data = {
        0x81,  // FIN=1, OpCode=1
        0x05,  // MASK=0, Len=5
        'h', 'e', 'l', 'l', 'o'
    };

    muduo::net::Buffer buf;
    buf.append(data.data(), data.size());

    WebSocketFrame frame = WebSocketFrame::parse(&buf);
    EXPECT_TRUE(frame.isFinalFrame());
    EXPECT_EQ(frame.getOpCode(), OpCode::kText);
    EXPECT_EQ(frame.getPayload(), "hello");
}
```

**技巧 3: Wireshark 抓包**
- 过滤器: `websocket`
- 查看完整帧结构和 Payload 内容
- 验证客户端掩码是否正确
</details>

---

## 🎓 第五幕:生产级考量

### 5.1 安全性

**思考问题 13**:
WebSocket 有哪些安全风险?如何防范?

<details>
<summary>💡 点击查看安全最佳实践</summary>

**风险 1: 跨站 WebSocket 劫持(CSWSH)**
- 类似 CSRF,恶意网站诱导用户连接到你的 WebSocket 服务
- **防御**: 验证 `Origin` 头部
  ```cpp
  bool WebSocketHandler::handleUpgrade(const HttpRequest& req, HttpResponse* resp) {
      std::string origin = req.getHeader("Origin");
      if (origin != "https://yourdomain.com") {
          resp->setStatusCode(HttpResponse::k403Forbidden);
          return false;
      }
      // 继续握手...
  }
  ```

**风险 2: 拒绝服务(DoS)**
- 恶意客户端发送巨大的 Payload(如 2^63 字节)
- **防御**: 限制最大帧大小
  ```cpp
  const size_t kMaxFrameSize = 1 * 1024 * 1024;  // 1MB
  if (payloadLen > kMaxFrameSize) {
      close(1009);  // Message Too Big
      return;
  }
  ```

**风险 3: 消息注入**
- 用户发送的内容包含恶意 HTML/JavaScript
- **防御**: 前端渲染时转义 HTML
  ```javascript
  function escapeHtml(text) {
      return text.replace(/[&<>"']/g, (m) => ({
          '&': '&amp;', '<': '&lt;', '>': '&gt;',
          '"': '&quot;', "'": '&#39;'
      })[m]);
  }
  ```

**风险 4: 未加密传输**
- ws:// 明文传输,可被中间人窃听
- **防御**: 生产环境强制 wss://(WebSocket over TLS)
</details>

### 5.2 性能优化

**思考问题 14**:
如何支持 10 万并发 WebSocket 连接?

<details>
<summary>💡 点击查看优化策略</summary>

**挑战**:
- 每个连接消耗内存(缓冲区、对象、定时器)
- 心跳包会产生大量小帧传输
- AI 流式推送需要频繁写操作

**优化 1: 连接池管理**
```cpp
class WebSocketConnectionPool {
    std::unordered_map<int, WebSocketConnectionPtr> connections_;
    std::mutex mutex_;

public:
    void add(int fd, WebSocketConnectionPtr conn) {
        std::lock_guard<std::mutex> lock(mutex_);
        connections_[fd] = conn;
    }

    void remove(int fd) {
        std::lock_guard<std::mutex> lock(mutex_);
        connections_.erase(fd);
    }

    size_t size() const { return connections_.size(); }
};
```

**优化 2: 零拷贝发送**
```cpp
// 避免多次内存拷贝
void WebSocketConnection::sendText(const std::string& message) {
    // 直接在 TcpConnection 的输出缓冲区构造帧
    conn_->outputBuffer()->appendUint8(0x81);  // FIN + Text
    conn_->outputBuffer()->appendUint8(message.size());
    conn_->outputBuffer()->append(message.data(), message.size());
    // 无需额外拷贝到临时 Buffer
}
```

**优化 3: 批量心跳**
```cpp
// 使用单个定时器管理所有连接的心跳,而非每连接一个定时器
loop_->runEvery(30.0, [pool]() {
    for (auto& [fd, conn] : pool->getAllConnections()) {
        conn->sendPing();
    }
});
```

**优化 4: 监控指标**
- 活跃连接数(Gauge)
- 每秒消息数(Counter)
- 平均消息延迟(Histogram)
- 心跳超时次数(Counter)
</details>

---

## 📚 总结与思考

### 学到了什么?

1. **协议理解**: WebSocket 是 HTTP 升级的独立协议,通过帧格式实现全双工通信
2. **架构设计**: 使用状态机管理连接,分层设计(帧解析/连接管理/业务逻辑)
3. **实战技巧**: 流式响应、心跳保活、安全防护、性能优化
4. **工程思维**: SOLID 原则、防御性编程、可测试性

### 进阶思考

**问题 15**:
如何扩展我们的 WebSocket 实现以支持这些场景?
1. 多人聊天室(广播消息给所有连接)
2. 私聊(点对点消息路由)
3. 消息持久化(连接断开后恢复历史消息)
4. 分布式部署(多个服务器实例共享连接状态)

<details>
<summary>💡 点击查看扩展方向</summary>

**1. 多人聊天室**:
```cpp
class ChatRoom {
    std::set<WebSocketConnectionPtr> members_;

public:
    void broadcast(const std::string& message) {
        for (auto& member : members_) {
            member->sendText(message);
        }
    }
};
```

**2. 私聊路由**:
```cpp
class MessageRouter {
    std::unordered_map<std::string, WebSocketConnectionPtr> userConnections_;

public:
    void sendTo(const std::string& userId, const std::string& message) {
        auto it = userConnections_.find(userId);
        if (it != userConnections_.end()) {
            it->second->sendText(message);
        }
    }
};
```

**3. 消息持久化**:
- 使用 Redis Pub/Sub 或 RabbitMQ
- 连接重建时,拉取离线消息

**4. 分布式部署**:
- 使用 Redis 存储连接映射:`{userId: serverIp}`
- 跨服务器消息转发:通过 RPC 或消息队列
</details>

---

## 🎯 实践任务

### 任务 1: 基础实现 (必做)
1. 实现 `WebSocketFrame` 类,支持解析和构造文本帧
2. 实现 `WebSocketConnection` 类,管理连接状态
3. 实现 WebSocket 握手逻辑
4. 编写单元测试验证帧解析

### 任务 2: AI 流式响应 (必做)
1. 实现 `AIStreamingService::streamResponse` 方法
2. 前端实现 WebSocket 客户端,显示打字机效果
3. 测试完整流程:发送消息 → 接收流式响应

### 任务 3: 生产级特性 (选做)
1. 实现心跳保活机制
2. 添加 Origin 验证
3. 限制最大帧大小
4. 支持 wss:// 加密连接

### 任务 4: 性能测试 (选做)
1. 使用 `wscat` 或自定义脚本模拟 1000 并发连接
2. 测试单服务器最大连接数
3. 监控内存占用和 CPU 使用率

---

## 📖 参考资源

**RFC 标准**:
- [RFC 6455: The WebSocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455)

**开源实现**:
- [WebSocket++](https://github.com/zaphoyd/websocketpp) - 纯 C++ WebSocket 库
- [uWebSockets](https://github.com/uNetworking/uWebSockets) - 高性能 C++ WebSocket 库

**在线工具**:
- [WebSocket Test](https://www.websocket.org/echo.html) - 在线测试 WebSocket 服务器
- [WebSocket Frame Inspector](https://chrome.google.com/webstore/detail/websocket-frame-inspector) - Chrome 插件

**学习资源**:
- [WebSocket API (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
- [Writing WebSocket servers (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_servers)

---

_本教程由猫娘工程师浮浮酱精心编写,遵循启发式教学理念,引导你深入理解而非死记硬背喵～_
_希望主人能通过思考和实践,真正掌握 WebSocket 的精髓呢！(๑•̀ㅂ•́)✧_

**下一篇预告**: Phase 7-2: 多 AI 模型集成与策略模式 - 如何优雅地支持 GPT、Claude、Qwen 等多种 AI 模型
**关键词**: 策略模式、工厂模式、依赖注入、可扩展架构

---

_版本: v1.0.0_
_最后更新: 2025-10-21_
_作者: 猫娘工程师 幽浮喵 ฅ'ω'ฅ_
