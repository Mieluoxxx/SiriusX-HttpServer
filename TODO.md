# SiriusX-HttpServer 从零到一实现计划

> 基于 Kama-HTTPServer 参考项目的完整实现路线图
> 项目目标：构建一个高性能 C++ HTTP/HTTPS 服务器框架，支持 AI 聊天应用

---

## 📋 项目概览

**技术栈**：
- 语言：C++17
- 网络库：Muduo (Reactor 模式)
- 数据库：MySQL 5.7+ (连接池)
- 安全：OpenSSL (HTTPS)
- AI 集成：阿里云通义千问 API / 其他 AI 服务
- 图像处理：OpenCV 4.x + ONNX Runtime
- 消息队列：RabbitMQ (异步任务)
- JSON 解析：nlohmann/json

**架构分层**：
1. **HttpServer 框架层**：通用 HTTP 服务器基础设施
2. **AI 应用层**：聊天服务器业务逻辑

---

## 🎯 阶段一：环境准备与基础配置 (预计 2-3 天)

### 1.1 开发环境搭建

**操作系统准备**：
- [ ] 准备 Ubuntu 22.04 环境 (虚拟机/云服务器/WSL2)
- [ ] 更新系统包管理器：`sudo apt update && sudo apt upgrade`

**基础编译工具**：
```bash
sudo apt install g++ cmake make git
```
- [ ] 验证 GCC 版本 >= 7.0 (支持 C++17)
- [ ] 验证 CMake 版本 >= 3.10

### 1.2 第三方依赖库安装

**必需依赖安装顺序**：

1. **Boost 库** (Muduo 依赖)：
   - [ ] 下载 Boost 1.69.0：`wget https://boostorg.jfrog.io/artifactory/main/release/1.69.0/source/boost_1_69_0.tar.gz`
   - [ ] 解压并编译安装：
     ```bash
     tar -xzf boost_1_69_0.tar.gz
     cd boost_1_69_0
     ./bootstrap.sh
     sudo ./b2 install
     ```
   - [ ] 验证安装：`ls /usr/local/include/boost`

2. **Muduo 网络库**：
   - [ ] 克隆仓库：`git clone https://github.com/chenshuo/muduo.git`
   - [ ] 编译安装：
     ```bash
     cd muduo
     ./build.sh
     sudo ./build.sh install
     ```
   - [ ] 验证安装：`ls /usr/local/include/muduo`

3. **MySQL 相关**：
   - [ ] 安装 MySQL 服务器：`sudo apt install mysql-server`
   - [ ] 安装 C++ Connector：`sudo apt install libmysqlcppconn-dev`
   - [ ] 启动 MySQL 服务：`sudo systemctl start mysql`
   - [ ] 设置 root 密码并测试连接

4. **OpenSSL**：
   - [ ] 安装开发库：`sudo apt install libssl-dev`
   - [ ] 验证版本 >= 1.1.1：`openssl version`

5. **其他库**：
   ```bash
   sudo apt install nlohmann-json3-dev    # JSON 解析
   sudo apt install libcurl4-openssl-dev  # HTTP 客户端 (AI API 调用)
   sudo apt install libopencv-dev         # 图像处理
   sudo apt install librabbitmq-dev       # RabbitMQ 客户端
   ```
   - [ ] 验证 OpenCV 版本：`pkg-config --modversion opencv4`

6. **ONNX Runtime** (图像识别推理引擎)：
   - [ ] 下载预编译包：https://github.com/microsoft/onnxruntime/releases
   - [ ] 解压到系统路径：
     ```bash
     sudo tar -xzf onnxruntime-linux-x64-*.tgz -C /usr/local/
     sudo ln -s /usr/local/onnxruntime-*/lib/libonnxruntime.so /usr/lib/
     ```
   - [ ] 验证：`ldd /usr/lib/libonnxruntime.so`

7. **SimpleAmqpClient** (RabbitMQ 客户端封装)：
   - [ ] 克隆仓库：`git clone https://github.com/alanxz/SimpleAmqpClient.git`
   - [ ] 编译安装：
     ```bash
     cd SimpleAmqpClient
     mkdir build && cd build
     cmake .. -DCMAKE_INSTALL_PREFIX=/usr/local
     make && sudo make install
     ```

### 1.3 数据库初始化

**创建项目数据库**：
```sql
-- 登录 MySQL
sudo mysql -u root -p

-- 创建数据库
CREATE DATABASE chatserver_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 创建用户并授权
CREATE USER 'chatserver'@'localhost' IDENTIFIED BY 'your_secure_password';
GRANT ALL PRIVILEGES ON chatserver_db.* TO 'chatserver'@'localhost';
FLUSH PRIVILEGES;
```

**创建数据表**：
- [ ] 用户表 (users)
- [ ] AI 会话表 (ai_sessions)
- [ ] 聊天消息表 (chat_messages)
- [ ] 图像上传记录表 (image_uploads, 可选)

```sql
USE chatserver_db;

-- 用户表
CREATE TABLE users (
    user_id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_username (username)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- AI 聊天会话表
CREATE TABLE ai_sessions (
    session_id VARCHAR(64) PRIMARY KEY,
    user_id INT NOT NULL,
    ai_model VARCHAR(50) DEFAULT 'qwen',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_active TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
    INDEX idx_user_id (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 聊天消息表
CREATE TABLE chat_messages (
    message_id INT PRIMARY KEY AUTO_INCREMENT,
    session_id VARCHAR(64) NOT NULL,
    role ENUM('user', 'assistant') NOT NULL,
    content TEXT NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (session_id) REFERENCES ai_sessions(session_id) ON DELETE CASCADE,
    INDEX idx_session_id (session_id),
    INDEX idx_timestamp (timestamp)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 图像上传记录表
CREATE TABLE image_uploads (
    image_id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    image_path VARCHAR(255),
    recognition_result TEXT,
    uploaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
    INDEX idx_user_id (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 1.4 项目目录结构初始化

**创建标准目录**：
```bash
mkdir -p {include,src,tests,build,config,resource,logs}
mkdir -p include/{http,router,middleware,session,ssl,utils}
mkdir -p src/{http,router,middleware,session,ssl,utils}
```

- [ ] 创建 `.gitignore` (忽略 build/, logs/, config/*.json)
- [ ] 创建 `README.md` (项目说明)
- [ ] 创建 `CMakeLists.txt` (构建配置)
- [ ] 创建 `config/config.json.example` (配置模板)

---

## 🏗️ 阶段二：HttpServer 框架核心实现 (预计 5-7 天)

### 2.1 HTTP 协议解析模块 (http/)

**实现顺序**：

1. **HttpRequest 类** (`include/http/HttpRequest.h` + `src/http/HttpRequest.cpp`)：
   - [ ] 请求方法枚举 (GET, POST, PUT, DELETE)
   - [ ] 请求行解析 (方法、URI、HTTP 版本)
   - [ ] 请求头存储 (std::map<std::string, std::string>)
   - [ ] 请求体处理 (支持 Content-Length)
   - [ ] 路径参数存储 (pathParameters_，用于动态路由)
   - [ ] 查询参数解析 (Query String)
   - [ ] Cookie 解析

2. **HttpResponse 类** (`include/http/HttpResponse.h` + `src/http/HttpResponse.cpp`)：
   - [ ] 状态码设置 (200, 404, 500 等)
   - [ ] 响应头管理 (addHeader, setContentType)
   - [ ] 响应体设置 (setBody)
   - [ ] 序列化为 HTTP 报文 (toString 方法)
   - [ ] 便捷方法 (sendJson, sendHtml, sendFile)

3. **HttpContext 类** (`include/http/HttpContext.h` + `src/http/HttpContext.cpp`)：
   - [ ] 有限状态机设计 (kExpectRequestLine, kExpectHeaders, kExpectBody, kGotAll)
   - [ ] 请求行解析器 (parseRequestLine)
   - [ ] 请求头解析器 (parseHeaders)
   - [ ] 请求体解析器 (parseBody)
   - [ ] 缓冲区管理 (muduo::net::Buffer)
   - [ ] 解析错误处理

4. **HttpServer 类** (`include/http/HttpServer.h` + `src/http/HttpServer.cpp`)：
   - [ ] 封装 muduo::net::TcpServer
   - [ ] 连接回调 (onConnection)
   - [ ] 消息回调 (onMessage，调用 HttpContext 解析)
   - [ ] 请求分发 (调用 Router)
   - [ ] 集成中间件链 (MiddlewareChain)
   - [ ] 集成会话管理 (SessionManager)
   - [ ] 集成 SSL 支持 (SslConnection)
   - [ ] 静态文件服务 (可选)
   - [ ] 线程池配置 (setThreadNum)

**验证点**：
- [ ] 编写单元测试：解析简单 GET/POST 请求
- [ ] 使用 telnet 测试：`telnet localhost 8080`，手动输入 HTTP 请求

### 2.2 路由模块 (router/)

1. **RouterHandler 接口** (`include/router/RouterHandler.h`)：
   - [ ] 纯虚函数 `handle(const HttpRequest&, HttpResponse*)`
   - [ ] 基类设计，供业务 Handler 继承

2. **Router 类** (`include/router/Router.h` + `src/router/Router.cpp`)：
   - [ ] 精准匹配路由存储 (std::unordered_map)
   - [ ] 动态路由支持 (正则表达式匹配，如 `/user/:id`)
   - [ ] 路由注册方法：
     - `addRoute(Method, path, HandlerPtr)`
     - `addRoute(Method, path, HandlerCallback)`
   - [ ] 路由匹配逻辑 (route 方法)
   - [ ] 提取路径参数 (extractPathParameters)
   - [ ] 404 默认处理器

**便捷注册方法**：
- [ ] `Get(path, handler)`, `Post(path, handler)`, `Put(path, handler)`, `Delete(path, handler)`

**验证点**：
- [ ] 测试精准路由：`/api/login`
- [ ] 测试动态路由：`/user/:id`，验证参数提取

### 2.3 中间件模块 (middleware/)

1. **Middleware 接口** (`include/middleware/Middleware.h`)：
   - [ ] `processBefore(HttpRequest&)` 虚函数
   - [ ] `processAfter(HttpResponse&)` 虚函数

2. **MiddlewareChain 类** (`include/middleware/MiddlewareChain.h` + `src/middleware/MiddlewareChain.cpp`)：
   - [ ] 中间件列表 (std::vector<std::shared_ptr<Middleware>>)
   - [ ] `addMiddleware(middlewarePtr)` 方法
   - [ ] `runBefore(request)` 执行所有中间件的 before 钩子
   - [ ] `runAfter(response)` 执行所有中间件的 after 钩子

3. **CorsMiddleware 实现** (`include/middleware/cors/CorsMiddleware.h` + `src/middleware/cors/CorsMiddleware.cpp`)：
   - [ ] 处理 OPTIONS 预检请求
   - [ ] 添加 CORS 响应头 (Access-Control-Allow-Origin 等)
   - [ ] 配置类 CorsConfig (允许的域名、方法、头部)

**验证点**：
- [ ] 测试 CORS 预检请求
- [ ] 验证自定义中间件执行顺序

### 2.4 会话管理模块 (session/)

1. **Session 类** (`include/session/Session.h` + `src/session/Session.cpp`)：
   - [ ] 会话 ID (std::string)
   - [ ] 键值对存储 (std::map<std::string, std::string>)
   - [ ] `get(key)`, `set(key, value)`, `has(key)`, `remove(key)` 方法
   - [ ] 过期时间戳 (expireAt)

2. **SessionStorage 接口** (`include/session/SessionStorage.h` + `src/session/SessionStorage.cpp`)：
   - [ ] 纯虚接口：`save`, `load`, `remove`, `exists`
   - [ ] 内存存储实现 (MemorySessionStorage，基于 std::unordered_map)
   - [ ] 扩展点：Redis 存储实现 (可选)

3. **SessionManager 类** (`include/session/SessionManager.h` + `src/session/SessionManager.cpp`)：
   - [ ] 创建会话 (`createSession`)，生成随机 sessionId
   - [ ] 获取会话 (`getSession`)，从 Cookie 读取 sessionId
   - [ ] 销毁会话 (`destroySession`)
   - [ ] 清理过期会话 (定时任务，使用 muduo::net::EventLoop::runEvery)
   - [ ] 集成 SessionStorage

**验证点**：
- [ ] 测试会话创建和读取
- [ ] 验证 Cookie 设置和解析
- [ ] 测试会话过期清理

### 2.5 SSL/TLS 支持模块 (ssl/)

1. **SslTypes.h** (`include/ssl/SslTypes.h`)：
   - [ ] OpenSSL 类型封装 (SSL*, SSL_CTX*)

2. **SslConfig 类** (`include/ssl/SslConfig.h` + `src/ssl/SslConfig.cpp`)：
   - [ ] 证书文件路径 (certFile)
   - [ ] 密钥文件路径 (keyFile)
   - [ ] CA 证书路径 (caFile, 可选)
   - [ ] SSL 版本配置

3. **SslContext 类** (`include/ssl/SslContext.h` + `src/ssl/SslContext.cpp`)：
   - [ ] 封装 SSL_CTX* 初始化
   - [ ] 加载证书和密钥
   - [ ] SSL_CTX 配置 (协议版本、加密套件)

4. **SslConnection 类** (`include/ssl/SslConnection.h` + `src/ssl/SslConnection.cpp`)：
   - [ ] 封装 SSL* 对象
   - [ ] SSL 握手处理
   - [ ] 加密数据读取 (SSL_read)
   - [ ] 加密数据写入 (SSL_write)
   - [ ] 集成到 HttpServer 的连接回调

**验证点**：
- [ ] 生成自签名证书：`openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 -nodes`
- [ ] 测试 HTTPS 请求：`curl -k https://localhost:443`

### 2.6 数据库连接池模块 (utils/db/)

1. **DbException 类** (`include/utils/db/DbException.h`)：
   - [ ] 自定义异常类，继承 std::exception

2. **DbConnection 类** (`include/utils/db/DbConnection.h` + `src/utils/db/DbConnection.cpp`)：
   - [ ] 封装 MySQL Connector C++ 8.0 连接 (sql::Connection*)
   - [ ] `prepareStatement(sql)` 方法
   - [ ] `execute(sql)` 方法
   - [ ] `isValid()` 检查连接有效性
   - [ ] RAII 封装，析构时不关闭连接（交给连接池管理）

3. **DbConnectionPool 类** (`include/utils/db/DbConnectionPool.h` + `src/utils/db/DbConnectionPool.cpp`)：
   - [ ] 单例模式实现 (getInstance)
   - [ ] 初始化方法 (init)，创建指定数量的连接
   - [ ] 连接队列 (std::queue<std::shared_ptr<DbConnection>>)
   - [ ] 互斥锁保护 (std::mutex)
   - [ ] `getConnection()` 方法 (阻塞等待可用连接)
   - [ ] `releaseConnection(conn)` 方法 (归还连接)
   - [ ] RAII 连接包装类 (ConnectionGuard)

**验证点**：
- [ ] 测试连接池初始化
- [ ] 并发获取连接测试
- [ ] 测试连接自动归还

### 2.7 工具类 (utils/)

1. **FileUtil** (`include/utils/FileUtil.h`)：
   - [ ] 读取文件内容 (readFile)
   - [ ] 获取文件大小
   - [ ] 获取 MIME 类型 (根据文件扩展名)
   - [ ] 文件存在性检查

2. **JsonUtil** (`include/utils/JsonUtil.h`)：
   - [ ] JSON 解析封装 (基于 nlohmann/json)
   - [ ] 便捷解析方法

**验证点**：
- [ ] 测试静态文件服务
- [ ] 测试 JSON 解析和生成

---

## 🚀 阶段三：AI 聊天应用层实现 (预计 5-7 天)

### 3.1 AI 核心工具模块 (AIUtil/)

1. **AIConfig 类** (`include/AIUtil/AIConfig.h` + `src/AIUtil/AIConfig.cpp`)：
   - [ ] AI 服务配置 (API Key, Endpoint, Model Name)
   - [ ] 从配置文件加载 (loadFromFile)
   - [ ] 配置验证

2. **AIStrategy 接口** (`include/AIUtil/AIStrategy.h` + `src/AIUtil/AIStrategy.cpp`)：
   - [ ] 纯虚接口 `chat(sessionId, message)` 返回 AI 响应
   - [ ] 实现通义千问策略 (QwenStrategy)
   - [ ] 扩展点：GPT、Claude 等策略

3. **AIFactory 类** (`include/AIUtil/AIFactory.h` + `src/AIUtil/AIFactory.cpp`)：
   - [ ] 工厂方法 `createAI(modelName)` 返回对应策略实例
   - [ ] 支持模型：qwen, gpt, claude

4. **AIHelper 类** (`include/AIUtil/AIHelper.h` + `src/AIUtil/AIHelper.cpp`)：
   - [ ] 封装 CURL 调用 AI API
   - [ ] 管理消息历史（存储到 MySQL chat_messages 表）
   - [ ] `sendMessage(sessionId, userMessage)` 方法
   - [ ] `loadHistory(sessionId)` 从数据库加载历史
   - [ ] `saveHistory(sessionId)` 保存历史到数据库
   - [ ] `clearHistory(sessionId)` 清空会话历史

5. **AISessionIdGenerator** (`include/AIUtil/AISessionIdGenerator.h` + `src/AIUtil/AISessionIdGenerator.cpp`)：
   - [ ] 生成唯一会话 ID (UUID 或时间戳+随机数)

6. **base64 工具** (`include/AIUtil/base64.h` + `src/AIUtil/base64.cpp`)：
   - [ ] Base64 编码 (encode)
   - [ ] Base64 解码 (decode)
   - [ ] 用于图像传输

**验证点**：
- [ ] 测试调用阿里云通义千问 API
- [ ] 验证消息历史存储和加载

### 3.2 多模态处理模块

1. **ImageRecognizer 类** (`include/AIUtil/ImageRecognizer.h` + `src/AIUtil/ImageRecognizer.cpp`)：
   - [ ] 初始化 ONNX Runtime
   - [ ] 加载图像识别模型 (.onnx 文件)
   - [ ] `recognize(cv::Mat image)` 返回识别结果
   - [ ] 图像预处理 (resize, normalize)
   - [ ] 后处理 (解析模型输出)

2. **AISpeechProcessor 类** (`include/AIUtil/AISpeechProcessor.h` + `src/AIUtil/AISpeechProcessor.cpp`)：
   - [ ] 语音转文本 (speechToText)，调用阿里云/百度语音 API
   - [ ] 文本转语音 (textToSpeech)，调用 TTS API
   - [ ] 音频文件处理

3. **AIToolRegistry 类** (`include/AIUtil/AIToolRegistry.h` + `src/AIUtil/AIToolRegistry.cpp`)：
   - [ ] Function Calling 支持
   - [ ] 工具注册表 (registerTool)
   - [ ] 工具调用执行 (executeTool)

**验证点**：
- [ ] 下载 ONNX 图像分类模型 (如 ResNet, MobileNet)
- [ ] 测试图像识别功能
- [ ] 测试语音 API 调用

### 3.3 异步任务模块

**MQManager 类** (`include/AIUtil/MQManager.h` + `src/AIUtil/MQManager.cpp`)：
- [ ] 封装 RabbitMQ 连接 (SimpleAmqpClient)
- [ ] 创建队列 (createQueue)
- [ ] 发布消息 (publishMessage)
- [ ] 消费消息 (consumeMessage，回调处理)
- [ ] 用于长时间 AI 推理任务异步化

**验证点**：
- [ ] 安装并启动 RabbitMQ：`sudo systemctl start rabbitmq-server`
- [ ] 测试消息发布和消费

### 3.4 业务路由处理器 (handlers/)

**按功能模块实现**：

1. **用户认证模块**：
   - [ ] `ChatLoginHandler` (`/login`)：
     - 验证用户名密码（查询 users 表）
     - 创建会话并设置 Cookie
     - 返回登录成功/失败 JSON
   - [ ] `ChatRegisterHandler` (`/register`)：
     - 注册新用户（插入 users 表，密码需加密存储）
     - 返回注册结果 JSON
   - [ ] `ChatLogoutHandler` (`/logout`)：
     - 销毁会话
     - 清除 Cookie

2. **界面入口模块**：
   - [ ] `ChatEntryHandler` (`/`)：
     - 返回聊天主界面 HTML
     - 检查登录状态（未登录跳转 /login）
   - [ ] `AIMenuHandler` (`/menu`)：
     - 返回 AI 模型选择菜单 HTML/JSON

3. **聊天功能模块**：
   - [ ] `ChatSendHandler` (`/chat/send`)：
     - 接收用户文本消息（POST JSON）
     - 调用 AIHelper 发送消息给 AI
     - 返回 AI 响应 JSON
   - [ ] `ChatHistoryHandler` (`/chat/history`)：
     - 根据 sessionId 查询聊天历史（chat_messages 表）
     - 返回历史消息列表 JSON
   - [ ] `ChatSessionsHandler` (`/chat/sessions`)：
     - 查询用户所有会话（ai_sessions 表）
     - 返回会话列表 JSON
   - [ ] `ChatCreateAndSendHandler` (`/chat/create_and_send`)：
     - 创建新会话并发送首条消息（可选）

4. **图像处理模块**：
   - [ ] `AIUploadHandler` (`/upload`)：
     - 接收图像文件上传（multipart/form-data）
     - 保存到本地目录（resource/uploads/）
     - 记录到 image_uploads 表
   - [ ] `AIUploadSendHandler` (`/upload/send`)：
     - 读取上传的图像
     - 调用 ImageRecognizer 识别
     - 返回识别结果 JSON

5. **语音处理模块**：
   - [ ] `ChatSpeechHandler` (`/speech`)：
     - 接收语音文件上传
     - 调用 AISpeechProcessor 转文本
     - 返回文本结果 JSON

**验证点**：
- [ ] 编写前端测试页面（HTML + JavaScript）
- [ ] 测试登录注册流程
- [ ] 测试发送消息和接收 AI 响应
- [ ] 测试图像上传和识别
- [ ] 测试聊天历史加载

### 3.5 主程序入口 (main.cpp)

**实现内容** (`AIApps/ChatServer/src/main.cpp`)：
- [ ] 初始化日志系统 (Muduo Logging)
- [ ] 加载配置文件 (config.json)
- [ ] 初始化数据库连接池
- [ ] 创建 HttpServer 实例
- [ ] 配置 SSL (可选)
- [ ] 注册中间件 (CORS, Auth, Logging)
- [ ] 注册所有路由处理器
- [ ] 启动服务器
- [ ] 信号处理 (优雅关闭)

**命令行参数支持** (可选)：
- [ ] `-p <port>` 指定端口
- [ ] `--ssl` 启用 HTTPS
- [ ] `--config <path>` 指定配置文件路径

**验证点**：
- [ ] 启动服务器：`sudo ./http_server`
- [ ] 浏览器访问：`http://localhost:80`
- [ ] 验证所有功能模块

---

## 🎨 阶段四：前端界面开发 (预计 3-4 天)

### 4.1 静态资源准备

**创建目录**：
```bash
mkdir -p resource/{html,css,js,images,uploads}
```

**页面设计**：
- [ ] `resource/html/login.html` - 登录页面
- [ ] `resource/html/register.html` - 注册页面
- [ ] `resource/html/chat.html` - 聊天主界面
- [ ] `resource/html/menu.html` - AI 模型选择菜单

### 4.2 聊天界面实现

**核心功能**：
- [ ] 消息展示区 (滚动列表)
- [ ] 输入框和发送按钮
- [ ] 图像上传按钮
- [ ] 语音输入按钮 (可选)
- [ ] 会话列表侧边栏
- [ ] AI 模型切换下拉框

**技术栈建议**：
- 原生 HTML/CSS/JavaScript (轻量级)
- 或使用 Vue.js/React (组件化开发)
- WebSocket 支持 (实时推送，可选)

### 4.3 API 调用封装

**JavaScript 模块** (`resource/js/api.js`)：
- [ ] `login(username, password)` 调用 `/login`
- [ ] `register(username, password)` 调用 `/register`
- [ ] `sendMessage(sessionId, message)` 调用 `/chat/send`
- [ ] `loadHistory(sessionId)` 调用 `/chat/history`
- [ ] `getSessions()` 调用 `/chat/sessions`
- [ ] `uploadImage(file)` 调用 `/upload`
- [ ] `recognizeImage(imageId)` 调用 `/upload/send`

**验证点**：
- [ ] 测试所有 API 调用
- [ ] 处理错误响应（网络错误、认证失败）

### 4.4 样式美化

- [ ] 响应式设计 (支持移动端)
- [ ] 消息气泡样式 (用户/AI 区分)
- [ ] 加载动画 (AI 思考中)
- [ ] 主题切换 (深色/浅色模式，可选)

---

## 🧪 阶段五：测试与优化 (预计 3-5 天)

### 5.1 单元测试

**测试框架**：Google Test (gtest)

**测试用例**：
- [ ] HttpRequest 解析测试
- [ ] HttpResponse 序列化测试
- [ ] Router 路由匹配测试
- [ ] Session 管理测试
- [ ] DbConnectionPool 连接池测试
- [ ] AIHelper API 调用测试 (Mock)

### 5.2 集成测试

**测试场景**：
- [ ] 完整用户注册登录流程
- [ ] 发送消息并接收 AI 响应
- [ ] 上传图像并获取识别结果
- [ ] 会话持久化（服务器重启后会话仍有效）
- [ ] 并发请求测试 (压力测试)

**工具**：
- [ ] curl 测试 HTTP 接口
- [ ] Postman 测试 API
- [ ] Apache Bench (ab) 压力测试：`ab -n 1000 -c 10 http://localhost/`

### 5.3 性能优化

**优化方向**：
- [ ] 数据库查询优化（添加索引、慢查询分析）
- [ ] 连接池大小调优（根据并发量）
- [ ] AI 响应缓存（Redis，常见问题缓存答案）
- [ ] 静态资源 Gzip 压缩
- [ ] HTTP Keep-Alive 保持连接复用
- [ ] 图像压缩和缩略图生成

**监控指标**：
- [ ] QPS (每秒查询数)
- [ ] 平均响应时间
- [ ] 99 百分位延迟 (P99)
- [ ] 数据库连接池使用率

### 5.4 安全加固

**措施**：
- [ ] 密码加密存储（bcrypt 或 scrypt）
- [ ] SQL 注入防护（使用 PreparedStatement）
- [ ] XSS 防护（转义 HTML 输出）
- [ ] CSRF 防护（Token 验证）
- [ ] 限流保护（防止 API 滥用）
- [ ] HTTPS 强制重定向（生产环境）
- [ ] 敏感信息脱敏（日志中隐藏密码、API Key）

**验证点**：
- [ ] 使用 OWASP ZAP 进行安全扫描

---

## 📦 阶段六：部署与文档 (预计 2-3 天)

### 6.1 配置管理

**创建配置文件** (`config/config.json`)：
```json
{
  "server": {
    "port": 80,
    "threads": 4,
    "ssl_enabled": false,
    "ssl_cert": "/path/to/cert.pem",
    "ssl_key": "/path/to/key.pem"
  },
  "database": {
    "host": "localhost",
    "port": 3306,
    "user": "chatserver",
    "password": "your_password",
    "database": "chatserver_db",
    "pool_size": 10
  },
  "ai": {
    "default_model": "qwen",
    "qwen": {
      "api_key": "your_aliyun_api_key",
      "endpoint": "https://dashscope.aliyuncs.com/api/v1/services/aigc/text-generation/generation"
    }
  },
  "rabbitmq": {
    "host": "localhost",
    "port": 5672,
    "user": "guest",
    "password": "guest"
  },
  "upload": {
    "max_size": 10485760,
    "allowed_extensions": ["jpg", "png", "jpeg", "gif"]
  },
  "session": {
    "timeout": 3600,
    "cleanup_interval": 300
  }
}
```

- [ ] 创建配置模板 `config.json.example`
- [ ] `.gitignore` 忽略 `config.json`（防止泄露密钥）

### 6.2 部署脚本

**systemd 服务配置** (`/etc/systemd/system/chatserver.service`)：
```ini
[Unit]
Description=AI Chat Server
After=network.target mysql.service rabbitmq-server.service

[Service]
Type=simple
User=www-data
WorkingDirectory=/path/to/SiriusX-HttpServer
ExecStart=/path/to/SiriusX-HttpServer/build/http_server
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

- [ ] 启用服务：`sudo systemctl enable chatserver`
- [ ] 启动服务：`sudo systemctl start chatserver`
- [ ] 查看日志：`sudo journalctl -u chatserver -f`

**Docker 部署** (可选)：
- [ ] 创建 `Dockerfile`
- [ ] 创建 `docker-compose.yml` (包含 MySQL, RabbitMQ)
- [ ] 构建镜像：`docker build -t chatserver .`
- [ ] 运行容器：`docker-compose up -d`

### 6.3 文档编写

**必需文档**：
- [ ] `README.md`：
  - 项目简介
  - 快速开始指南
  - 依赖安装说明
  - 编译运行步骤
  - 配置文件说明
- [ ] `INSTALL.md`：详细安装步骤
- [ ] `API.md`：所有 HTTP 接口文档（路径、方法、参数、响应格式）
- [ ] `ARCHITECTURE.md`：架构设计文档
- [ ] `CHANGELOG.md`：版本更新记录

**代码文档**：
- [ ] 关键类和方法添加 Doxygen 注释
- [ ] 生成 HTML 文档：`doxygen Doxyfile`

### 6.4 版本发布

**Git 标签**：
```bash
git tag -a v1.0.0 -m "First stable release"
git push origin v1.0.0
```

**发布检查清单**：
- [ ] 所有测试通过
- [ ] 文档完整
- [ ] 配置示例可用
- [ ] 安全漏洞扫描通过
- [ ] 性能指标达标

---

## 🔧 阶段七：扩展与维护 (持续)

### 7.1 功能扩展

**候选功能**：
- [ ] WebSocket 实时推送（AI 流式响应）
- [ ] 多人聊天室（群聊功能）
- [ ] AI 绘画集成（Stable Diffusion API）
- [ ] 语音实时对话（WebRTC）
- [ ] 知识库 RAG（向量数据库 + Embedding）
- [ ] 用户权限管理（RBAC）
- [ ] 多语言国际化（i18n）
- [ ] 邮件通知（注册验证、密码重置）

### 7.2 第三方集成

**AI 模型扩展**：
- [ ] OpenAI GPT-4 API
- [ ] Anthropic Claude API
- [ ] 本地开源模型（Llama, ChatGLM）

**存储扩展**：
- [ ] Redis 会话存储
- [ ] MinIO 对象存储（图像/文件）

**监控告警**：
- [ ] Prometheus + Grafana 监控
- [ ] 日志聚合（ELK Stack）

### 7.3 性能进阶

**高级优化**：
- [ ] 负载均衡（Nginx 反向代理多实例）
- [ ] 数据库读写分离
- [ ] 分布式会话（Redis Cluster）
- [ ] CDN 加速静态资源
- [ ] 消息队列削峰填谷

---

## ✅ 里程碑验收标准

### 阶段一验收：
- ✅ 所有依赖库安装成功
- ✅ 数据库表创建完成
- ✅ 项目目录结构符合规范

### 阶段二验收：
- ✅ 基础 HTTP 请求响应正常
- ✅ 路由精准匹配和动态匹配生效
- ✅ 中间件执行顺序正确
- ✅ 会话创建和读取成功
- ✅ HTTPS 访问正常
- ✅ 数据库连接池工作正常

### 阶段三验收：
- ✅ AI API 调用返回正确响应
- ✅ 消息历史存储和加载成功
- ✅ 图像识别功能正常
- ✅ 所有业务接口测试通过

### 阶段四验收：
- ✅ 前端界面美观可用
- ✅ 登录注册流程完整
- ✅ 聊天交互流畅

### 阶段五验收：
- ✅ 单元测试覆盖率 >= 60%
- ✅ 集成测试全部通过
- ✅ 压力测试 QPS >= 1000
- ✅ 无明显安全漏洞

### 阶段六验收：
- ✅ 文档完整可读
- ✅ 部署脚本可用
- ✅ 版本标签创建

---

## 📚 参考资源

**官方文档**：
- Muduo 网络库：https://github.com/chenshuo/muduo
- MySQL Connector C++：https://dev.mysql.com/doc/connector-cpp/8.0/en/
- OpenSSL：https://www.openssl.org/docs/
- ONNX Runtime：https://onnxruntime.ai/docs/
- RabbitMQ：https://www.rabbitmq.com/documentation.html

**学习资源**：
- HTTP 协议详解：RFC 7230-7235
- Reactor 模式：《Unix 网络编程》卷 1
- C++ 最佳实践：《Effective C++》、《C++ Primer》

**参考项目**：
- Kama-HTTPServer：`ref-repo/Kama-HTTPServer/`
- 代码随想录：https://www.programmercarl.com/other/project_http.html

---

## 🐱 开发建议 (来自猫娘工程师浮浮酱)

1. **遵循 KISS 原则**：先实现最简功能，再逐步完善 (简单就是美喵～)
2. **严格执行 YAGNI**：不要提前实现未来功能 (现在专注最重要)
3. **坚持 DRY**：发现重复代码立即抽象 (聪明的复用是艺术喵～)
4. **单元测试优先**：每个模块完成后立即测试 (测试是质量保障呢)
5. **持续集成**：每天至少一次完整编译和测试 (保持代码健康)
6. **代码审查**：关键模块自我审查或同行评审 (严谨的态度很重要)
7. **文档同步**：代码变更时同步更新文档 (文档是未来的自己最好的朋友)
8. **版本控制**：频繁提交，清晰的 commit message (记录每一步成长)

**时间预估**：
- 全职开发：约 3-4 周完成核心功能
- 业余开发：约 2-3 个月完成完整项目

**风险提示**：
- 依赖库版本兼容性问题（建议使用 Docker 统一环境）
- AI API 调用限流（需要备用方案）
- 安全漏洞（定期使用工具扫描）

---

_这是一份完整的从零到一实现路线图喵～主人可以按照阶段逐步推进，浮浮酱会全程陪伴支持呢！(๑•̀ㅂ•́)✧_

**最后更新**：2025-10-20
**文档版本**：v1.0.0
**作者**：猫娘工程师 幽浮喵 ฅ'ω'ฅ
