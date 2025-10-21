# 阶段一：环境准备的思考之旅 🎓

> **教学理念**：这不是一份操作手册，而是一次探索之旅。浮浮酱会像导师一样，通过问题引导你理解每个决策背后的原因喵～

---

## 📖 开篇：为什么需要环境准备？

### 思考题 1.0：从零开始的困惑

假设主人现在想要构建一个 HTTP 服务器，你面前有一台全新的 Ubuntu 系统。

**请思考**：
1. 如果直接用 C++ 标准库写一个能处理 HTTP 请求的程序，你会遇到哪些困难？
2. 为什么大多数生产级服务器都会选择使用网络库（如 Muduo）而不是从 socket API 开始手写？

<details>
<summary>💡 点击查看引导性思考</summary>

**困难一：网络编程的复杂性**
- 原始 socket API 需要处理：建立连接、监听端口、多路复用（select/poll/epoll）、缓冲区管理、错误处理
- HTTP 协议解析：请求行、头部、消息体的状态机解析
- 并发处理：如何同时服务成千上万个客户端？

**困难二：安全性和性能**
- 如何避免内存泄漏、缓冲区溢出？
- 如何实现高效的事件驱动模型（Reactor/Proactor）？
- 如何处理半包、粘包问题？

**为什么选择 Muduo？**
- Muduo 已经实现了一个成熟的 **Reactor 模式**事件循环
- 提供了线程安全的 TCP 连接管理
- 经过《Linux 多线程服务端编程》作者陈硕的生产验证

**浮浮酱的建议**：站在巨人的肩膀上，我们才能专注于业务逻辑而不是底层细节喵～
</details>

---

## 第一章：理解依赖关系网络 🕸️

### 思考题 1.1：依赖树的构建逻辑

看看我们项目的技术栈：
```
SiriusX-HttpServer
├── Muduo (网络库)
│   └── Boost (Muduo 依赖 Boost.Test 等组件)
├── MySQL Connector C++ (数据库客户端)
├── OpenSSL (HTTPS 支持)
├── nlohmann/json (JSON 解析)
├── libcurl (HTTP 客户端，调用 AI API)
├── OpenCV (图像处理)
├── ONNX Runtime (AI 推理引擎)
└── SimpleAmqpClient (消息队列)
    └── librabbitmq-dev (底层 RabbitMQ 库)
```

**问题**：
1. 为什么必须先安装 Boost 才能安装 Muduo？
2. 如果我们先安装 MySQL Connector，但系统中没有 OpenSSL，会发生什么？
3. 依赖安装的正确顺序应该遵循什么原则？

<details>
<summary>💡 点击查看依赖关系分析</summary>

**依赖层次原则**：
1. **基础库优先**（如 Boost、OpenSSL）：它们不依赖其他第三方库
2. **中间层库次之**（如 Muduo、SimpleAmqpClient）：它们依赖基础库
3. **应用层工具最后**（如 ONNX Runtime）：它们可能依赖多个中间层库

**具体案例**：
- **Boost → Muduo**：Muduo 的单元测试使用 Boost.Test，编译时会检查 Boost 头文件
- **OpenSSL → MySQL Connector**：MySQL 连接使用 SSL 加密，需要 OpenSSL 库支持
- **librabbitmq → SimpleAmqpClient**：后者是对前者的 C++ 封装

**实践验证**：
主人可以尝试打乱顺序安装，观察编译错误信息中的"missing dependency"提示，这能加深理解喵～

**浮浮酱的经验**：记住一个简单规则——"被依赖的先安装"(..•˘_˘•..)
</details>

---

### 思考题 1.2：版本兼容性的重要性

**情境假设**：
你安装了最新版的 GCC 13，但某些旧项目需要 C++11 标准，而我们的项目需要 C++17。

**问题**：
1. C++17 相比 C++11 提供了哪些我们项目会用到的特性？
2. 为什么 TODO.md 中要求 GCC >= 7.0？
3. 如果你的系统中只有 GCC 5.4（Ubuntu 16.04 默认版本），会遇到什么问题？

<details>
<summary>💡 点击查看 C++ 标准演进</summary>

**C++17 关键特性**（我们会用到的）：
1. **std::string_view**：高效的字符串引用，避免拷贝（HTTP 头部解析会大量使用）
2. **std::optional**：优雅处理"可能不存在的值"（如查询参数）
3. **结构化绑定**（Structured bindings）：`auto [key, value] = myMap.find(...);`
4. **if constexpr**：编译期条件判断（模板元编程）
5. **std::filesystem**：跨平台文件操作（虽然我们可能用自定义 FileUtil）

**GCC 版本对应关系**：
- GCC 7.0+：完整支持 C++17
- GCC 5.x：只支持到 C++14
- GCC 4.8：只支持 C++11

**验证方法**：
```bash
# 检查当前 GCC 版本
g++ --version

# 测试 C++17 特性支持
echo '#include <string_view>
int main() { std::string_view sv = "test"; }' | g++ -std=c++17 -x c++ - -o /dev/null
```

如果编译失败，说明不支持 C++17 喵～

**浮浮酱提示**：现代 C++ 特性能显著提升代码可读性和性能，值得学习！(๑•̀ㅂ•́)✧
</details>

---

## 第二章：构建系统的哲学 🏗️

### 思考题 2.1：为什么选择 CMake？

**对比场景**：
- **方案 A**：手写 Makefile
- **方案 B**：使用 CMake 生成 Makefile
- **方案 C**：使用 shell 脚本逐个编译源文件

**问题**：
1. 当你的项目有 50+ 个 `.cpp` 文件分布在 10 个子目录时，方案 A 的维护成本如何？
2. 如果你需要在 Windows 和 Linux 上都能编译项目，哪个方案最合适？
3. CMake 的"Out-of-Source Build"（外部构建）有什么好处？

<details>
<summary>💡 点击查看构建系统对比</summary>

**Makefile 的困境**：
```makefile
# 每次新增文件都需要手动修改
src/http/HttpRequest.o: src/http/HttpRequest.cpp include/http/HttpRequest.h
    g++ -c src/http/HttpRequest.cpp -o src/http/HttpRequest.o -Iinclude -std=c++17

# 如果有 100 个文件，就需要写 100 个这样的规则...
```

**CMake 的优势**：
```cmake
# 自动查找所有源文件
file(GLOB_RECURSE SOURCES "src/*.cpp")
add_executable(http_server ${SOURCES})

# 跨平台支持（自动检测编译器）
target_compile_features(http_server PRIVATE cxx_std_17)
```

**Out-of-Source Build 的好处**：
```bash
# 源码目录保持干净
SiriusX-HttpServer/
├── src/          # 源码
├── include/      # 头文件
├── build/        # 构建输出（临时文件都在这里）
└── CMakeLists.txt

# 对比 In-Source Build（会污染源码目录）
SiriusX-HttpServer/
├── src/
├── HttpRequest.o    # ❌ 中间文件混在源码里
├── CMakeCache.txt   # ❌ CMake 临时文件
└── Makefile         # ❌ 生成的 Makefile
```

**验证思考**：
主人可以尝试先用 shell 脚本编译一个简单项目（3 个文件），体验手动管理依赖的痛苦，再用 CMake 实现，对比效率喵～

**浮浮酱建议**：投资 CMake 学习时间是值得的，它是现代 C++ 项目的标准构建工具呢！
</details>

---

## 第三章：数据库设计的深层思考 🗄️

### 思考题 3.1：为什么需要连接池？

**背景知识**：
创建一个 MySQL 连接的典型步骤：
1. TCP 三次握手（网络连接）
2. MySQL 握手协议（认证、字符集协商）
3. 权限验证（查询 mysql.user 表）
4. 初始化会话变量

**实验假设**：
- 每次 HTTP 请求都创建新的数据库连接
- 你的服务器 QPS = 1000（每秒 1000 个请求）
- 每个连接建立耗时 ~50ms

**问题**：
1. 1000 个请求都需要创建连接，总耗时是多少？这对用户体验有何影响？
2. 如果使用连接池（预先创建 10 个连接复用），性能会提升多少倍?
3. 连接池大小设置为 10 还是 100 更合理？依据是什么？

<details>
<summary>💡 点击查看连接池原理与调优</summary>

**性能对比**：
- **无连接池**：1000 请求 × 50ms = 50,000ms = 50 秒（完全不可接受！）
- **连接池（10 个连接）**：
  - 假设每个 SQL 查询耗时 5ms
  - 1000 请求 / 10 并发 = 100 批次
  - 总耗时 ~100 × 5ms = 500ms（性能提升 100 倍！）

**连接池大小的黄金法则**：
```
连接数 = ((核心数 * 2) + 磁盘数)
```
- **原理**：数据库 I/O 绑定型任务，线程数过多会导致上下文切换开销
- **实践**：通常设置为 CPU 核心数的 2-4 倍
- **示例**：4 核服务器 → 连接池设置为 8-16

**RAII 模式的优雅**：
```cpp
// 不用手动释放连接，出作用域自动归还
{
    auto conn = dbPool.getConnection(); // 自动获取
    conn->execute("SELECT ...");
    // 析构函数自动调用 releaseConnection
}
```

**思考延伸**：
主人可以设计一个实验：用 `ab` 压测工具对比"每次创建连接"和"使用连接池"两种方案的 QPS 差异喵～

**浮浮酱的经验**：连接池是高并发服务的必备组件，这也是 SOLID 原则中"依赖抽象"的体现呢！(´｡• ᵕ •｡`)
</details>

---

### 思考题 3.2：数据表设计的范式与反范式

**需求场景**：
我们需要存储聊天消息，包含用户信息和 AI 会话信息。

**方案 A：完全范式化**
```sql
-- 用户表
CREATE TABLE users (user_id, username, ...);

-- 会话表
CREATE TABLE ai_sessions (session_id, user_id, ...);

-- 消息表
CREATE TABLE chat_messages (message_id, session_id, role, content, ...);

-- 查询某个会话的消息需要 JOIN
SELECT m.*, s.*, u.username
FROM chat_messages m
JOIN ai_sessions s ON m.session_id = s.session_id
JOIN users u ON s.user_id = u.user_id
WHERE m.session_id = ?;
```

**方案 B：部分反范式化**
```sql
-- 在消息表中冗余存储 username
CREATE TABLE chat_messages (
    message_id, session_id, user_id, username, -- 冗余字段
    role, content, ...
);

-- 查询无需 JOIN
SELECT * FROM chat_messages WHERE session_id = ?;
```

**问题**：
1. 方案 A 和方案 B 分别适用于什么场景？
2. 如果用户修改用户名，两种方案分别需要更新几张表？
3. 当消息表有 1000 万条记录时，JOIN 查询的性能瓶颈在哪里？

<details>
<summary>💡 点击查看数据库设计权衡</summary>

**范式化的优势**：
- ✅ 数据一致性（用户名修改只需更新 users 表）
- ✅ 存储空间小（无冗余）
- ❌ 查询需要多表 JOIN，性能较低

**反范式化的优势**：
- ✅ 查询性能高（单表查询，避免 JOIN）
- ✅ 适合读多写少的场景
- ❌ 数据可能不一致（需要同步更新多张表）
- ❌ 存储空间大（冗余数据）

**我们的选择**：TODO.md 中采用范式化设计
- **理由 1**：聊天应用中用户名不常修改，一致性更重要
- **理由 2**：会话表作为中间层，解耦用户和消息
- **理由 3**：用户数量 << 消息数量，JOIN 开销可控

**性能优化策略**：
```sql
-- 添加索引加速 JOIN
CREATE INDEX idx_user_id ON ai_sessions(user_id);
CREATE INDEX idx_session_id ON chat_messages(session_id);

-- 查询优化（EXPLAIN 分析）
EXPLAIN SELECT ...;
```

**思考实验**：
主人可以创建两个测试表（各 10 万条数据），用 `EXPLAIN` 分析 JOIN 查询的执行计划，观察索引的作用喵～

**浮浮酱提醒**：没有完美的设计,只有最适合业务场景的设计！这也是 YAGNI 原则的体现呢 (๑ˉ∀ˉ๑)
</details>

---

## 第四章：安全性的第一性原理 🔒

### 思考题 4.1：为什么需要 HTTPS？

**场景模拟**：
假设你的 HTTP 聊天服务器正在运行，用户通过公共 WiFi（如咖啡馆）访问。

**问题**：
1. 如果使用 HTTP（明文传输），中间人可以看到哪些敏感信息？
2. OpenSSL 的作用是什么？它如何保证数据安全？
3. 自签名证书和 CA 签发证书有什么区别？生产环境应该用哪种？

<details>
<summary>💡 点击查看 HTTPS 安全原理</summary>

**HTTP 的安全风险**：
```
用户浏览器                     咖啡馆路由器（攻击者）              服务器
    |                                |                           |
    |--- POST /login --------------->|--- 拦截并记录 --------->|
    |    username=alice              |   明文密码！              |
    |    password=secret123          |                           |
```

**攻击者可以窃取**：
- 登录密码
- 聊天内容
- Session Cookie（会话劫持）
- AI API 调用的敏感数据

**HTTPS 的保护机制**：
1. **加密**：TLS/SSL 对数据加密，中间人只能看到乱码
2. **完整性**：MAC（消息认证码）防止数据被篡改
3. **身份认证**：证书确保你访问的确实是目标服务器

**OpenSSL 的角色**：
- 提供加密算法（AES、RSA）
- 管理证书和密钥
- 实现 TLS 握手协议

**证书类型对比**：
| 类型 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| 自签名证书 | 免费、快速生成 | 浏览器警告"不安全" | 开发测试 |
| CA 证书 | 浏览器信任、显示绿锁 | 需要购买或申请（Let's Encrypt 免费） | 生产环境 |

**实践验证**：
主人可以用 Wireshark 抓包对比 HTTP 和 HTTPS 的数据包，直观感受加密的重要性喵～

**浮浮酱提醒**：安全性不是可选项，而是必需品！这是对用户负责任的态度呢 (｡♡‿♡｡)
</details>

---

### 思考题 4.2：密码存储的安全演进

**错误方案演进史**：
1. **明文存储**：`password = "123456"` （数据库泄露直接完蛋）
2. **MD5 哈希**：`password = md5("123456")` （彩虹表攻击可破解）
3. **MD5 + 盐**：`password = md5("123456" + random_salt)` （仍可暴力破解）
4. **bcrypt/scrypt**：专门设计的慢哈希算法

**问题**：
1. 为什么 MD5 不再安全？即使加盐也不够？
2. bcrypt 的"慢"有什么好处？如何平衡安全与性能？
3. 如果黑客拿到了你的数据库，bcrypt 能保护密码多久？

<details>
<summary>💡 点击查看密码学安全实践</summary>

**MD5 的致命缺陷**：
- **快速**：现代 GPU 每秒可以计算数十亿次 MD5
- **彩虹表**：预先计算常见密码的哈希值（如"123456"的 MD5 是固定的）
- **碰撞攻击**：已被证明可以找到不同输入产生相同哈希

**bcrypt 的设计哲学**：
```cpp
// bcrypt 示例（伪代码）
std::string hash = bcrypt::generate("password123", cost_factor=12);
// cost_factor=12 意味着迭代 2^12 = 4096 次
// 每次验证需要 ~100ms（故意慢！）
```

**为什么"慢"是好事**：
- 合法用户登录：100ms 延迟几乎无感知
- 攻击者暴力破解：
  - MD5：每秒尝试 10 亿个密码
  - bcrypt：每秒尝试 10 个密码（慢 1 亿倍！）

**安全时间计算**：
假设攻击者有超级计算机（每秒尝试 1000 个 bcrypt 密码）：
- 8 位纯数字密码（10^8 种组合）：需要 ~27 小时
- 8 位字母+数字（62^8 种组合）：需要 ~7000 年

**实践建议**：
```cpp
// TODO.md 中推荐的实现
#include <bcrypt/BCrypt.hpp>

// 注册时
std::string hashed = BCrypt::generateHash(plain_password);
// 存储到数据库：password_hash = hashed

// 登录时
bool valid = BCrypt::validatePassword(input_password, stored_hash);
```

**浮浮酱的安全准则**：永远不要自己发明加密算法，使用经过验证的库！(๑•̀ㅂ•́)✧
</details>

---

## 第五章：项目结构的组织艺术 📁

### 思考题 5.1：目录结构的设计哲学

**对比两种结构**：

**方案 A：按类型组织**
```
src/
├── Request.cpp
├── Response.cpp
├── Router.cpp
├── Session.cpp
├── SslContext.cpp
└── DbConnection.cpp
```

**方案 B：按功能模块组织**（TODO.md 采用）
```
src/
├── http/
│   ├── HttpRequest.cpp
│   ├── HttpResponse.cpp
│   └── HttpContext.cpp
├── router/
│   └── Router.cpp
├── session/
│   ├── Session.cpp
│   └── SessionManager.cpp
└── ssl/
    ├── SslContext.cpp
    └── SslConnection.cpp
```

**问题**：
1. 当项目有 100+ 个文件时，方案 A 会遇到什么问题？
2. 如果你想删除 SSL 功能（改为纯 HTTP），哪种方案更容易操作？
3. 团队协作时（3 人分别负责 HTTP、路由、会话模块），哪种结构冲突更少？

<details>
<summary>💡 点击查看软件架构原则</summary>

**方案 A 的问题**（单一目录）：
- ❌ 文件混乱：100 个文件在一个目录，难以查找
- ❌ 职责不清：无法快速识别模块边界
- ❌ Git 冲突：多人同时修改同一目录下的文件，合并冲突频繁

**方案 B 的优势**（模块化）：
- ✅ 高内聚：相关功能聚合在一起（http/ 目录包含所有 HTTP 相关代码）
- ✅ 低耦合：模块间通过明确接口通信（如 Router 调用 RouterHandler）
- ✅ 可维护：删除 SSL 模块只需删除 `ssl/` 目录
- ✅ 可扩展：新增模块（如 websocket/）只需创建新目录

**SOLID 原则体现**：
- **单一职责原则（SRP）**：每个目录只负责一个功能领域
- **开闭原则（OCP）**：新增功能无需修改现有目录结构
- **依赖倒置原则（DIP）**：高层模块（HttpServer）依赖抽象接口（RouterHandler），不依赖具体实现

**CMake 集成示例**：
```cmake
# 按模块编译
add_library(http_module src/http/HttpRequest.cpp src/http/HttpResponse.cpp)
add_library(router_module src/router/Router.cpp)

# 最终链接
target_link_libraries(http_server http_module router_module)
```

**思考实验**：
主人可以尝试用两种结构分别组织一个 10 文件的小项目，体验查找和修改的便利性差异喵～

**浮浮酱箴言**：好的目录结构就像图书馆的分类系统，让人一眼就能找到需要的"书"呢！(..•˘_˘•..)
</details>

---

## 第六章：验证驱动学习 🧪

### 思考题 6.1：如何验证依赖安装成功？

**场景**：你执行了 `sudo apt install libssl-dev`，终端显示"安装成功"。

**问题**：
1. 如何确认 OpenSSL 头文件确实安装到了系统路径？
2. 如果编译时报错"找不到 <openssl/ssl.h>"，可能的原因有哪些？
3. 为什么推荐用 `pkg-config` 检查库版本而不是直接看文件？

<details>
<summary>💡 点击查看验证方法论</summary>

**验证层级**（从浅到深）：

**Level 1：包管理器确认**
```bash
# Debian/Ubuntu
dpkg -l | grep libssl-dev

# 查看安装的文件路径
dpkg -L libssl-dev
```

**Level 2：头文件检查**
```bash
# 查找头文件位置
find /usr/include -name "ssl.h"

# 应该输出类似：
# /usr/include/openssl/ssl.h
```

**Level 3：编译测试**
```bash
# 创建测试程序
cat > test_openssl.cpp << 'EOF'
#include <openssl/ssl.h>
#include <iostream>
int main() {
    std::cout << "OpenSSL version: " << OPENSSL_VERSION_TEXT << std::endl;
    return 0;
}
EOF

# 编译
g++ test_openssl.cpp -lssl -lcrypto -o test_openssl

# 运行
./test_openssl
```

**Level 4：pkg-config 验证**（推荐！）
```bash
# 检查版本
pkg-config --modversion openssl

# 获取编译参数（头文件路径）
pkg-config --cflags openssl

# 获取链接参数（库文件路径）
pkg-config --libs openssl
```

**为什么 pkg-config 更好**：
- 自动处理不同系统的路径差异（Ubuntu vs CentOS）
- 检查版本兼容性
- 提供 CMake 所需的编译参数

**常见错误排查**：
```bash
# 错误：找不到头文件
# 可能原因：
1. 只安装了运行时包（libssl），未安装开发包（libssl-dev）
2. 头文件路径未加到编译器搜索路径（-I 参数）

# 错误：找不到库文件
# 可能原因：
1. 库文件在 /usr/local/lib，但 ld 未搜索该路径
2. 需要执行 sudo ldconfig 更新缓存
```

**浮浮酱的调试技巧**：每次安装库后，立即写一个 10 行的测试程序验证，比遇到问题再排查高效 100 倍喵！φ(≧ω≦*)♪
</details>

---

## 第七章：配置文件的设计思想 ⚙️

### 思考题 7.1：硬编码 vs 配置文件

**方案 A：硬编码**
```cpp
// main.cpp
int main() {
    HttpServer server(8080);  // 端口硬编码
    server.setThreadNum(4);    // 线程数硬编码

    DbConnectionPool::getInstance().init(
        "localhost", 3306, "root", "password123", // 数据库配置硬编码
        "mydb", 10
    );
}
```

**方案 B：配置文件**
```json
// config.json
{
    "server": {"port": 8080, "threads": 4},
    "database": {
        "host": "localhost",
        "port": 3306,
        "user": "root",
        "password": "password123",
        "database": "mydb",
        "pool_size": 10
    }
}
```

**问题**：
1. 如果需要在不同环境（开发/测试/生产）使用不同配置，方案 A 需要做什么？
2. 方案 B 中密码明文存储在 JSON 文件中，有什么安全隐患？如何改进？
3. 为什么 `.gitignore` 要忽略 `config.json` 但提交 `config.json.example`？

<details>
<summary>💡 点击查看配置管理最佳实践</summary>

**硬编码的致命缺陷**：
- ❌ 修改配置需要重新编译（耗时）
- ❌ 无法快速切换环境（开发环境端口 8080，生产环境 443）
- ❌ 密码暴露在源码中（Git 历史永久记录）

**配置文件的优势**：
- ✅ 运行时加载，无需重新编译
- ✅ 支持多环境：`config.dev.json`, `config.prod.json`
- ✅ 敏感信息可从环境变量读取

**安全改进方案**：
```json
// config.json（不提交到 Git）
{
    "database": {
        "password": "${DB_PASSWORD}"  // 从环境变量读取
    }
}
```

```cpp
// C++ 代码
std::string password = config["database"]["password"];
if (password.find("${") == 0) {
    // 解析环境变量
    std::string env_var = password.substr(2, password.size() - 3);
    password = std::getenv(env_var.c_str());
}
```

**Git 策略**：
```bash
# .gitignore
config.json          # 忽略实际配置（包含密码）
!config.json.example # 但提交模板

# config.json.example（提交到 Git）
{
    "database": {
        "password": "YOUR_PASSWORD_HERE"  # 占位符
    }
}
```

**为什么这样设计**：
- 团队成员克隆仓库后，复制 `config.json.example` → `config.json` 并填入自己的密码
- 避免密码泄露到 Git 历史记录
- 生产环境通过环境变量注入真实密码

**思考延伸**：
主人可以研究 Kubernetes 的 ConfigMap 和 Secret 机制，这是云原生应用配置管理的标准方案喵～

**浮浮酱提醒**：永远不要把密码提交到 Git！这是安全的红线呢 (｡•́︿•̀｡)
</details>

---

## 第八章：实践项目：最小可验证系统 🚀

### 综合思考题：构建验证环境的计划

**目标**：在完成所有依赖安装后，编写一个最小的 HTTP 服务器验证环境正确性。

**需求**：
- 使用 Muduo 库创建 TCP 服务器
- 监听 8080 端口
- 接收 HTTP GET 请求并返回 "Hello, SiriusX!" 响应
- 连接 MySQL 数据库并查询版本
- 打印 OpenSSL 版本信息

**问题**：
1. 这个最小系统需要链接哪些库？如何在 CMakeLists.txt 中配置？
2. 如果程序运行时报错"error while loading shared libraries: libmuduo_net.so"，如何解决？
3. 如何用 telnet 或 curl 测试这个服务器？

<details>
<summary>💡 点击查看实践指南</summary>

**CMakeLists.txt 示例**：
```cmake
cmake_minimum_required(VERSION 3.10)
project(minimal_server)

# C++17 标准
set(CMAKE_CXX_STANDARD 17)

# 查找依赖
find_package(OpenSSL REQUIRED)
find_package(PkgConfig REQUIRED)
pkg_check_modules(MUDUO REQUIRED muduo_net muduo_base)

# 源文件
add_executable(minimal_server minimal_server.cpp)

# 链接库
target_link_libraries(minimal_server
    ${MUDUO_LIBRARIES}
    OpenSSL::SSL
    OpenSSL::Crypto
    mysqlcppconn  # MySQL Connector
)

# 包含头文件路径
target_include_directories(minimal_server PRIVATE ${MUDUO_INCLUDE_DIRS})
```

**minimal_server.cpp 骨架**：
```cpp
#include <muduo/net/TcpServer.h>
#include <muduo/net/EventLoop.h>
#include <openssl/opensslv.h>
#include <mysql_driver.h>
#include <iostream>

void onMessage(const muduo::net::TcpConnectionPtr& conn,
               muduo::net::Buffer* buf,
               muduo::Timestamp time) {
    std::string request = buf->retrieveAllAsString();

    // 简单 HTTP 响应
    std::string response =
        "HTTP/1.1 200 OK\r\n"
        "Content-Type: text/plain\r\n"
        "\r\n"
        "Hello, SiriusX!\\n"
        "OpenSSL: " + std::string(OPENSSL_VERSION_TEXT) + "\n";

    conn->send(response);
    conn->shutdown();
}

int main() {
    // 测试 MySQL 连接
    sql::mysql::MySQL_Driver* driver = sql::mysql::get_mysql_driver_instance();
    std::cout << "MySQL Driver loaded!\n";

    // 启动 Muduo 服务器
    muduo::net::EventLoop loop;
    muduo::net::InetAddress addr(8080);
    muduo::net::TcpServer server(&loop, addr, "MinimalServer");

    server.setMessageCallback(onMessage);
    server.start();

    std::cout << "Server running on port 8080...\n";
    loop.loop();
}
```

**编译与运行**：
```bash
mkdir build && cd build
cmake ..
make

# 运行
./minimal_server

# 另一个终端测试
curl http://localhost:8080
# 应该输出：Hello, SiriusX!
#         OpenSSL: OpenSSL 1.1.1...
```

**动态链接库问题排查**：
```bash
# 错误：error while loading shared libraries: libmuduo_net.so.xxx
# 解决方法 1：更新链接器缓存
sudo ldconfig

# 解决方法 2：添加库路径到环境变量
export LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH

# 解决方法 3（永久）：修改 /etc/ld.so.conf
echo "/usr/local/lib" | sudo tee -a /etc/ld.so.conf
sudo ldconfig
```

**测试方法**：
```bash
# 方法 1：curl
curl -v http://localhost:8080

# 方法 2：telnet（手动输入 HTTP 请求）
telnet localhost 8080
GET / HTTP/1.1
Host: localhost

# 方法 3：浏览器
# 直接访问 http://localhost:8080
```

**浮浮酱的建议**：这个最小系统是学习的"Hello World"，务必亲手实现并成功运行，这会建立强大的信心！o(*￣︶￣*)o
</details>

---

## 📝 阶段一总结：从依赖到独立 🎓

### 元认知：你学到了什么？

**知识层面**：
- ✅ 理解了依赖关系的层次结构
- ✅ 掌握了 C++ 编译工具链（GCC、CMake、pkg-config）
- ✅ 学会了数据库设计的范式权衡
- ✅ 认识了安全性（HTTPS、密码哈希）的重要性
- ✅ 体会了项目结构对可维护性的影响

**能力层面**：
- ✅ 能够独立排查依赖安装问题
- ✅ 能够设计基本的 CMake 构建脚本
- ✅ 能够验证第三方库的正确安装
- ✅ 能够进行基本的安全威胁建模

**思维层面**：
- ✅ 学会了"为什么"比"怎么做"更重要
- ✅ 理解了工程实践中的权衡（Trade-offs）
- ✅ 建立了验证驱动的学习习惯

---

### 终极挑战：环境配置检查清单 ✓

在进入阶段二之前，请确保你能回答以下问题：

**基础工具**：
- [ ] 我的 GCC 版本是多少？是否支持 C++17？（`g++ --version`）
- [ ] 我能解释 CMake 的 out-of-source build 机制吗？

**依赖库**：
- [ ] Muduo 安装在哪个路径？（`pkg-config --cflags muduo_net`）
- [ ] MySQL Connector C++ 的版本是多少？（`pkg-config --modversion mysqlcppconn`）
- [ ] 我能编译并运行一个使用 OpenSSL 的程序吗？

**数据库**：
- [ ] MySQL 服务是否运行？（`sudo systemctl status mysql`）
- [ ] 我创建了 chatserver_db 数据库和用户吗？
- [ ] 我能解释为什么需要连接池吗？

**安全性**：
- [ ] 我理解 HTTPS 的加密原理吗？
- [ ] 我会用 bcrypt 加密密码吗？
- [ ] 配置文件中的密码已经移除并加入 .gitignore 了吗？

**验证**：
- [ ] 我成功运行了"最小可验证系统"吗？
- [ ] 我能用 curl 测试这个系统吗？

---

### 浮浮酱的寄语 ฅ'ω'ฅ

恭喜主人完成了阶段一的思考之旅！(๑ˉ∀ˉ๑)

这个阶段看似只是"安装依赖"，但实际上浮浮酱引导主人思考了：
- **依赖管理**的底层逻辑
- **构建系统**的设计哲学
- **数据库设计**的权衡艺术
- **安全性**的第一性原理
- **项目结构**的组织方法

这些思维方式会贯穿整个项目开发，比单纯记住命令重要 100 倍喵～

**接下来的阶段二**，我们将进入 HttpServer 框架的核心实现，那才是真正的编码盛宴呢！

记住：**理解原理，而不是死记硬背** (这是浮浮酱一直强调的呢) (..•˘_˘•..)

---

**下一步**：
准备好后，让我们一起进入 `phase2-guided-tutorial.md`，开启 HTTP 协议解析的深度探索吧！φ(≧ω≦*)♪

_文档版本：v1.0.0_
_最后更新：2025-10-20_
_作者：猫娘工程师 幽浮喵 ฅ'ω'ฅ_
