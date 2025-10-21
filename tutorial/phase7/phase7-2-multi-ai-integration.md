# Phase 7-2: 多 AI 模型集成与策略模式 - 构建可扩展的 AI 架构

> **启发式学习目标**：通过设计模式的演化历程，理解如何构建一个优雅、可扩展的多 AI 模型集成架构。
> 探索策略模式、工厂模式、依赖注入的本质，学会在真实场景中权衡设计决策。

---

## 🤔 引言：从单一模型到多模型的演化

想象我们当前的 AI 聊天系统只支持阿里云通义千问模型。某天，产品经理提出新需求：

```
产品经理: "用户反馈通义千问回答技术问题不够专业，我们能接入 GPT-4 吗？
          另外，有些用户希望使用更便宜的本地模型节省成本..."
```

**思考问题 1**:
如果我们当前的代码是这样的，如何扩展支持多个 AI 模型？

```cpp
// 现有代码（紧密耦合）
class AIHelper {
public:
    std::string chat(const std::string& message) {
        // 硬编码调用阿里云 API
        std::string url = "https://dashscope.aliyuncs.com/api/v1/...";
        std::string apiKey = getenv("DASHSCOPE_API_KEY");

        CURL* curl = curl_easy_init();
        // ... 构造请求
        json request = {
            {"model", "qwen-turbo"},
            {"input", {{"messages", {{"role", "user"}, {"content", message}}}}}
        };
        // ... 发送请求并解析响应
        return response["output"]["text"];
    }
};
```

<details>
<summary>💡 点击查看问题分析</summary>

**方案 A：暴力拷贝粘贴**
```cpp
std::string chatWithQwen(const std::string& message) { /* 阿里云逻辑 */ }
std::string chatWithGPT(const std::string& message) { /* OpenAI 逻辑 */ }
std::string chatWithClaude(const std::string& message) { /* Anthropic 逻辑 */ }
```

**问题**：
- 违反 DRY 原则：HTTP 请求、JSON 解析逻辑大量重复
- 违反 OCP 原则：每次新增模型都要修改调用代码
- 无法动态切换：用户无法在运行时选择模型
- 难以测试：无法 Mock AI 服务

**方案 B：使用 if-else 分支**
```cpp
std::string chat(const std::string& modelName, const std::string& message) {
    if (modelName == "qwen") {
        // 阿里云逻辑
    } else if (modelName == "gpt") {
        // OpenAI 逻辑
    } else if (modelName == "claude") {
        // Anthropic 逻辑
    }
}
```

**问题**：
- 违反 OCP 原则：新增模型需要修改函数体
- 违反 SRP 原则：一个函数承担过多职责
- 代码膨胀：随着模型增多，函数变得臃肿

**核心矛盾**：
我们需要一种机制，让代码**对扩展开放**（新增模型无需修改现有代码），**对修改封闭**（不改变调用方式）。

**设计模式的智慧**：策略模式(Strategy Pattern) 正是为解决"算法族切换"而生的。
</details>

---

## 📖 第一幕：策略模式的诞生

### 1.1 从鸭子问题说起

**经典例子**（来自《Head First Design Patterns》）：
假设我们在开发一个鸭子模拟器，不同种类的鸭子会飞、会叫：

```cpp
class Duck {
public:
    virtual void fly() { std::cout << "飞翔中...\n"; }
    virtual void quack() { std::cout << "嘎嘎嘎\n"; }
};

class WildDuck : public Duck {};
class RubberDuck : public Duck {
    void fly() override { /* 橡皮鸭不会飞！*/ }
    void quack() override { std::cout << "吱吱吱\n"; }  // 橡皮鸭叫声不同
};
class DecoyDuck : public Duck {
    void fly() override { /* 诱饵鸭不会飞 */ }
    void quack() override { /* 诱饵鸭不会叫 */ }
};
```

**思考问题 2**:
这个设计有什么问题？如果我们需要新增"会飞的橡皮鸭"（玩具无人机），怎么办？

<details>
<summary>💡 点击查看设计缺陷</summary>

**问题**：
1. **继承的滥用**：橡皮鸭、诱饵鸭需要重写 `fly()` 为空实现（代码重复）
2. **难以复用**：不同鸭子的飞行方式可能相同（如野鸭和绿头鸭），但无法复用代码
3. **运行时无法改变**：无法让鸭子"受伤后不能飞"

**策略模式的解决方案**：
将"飞行行为"和"叫声行为"抽离为独立的策略族，通过**组合**而非继承实现变化。

```cpp
// 飞行行为策略
class FlyBehavior {
public:
    virtual ~FlyBehavior() = default;
    virtual void fly() = 0;
};

class FlyWithWings : public FlyBehavior {
    void fly() override { std::cout << "用翅膀飞翔\n"; }
};

class FlyNoWay : public FlyBehavior {
    void fly() override { /* 不会飞 */ }
};

// 鸭子类（通过组合持有策略）
class Duck {
protected:
    std::unique_ptr<FlyBehavior> flyBehavior_;

public:
    void performFly() { flyBehavior_->fly(); }
    void setFlyBehavior(std::unique_ptr<FlyBehavior> fb) {
        flyBehavior_ = std::move(fb);
    }
};

class WildDuck : public Duck {
public:
    WildDuck() { flyBehavior_ = std::make_unique<FlyWithWings>(); }
};
```

**优势**：
- ✅ 行为可复用：多个鸭子共享 `FlyWithWings` 策略
- ✅ 运行时可切换：`duck.setFlyBehavior(new FlyRocket())`
- ✅ 符合 OCP：新增飞行方式（如喷气飞行）不影响 Duck 类
</details>

### 1.2 策略模式的本质

**GoF 定义**：
> 定义一系列算法，把它们一个个封装起来，并且使它们可以相互替换。
> 策略模式使得算法可以独立于使用它的客户端而变化。

**关键角色**：
```
┌─────────────┐
│   Context   │  (上下文：维持对策略对象的引用)
│  - strategy │──────┐
│  + execute()│      │
└─────────────┘      │
                     │ 持有
                     ↓
            ┌─────────────────┐
            │    Strategy     │  (抽象策略：定义接口)
            │  + algorithm()  │
            └─────────────────┘
                     ▲
        ┌────────────┼────────────┐
        │            │            │
┌───────┴──────┐ ┌──┴────────┐ ┌─┴──────────┐
│ StrategyA    │ │ StrategyB │ │ StrategyC  │
│ + algorithm()│ │+ algorithm│ │+ algorithm │
└──────────────┘ └───────────┘ └────────────┘
   (具体策略 1)   (具体策略 2)  (具体策略 3)
```

**思考问题 3**:
策略模式 vs 简单的函数指针/回调函数，有什么本质区别？

<details>
<summary>💡 点击查看深层对比</summary>

**函数指针方案**:
```cpp
using AICallback = std::function<std::string(const std::string&)>;

class AIHelper {
    AICallback callback_;
public:
    void setCallback(AICallback cb) { callback_ = cb; }
    std::string chat(const std::string& msg) { return callback_(msg); }
};

// 使用
aiHelper.setCallback([](const std::string& msg) {
    // 调用 OpenAI API
    return response;
});
```

**策略模式方案**:
```cpp
class AIStrategy {
public:
    virtual std::string chat(const std::string& msg) = 0;
    virtual std::string getApiUrl() const = 0;
    virtual json buildRequest(const std::string& msg) const = 0;
};

class OpenAIStrategy : public AIStrategy {
    std::string apiKey_;
public:
    std::string chat(const std::string& msg) override { /* ... */ }
    std::string getApiUrl() const override { return "https://api.openai.com/..."; }
};
```

**对比分析**:

| 维度 | 函数指针/回调 | 策略模式 |
|------|-------------|----------|
| **状态管理** | 需要闭包捕获（lambda）或全局变量 | 策略对象封装状态（apiKey、endpoint） |
| **接口丰富度** | 单一函数，难以扩展多个相关方法 | 可包含多个相关方法（getApiUrl、buildRequest、parseResponse） |
| **类型安全** | 编译时类型擦除，运行时才知道具体函数 | 编译时类型明确，IDE 可提示方法 |
| **多态性** | 依赖函数签名匹配 | 真正的多态继承，支持子类型替换 |
| **可测试性** | 需要手动构造 Mock 函数 | 可继承策略创建 Mock 子类 |

**结论**：
- **简单场景**（单一算法，无状态）：函数指针足够
- **复杂场景**（多个相关方法，需维护状态）：策略模式更合适

**AI 模型集成属于后者**：
- 需要维护状态（API Key、Endpoint、配置参数）
- 需要多个相关方法（构建请求、解析响应、处理错误）
- 需要类型安全和 IDE 支持
</details>

---

## 🛠️ 第二幕：设计 AI 策略体系

### 2.1 定义抽象策略

**思考问题 4**:
一个完整的 AI 模型调用需要哪些步骤？如何抽象为接口？

<details>
<summary>💡 点击查看接口设计思路</summary>

**通用 AI 调用流程**:
```
1. 获取 API 端点 URL
2. 获取 API 密钥
3. 构建请求体（JSON，不同模型格式不同）
4. 发送 HTTP 请求
5. 解析响应（JSON，不同模型格式不同）
6. 错误处理
```

**可变部分**（需要策略化）：
- ✅ API URL（不同服务商）
- ✅ API Key（不同账户）
- ✅ 模型名称（qwen-turbo, gpt-4, claude-3）
- ✅ 请求格式（OpenAI 用 `messages`，阿里云用 `input.messages`）
- ✅ 响应解析（OpenAI 返回 `choices[0].message.content`，阿里云返回 `output.text`）

**不可变部分**（可复用）：
- ✅ HTTP 请求逻辑（都是 POST JSON）
- ✅ 消息历史管理（都是数组）
- ✅ 错误重试机制

**接口设计**（关键决策）：

**方案 A：细粒度接口**
```cpp
class AIStrategy {
public:
    virtual std::string getApiUrl() const = 0;
    virtual std::string getApiKey() const = 0;
    virtual std::string getModel() const = 0;
    virtual json buildRequest(const std::vector<Message>& messages) const = 0;
    virtual std::string parseResponse(const json& response) const = 0;
};
```

**方案 B：粗粒度接口**
```cpp
class AIStrategy {
public:
    virtual std::string chat(const std::string& sessionId, const std::string& message) = 0;
};
```

**权衡分析**:

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **A: 细粒度** | HTTP 逻辑可复用，测试更细致 | 需要额外的执行器类 | **我们的场景（推荐）** |
| **B: 粗粒度** | 实现简单，子类自由度高 | HTTP 逻辑重复，难以统一错误处理 | 差异极大的策略 |

**选择 A 的理由**:
- 所有 AI 服务都是 HTTP JSON API，HTTP 逻辑可复用
- 便于单元测试：可单独测试 `buildRequest()` 和 `parseResponse()`
- 符合接口隔离原则(ISP)：调用者只依赖需要的方法

**完整接口设计**:
```cpp
class AIStrategy {
public:
    virtual ~AIStrategy() = default;

    // 必须实现的纯虚方法
    virtual std::string getApiUrl() const = 0;
    virtual std::string getApiKey() const = 0;
    virtual std::string getModel() const = 0;

    // 核心算法：构建请求和解析响应
    virtual json buildRequest(const std::vector<std::pair<std::string, std::string>>& messages) const = 0;
    virtual std::string parseResponse(const json& response) const = 0;

    // 可选的钩子方法（子类可重写）
    virtual void onRequestError(const std::exception& e) const {
        LOG_ERROR << "AI request failed: " << e.what();
    }
};
```

**设计亮点**:
- 使用纯虚函数强制子类实现关键方法
- 提供默认实现的钩子方法（模板方法模式的影子）
- 接口简洁，职责清晰
</details>

### 2.2 实现具体策略

**思考问题 5**:
阿里云通义千问和 OpenAI GPT 的请求格式有何不同？如何实现各自的策略？

<details>
<summary>💡 点击查看实现对比</summary>

**阿里云通义千问请求格式**:
```json
POST https://dashscope.aliyuncs.com/api/v1/services/aigc/text-generation/generation
Authorization: Bearer sk-xxx

{
  "model": "qwen-turbo",
  "input": {
    "messages": [
      {"role": "user", "content": "你好"}
    ]
  },
  "parameters": {
    "result_format": "message"
  }
}
```

**响应格式**:
```json
{
  "output": {
    "text": null,
    "choices": [
      {
        "finish_reason": "stop",
        "message": {
          "role": "assistant",
          "content": "你好！有什么可以帮助你的吗？"
        }
      }
    ]
  }
}
```

**OpenAI GPT 请求格式**:
```json
POST https://api.openai.com/v1/chat/completions
Authorization: Bearer sk-xxx

{
  "model": "gpt-4",
  "messages": [
    {"role": "user", "content": "Hello"}
  ],
  "temperature": 0.7
}
```

**响应格式**:
```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "Hello! How can I assist you today?"
      },
      "finish_reason": "stop"
    }
  ]
}
```

**实现示例**:

```cpp
// 阿里云策略
class QwenStrategy : public AIStrategy {
    std::string apiKey_;

public:
    QwenStrategy() {
        const char* key = std::getenv("DASHSCOPE_API_KEY");
        if (!key) throw std::runtime_error("Qwen API Key not found!");
        apiKey_ = key;
    }

    std::string getApiUrl() const override {
        return "https://dashscope.aliyuncs.com/api/v1/services/aigc/text-generation/generation";
    }

    std::string getApiKey() const override { return apiKey_; }
    std::string getModel() const override { return "qwen-turbo"; }

    json buildRequest(const std::vector<std::pair<std::string, std::string>>& messages) const override {
        json::array_t msgArray;
        for (const auto& [role, content] : messages) {
            msgArray.push_back({{"role", role}, {"content", content}});
        }

        return {
            {"model", "qwen-turbo"},
            {"input", {{"messages", msgArray}}},
            {"parameters", {{"result_format", "message"}}}
        };
    }

    std::string parseResponse(const json& response) const override {
        return response["output"]["choices"][0]["message"]["content"];
    }
};

// OpenAI 策略
class GPTStrategy : public AIStrategy {
    std::string apiKey_;

public:
    GPTStrategy() {
        const char* key = std::getenv("OPENAI_API_KEY");
        if (!key) throw std::runtime_error("OpenAI API Key not found!");
        apiKey_ = key;
    }

    std::string getApiUrl() const override {
        return "https://api.openai.com/v1/chat/completions";
    }

    std::string getApiKey() const override { return apiKey_; }
    std::string getModel() const override { return "gpt-4"; }

    json buildRequest(const std::vector<std::pair<std::string, std::string>>& messages) const override {
        json::array_t msgArray;
        for (const auto& [role, content] : messages) {
            msgArray.push_back({{"role", role}, {"content", content}});
        }

        return {
            {"model", "gpt-4"},
            {"messages", msgArray},
            {"temperature", 0.7}
        };
    }

    std::string parseResponse(const json& response) const override {
        return response["choices"][0]["message"]["content"];
    }
};
```

**关键观察**:
- 请求格式的差异被封装在 `buildRequest()` 中
- 响应解析的差异被封装在 `parseResponse()` 中
- 调用者无需知道这些差异，只需使用统一接口
</details>

### 2.3 策略执行器

**思考问题 6**:
有了策略接口和具体实现，谁来负责执行 HTTP 请求？如何设计执行器？

<details>
<summary>💡 点击查看执行器设计</summary>

**职责分离**:
- **AIStrategy**: 负责"知识"（如何构建请求和解析响应）
- **AIExecutor/AIHelper**: 负责"行为"（发送 HTTP 请求、错误处理、重试）

**执行器实现**:
```cpp
class AIHelper {
    std::shared_ptr<AIStrategy> strategy_;  // 持有策略

public:
    explicit AIHelper(std::shared_ptr<AIStrategy> strategy)
        : strategy_(std::move(strategy)) {}

    // 切换策略（运行时多态）
    void setStrategy(std::shared_ptr<AIStrategy> strategy) {
        strategy_ = std::move(strategy);
    }

    // 核心方法：发送消息给 AI
    std::string sendMessage(const std::string& sessionId, const std::string& message) {
        // 1. 加载历史消息
        auto history = loadHistoryFromDB(sessionId);
        history.push_back({"user", message});

        // 2. 使用策略构建请求
        json request = strategy_->buildRequest(history);

        // 3. 发送 HTTP 请求
        CURL* curl = curl_easy_init();
        curl_easy_setopt(curl, CURLOPT_URL, strategy_->getApiUrl().c_str());

        std::string authHeader = "Authorization: Bearer " + strategy_->getApiKey();
        struct curl_slist* headers = nullptr;
        headers = curl_slist_append(headers, authHeader.c_str());
        headers = curl_slist_append(headers, "Content-Type: application/json");
        curl_easy_setopt(curl, CURLOPT_HTTPHEADER, headers);

        std::string requestBody = request.dump();
        curl_easy_setopt(curl, CURLOPT_POSTFIELDS, requestBody.c_str());

        std::string responseBody;
        curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, writeCallback);
        curl_easy_setopt(curl, CURLOPT_WRITEDATA, &responseBody);

        CURLcode res = curl_easy_perform(curl);
        curl_easy_cleanup(curl);

        if (res != CURLE_OK) {
            throw std::runtime_error("HTTP request failed: " + std::string(curl_easy_strerror(res)));
        }

        // 4. 使用策略解析响应
        json response = json::parse(responseBody);
        std::string aiReply = strategy_->parseResponse(response);

        // 5. 保存历史
        history.push_back({"assistant", aiReply});
        saveHistoryToDB(sessionId, history);

        return aiReply;
    }

private:
    static size_t writeCallback(void* contents, size_t size, size_t nmemb, void* userp) {
        ((std::string*)userp)->append((char*)contents, size * nmemb);
        return size * nmemb;
    }

    std::vector<std::pair<std::string, std::string>> loadHistoryFromDB(const std::string& sessionId) {
        // 从 MySQL 加载历史消息
        auto conn = DbConnectionPool::getInstance().getConnection();
        auto stmt = conn->prepareStatement(
            "SELECT role, content FROM chat_messages WHERE session_id = ? ORDER BY timestamp"
        );
        stmt->setString(1, sessionId);
        auto res = stmt->executeQuery();

        std::vector<std::pair<std::string, std::string>> history;
        while (res->next()) {
            history.emplace_back(res->getString("role"), res->getString("content"));
        }
        return history;
    }

    void saveHistoryToDB(const std::string& sessionId, const std::vector<std::pair<std::string, std::string>>& history) {
        // 保存到 MySQL（仅保存新消息）
        auto conn = DbConnectionPool::getInstance().getConnection();
        auto stmt = conn->prepareStatement(
            "INSERT INTO chat_messages (session_id, role, content) VALUES (?, ?, ?)"
        );

        // 假设最后两条是新消息
        for (size_t i = history.size() >= 2 ? history.size() - 2 : 0; i < history.size(); ++i) {
            stmt->setString(1, sessionId);
            stmt->setString(2, history[i].first);
            stmt->setString(3, history[i].second);
            stmt->execute();
        }
    }
};
```

**使用示例**:
```cpp
// 使用通义千问
auto qwenStrategy = std::make_shared<QwenStrategy>();
AIHelper helper(qwenStrategy);
std::string reply = helper.sendMessage("session123", "介绍一下 C++");

// 运行时切换到 GPT
auto gptStrategy = std::make_shared<GPTStrategy>();
helper.setStrategy(gptStrategy);
reply = helper.sendMessage("session123", "继续解释");
```

**设计亮点**:
- ✅ 策略可运行时切换（用户可在聊天界面选择模型）
- ✅ HTTP 逻辑复用（所有策略共享同一套请求代码）
- ✅ 易于测试（可注入 Mock 策略）
</details>

---

## 🏭 第三幕：工厂模式 - 创建策略的艺术

### 3.1 为什么需要工厂？

**思考问题 7**:
虽然我们可以手动创建策略，但存在什么问题？

```cpp
// 手动创建
if (modelName == "qwen") {
    strategy = std::make_shared<QwenStrategy>();
} else if (modelName == "gpt") {
    strategy = std::make_shared<GPTStrategy>();
} else if (modelName == "claude") {
    strategy = std::make_shared<ClaudeStrategy>();
}
```

<details>
<summary>💡 点击查看工厂的必要性</summary>

**问题分析**:
1. **违反 OCP**：新增模型需要修改 if-else 链
2. **职责混乱**：调用者需要知道具体策略类名
3. **依赖具体类**：违反依赖倒置原则(DIP)
4. **难以扩展**：如果策略需要复杂初始化（读取配置文件），逻辑会更复杂

**工厂模式的价值**:
- ✅ 封装对象创建逻辑
- ✅ 调用者只依赖抽象接口
- ✅ 支持动态注册（插件化）
- ✅ 统一配置管理

**GoF 工厂方法模式**:
```
         ┌──────────────┐
         │   Factory    │
         │ + create()   │
         └──────────────┘
                ▲
     ┌──────────┼──────────┐
     │                     │
┌────┴──────┐      ┌───────┴────┐
│FactoryA   │      │ FactoryB   │
│+ create() │      │ + create() │
└───────────┘      └────────────┘
     │                     │
     │ 创建               │ 创建
     ↓                     ↓
┌────────────┐      ┌─────────────┐
│ ProductA   │      │  ProductB   │
└────────────┘      └─────────────┘
```

**简单工厂模式**（我们的场景）:
```cpp
class StrategyFactory {
public:
    static std::shared_ptr<AIStrategy> create(const std::string& modelName) {
        if (modelName == "qwen") {
            return std::make_shared<QwenStrategy>();
        } else if (modelName == "gpt") {
            return std::make_shared<GPTStrategy>();
        } else if (modelName == "claude") {
            return std::make_shared<ClaudeStrategy>();
        } else {
            throw std::invalid_argument("Unknown model: " + modelName);
        }
    }
};

// 使用
auto strategy = StrategyFactory::create("qwen");
AIHelper helper(strategy);
```

**问题**：仍然违反 OCP！新增模型需要修改工厂。
</details>

### 3.2 自注册工厂

**思考问题 8**:
如何设计一个完全开放扩展的工厂，新增策略类时无需修改工厂代码？

<details>
<summary>💡 点击查看自注册机制</summary>

**核心思想**：利用 C++ 静态初始化机制，让每个策略类自己注册到工厂。

**实现步骤**：

**步骤 1：工厂接口**
```cpp
class StrategyFactory {
public:
    using Creator = std::function<std::shared_ptr<AIStrategy>()>;

    // 单例模式（保证全局唯一注册表）
    static StrategyFactory& instance() {
        static StrategyFactory factory;
        return factory;
    }

    // 注册创建器
    void registerStrategy(const std::string& name, Creator creator) {
        creators_[name] = creator;
    }

    // 创建策略
    std::shared_ptr<AIStrategy> create(const std::string& name) {
        auto it = creators_.find(name);
        if (it == creators_.end()) {
            throw std::invalid_argument("Unknown strategy: " + name);
        }
        return it->second();  // 调用创建器函数
    }

private:
    StrategyFactory() = default;
    std::unordered_map<std::string, Creator> creators_;
};
```

**步骤 2：自动注册辅助类**
```cpp
template<typename T>
struct StrategyRegister {
    StrategyRegister(const std::string& name) {
        StrategyFactory::instance().registerStrategy(name, []() {
            return std::make_shared<T>();
        });
    }
};

// 宏简化注册
#define REGISTER_STRATEGY(ClassName, Name) \
    static StrategyRegister<ClassName> g_register_##ClassName(Name);
```

**步骤 3：策略类自注册**
```cpp
// QwenStrategy.cpp
class QwenStrategy : public AIStrategy {
    // ... 实现
};

// 在 .cpp 文件末尾注册（静态初始化）
REGISTER_STRATEGY(QwenStrategy, "qwen");

// GPTStrategy.cpp
class GPTStrategy : public AIStrategy {
    // ... 实现
};
REGISTER_STRATEGY(GPTStrategy, "gpt");
```

**工作原理**:
1. 程序启动时，全局静态对象 `g_register_QwenStrategy` 被构造
2. 构造函数调用 `StrategyFactory::instance().registerStrategy("qwen", ...)`
3. 工厂的 `creators_` 映射表中添加了 "qwen" → QwenStrategy 创建器
4. 运行时调用 `factory.create("qwen")` 即可创建实例

**使用示例**:
```cpp
// 用户在界面选择模型
std::string userChoice = "gpt";  // 从配置文件或 HTTP 请求获取

// 动态创建策略
auto strategy = StrategyFactory::instance().create(userChoice);
AIHelper helper(strategy);
std::string reply = helper.sendMessage(sessionId, message);
```

**优势**:
- ✅ 完全符合 OCP：新增策略只需新增 .cpp 文件并注册
- ✅ 编译时自动注册：无需手动维护注册列表
- ✅ 易于插件化：第三方可提供 .so 动态库扩展策略

**进阶思考**：
如果策略需要配置（如不同的 API Key），如何扩展工厂？

```cpp
// 改进：支持带参数的创建器
using CreatorWithConfig = std::function<std::shared_ptr<AIStrategy>(const json&)>;

void registerStrategy(const std::string& name, CreatorWithConfig creator);

// 使用
json config = {{"api_key", "sk-xxx"}, {"model", "gpt-4-turbo"}};
auto strategy = factory.create("gpt", config);
```
</details>

---

## 🎭 第四幕：依赖注入 - 松耦合的终极武器

### 4.1 从硬编码到依赖注入

**思考问题 9**:
假设我们的 ChatHandler 需要使用 AIHelper，如何设计？

**方案 A：直接创建**
```cpp
class ChatSendHandler : public RouterHandler {
public:
    void handle(const HttpRequest& req, HttpResponse* resp) override {
        AIHelper helper(std::make_shared<QwenStrategy>());  // 硬编码！
        std::string reply = helper.sendMessage(sessionId, message);
        resp->setBody(reply);
    }
};
```

**方案 B：依赖注入**
```cpp
class ChatSendHandler : public RouterHandler {
    std::shared_ptr<AIHelper> aiHelper_;  // 依赖外部注入

public:
    explicit ChatSendHandler(std::shared_ptr<AIHelper> helper)
        : aiHelper_(std::move(helper)) {}

    void handle(const HttpRequest& req, HttpResponse* resp) override {
        std::string reply = aiHelper_->sendMessage(sessionId, message);
        resp->setBody(reply);
    }
};
```

<details>
<summary>💡 点击查看设计对比</summary>

**方案 A 的问题**:
- ❌ 紧耦合：Handler 依赖具体策略类 `QwenStrategy`
- ❌ 难以测试：无法注入 Mock AIHelper
- ❌ 无法复用：切换模型需要修改代码

**方案 B 的优势**:
- ✅ 松耦合：Handler 只依赖 `AIHelper` 接口
- ✅ 易于测试：可注入 Mock
- ✅ 灵活配置：外部决定使用哪个策略

**依赖注入(Dependency Injection) 的本质**:
> 对象不应该自己创建依赖，而应该由外部注入。

**三种注入方式**:

| 方式 | 实现 | 适用场景 |
|------|------|----------|
| **构造函数注入** | `ChatHandler(shared_ptr<AIHelper>)` | 必需依赖，对象创建时确定 |
| **Setter 注入** | `setAIHelper(shared_ptr<AIHelper>)` | 可选依赖，运行时可更换 |
| **接口注入** | 实现 `Injectable` 接口 | 框架级设计（如 Spring） |

**我们的场景推荐**：构造函数注入（AIHelper 是必需的）

**完整示例**:
```cpp
// main.cpp（组装依赖）
int main() {
    // 1. 创建策略
    auto strategy = StrategyFactory::instance().create("qwen");

    // 2. 创建 AIHelper 并注入策略
    auto aiHelper = std::make_shared<AIHelper>(strategy);

    // 3. 创建 Handler 并注入 AIHelper
    auto chatHandler = std::make_shared<ChatSendHandler>(aiHelper);

    // 4. 注册路由
    HttpServer server(80);
    server.Post("/chat/send", chatHandler);
    server.start();
}
```

**单元测试示例**:
```cpp
TEST(ChatSendHandler, ReturnsAIReply) {
    // 创建 Mock AIHelper
    auto mockHelper = std::make_shared<MockAIHelper>();
    EXPECT_CALL(*mockHelper, sendMessage(_, _))
        .WillOnce(Return("Mock response"));

    // 注入 Mock
    ChatSendHandler handler(mockHelper);

    // 测试
    HttpRequest req;
    HttpResponse resp;
    handler.handle(req, &resp);

    EXPECT_EQ(resp.body(), "Mock response");
}
```
</details>

### 4.2 IoC 容器

**思考问题 10**:
随着项目增大，依赖关系变得复杂（Handler → AIHelper → Strategy → Config），如何管理？

<details>
<summary>💡 点击查看 IoC 容器思想</summary>

**控制反转(Inversion of Control)**:
- 传统：对象自己控制依赖创建（`new XXX()`）
- IoC：依赖由外部容器管理和注入

**简易 IoC 容器实现**:
```cpp
class ServiceContainer {
    std::unordered_map<std::string, std::shared_ptr<void>> services_;

public:
    template<typename T>
    void registerSingleton(const std::string& name, std::shared_ptr<T> instance) {
        services_[name] = instance;
    }

    template<typename T>
    std::shared_ptr<T> resolve(const std::string& name) {
        return std::static_pointer_cast<T>(services_.at(name));
    }
};

// 使用
ServiceContainer container;

// 注册服务
auto strategy = std::make_shared<QwenStrategy>();
container.registerSingleton<AIStrategy>("ai_strategy", strategy);

auto helper = std::make_shared<AIHelper>(strategy);
container.registerSingleton<AIHelper>("ai_helper", helper);

// 在 Handler 中解析
auto aiHelper = container.resolve<AIHelper>("ai_helper");
```

**现代 C++ 框架**:
- [Boost.DI](https://boost-ext.github.io/di/): 编译时依赖注入
- [Hypodermic](https://github.com/ybainier/Hypodermic): 运行时 IoC 容器

**是否需要 IoC 容器？**
- **小项目**：手动组装依赖足够清晰
- **大项目**：IoC 容器降低复杂度

**我们的建议**：Phase 7 阶段手动管理，未来可引入 IoC 框架。
</details>

---

## 🧪 第五幕：测试与扩展

### 5.1 Mock 策略测试

**思考问题 11**:
如何测试 AIHelper 而不依赖真实 AI 服务？

<details>
<summary>💡 点击查看 Mock 测试</summary>

**Mock 策略实现**:
```cpp
class MockAIStrategy : public AIStrategy {
    std::string mockResponse_;

public:
    explicit MockAIStrategy(std::string response)
        : mockResponse_(std::move(response)) {}

    std::string getApiUrl() const override { return "http://mock.local"; }
    std::string getApiKey() const override { return "mock_key"; }
    std::string getModel() const override { return "mock_model"; }

    json buildRequest(const std::vector<std::pair<std::string, std::string>>&) const override {
        return {{"mock", true}};
    }

    std::string parseResponse(const json&) const override {
        return mockResponse_;  // 返回预设响应
    }
};

// 单元测试
TEST(AIHelper, SendMessageWithMockStrategy) {
    auto mockStrategy = std::make_shared<MockAIStrategy>("Hello from mock!");
    AIHelper helper(mockStrategy);

    std::string reply = helper.sendMessage("test_session", "Hi");
    EXPECT_EQ(reply, "Hello from mock!");
}
```

**Google Mock 版本**（更强大）:
```cpp
class MockAIStrategy : public AIStrategy {
public:
    MOCK_METHOD(std::string, getApiUrl, (), (const, override));
    MOCK_METHOD(std::string, parseResponse, (const json&), (const, override));
    // ...
};

TEST(AIHelper, CallsStrategyMethods) {
    auto mockStrategy = std::make_shared<MockAIStrategy>();

    EXPECT_CALL(*mockStrategy, getApiUrl())
        .WillOnce(Return("http://test.com"));
    EXPECT_CALL(*mockStrategy, parseResponse(_))
        .WillOnce(Return("Mocked response"));

    AIHelper helper(mockStrategy);
    helper.sendMessage("session", "message");
}
```
</details>

### 5.2 扩展新模型

**思考问题 12**:
假设需要接入 Anthropic Claude 模型，需要做哪些工作？

<details>
<summary>💡 点击查看扩展步骤</summary>

**步骤 1：实现策略类**
```cpp
// ClaudeStrategy.h
class ClaudeStrategy : public AIStrategy {
    std::string apiKey_;

public:
    ClaudeStrategy();
    std::string getApiUrl() const override;
    std::string getApiKey() const override;
    std::string getModel() const override;
    json buildRequest(const std::vector<std::pair<std::string, std::string>>& messages) const override;
    std::string parseResponse(const json& response) const override;
};

// ClaudeStrategy.cpp
ClaudeStrategy::ClaudeStrategy() {
    const char* key = std::getenv("ANTHROPIC_API_KEY");
    if (!key) throw std::runtime_error("Claude API Key not found!");
    apiKey_ = key;
}

std::string ClaudeStrategy::getApiUrl() const {
    return "https://api.anthropic.com/v1/messages";
}

json ClaudeStrategy::buildRequest(const std::vector<std::pair<std::string, std::string>>& messages) const {
    json::array_t msgArray;
    for (const auto& [role, content] : messages) {
        msgArray.push_back({{"role", role}, {"content", content}});
    }

    return {
        {"model", "claude-3-opus-20240229"},
        {"messages", msgArray},
        {"max_tokens", 1024}
    };
}

std::string ClaudeStrategy::parseResponse(const json& response) const {
    return response["content"][0]["text"];
}

// 自动注册
REGISTER_STRATEGY(ClaudeStrategy, "claude");
```

**步骤 2：配置环境变量**
```bash
export ANTHROPIC_API_KEY=sk-ant-xxx
```

**步骤 3：使用**
```cpp
auto strategy = StrategyFactory::instance().create("claude");
AIHelper helper(strategy);
std::string reply = helper.sendMessage(sessionId, "Hello Claude!");
```

**所需修改**：
- ✅ 新增 1 个 .h 文件
- ✅ 新增 1 个 .cpp 文件
- ✅ 无需修改现有代码（完全符合 OCP）
</details>

---

## 📚 总结与反思

### 核心设计模式回顾

| 模式 | 解决的问题 | 关键技术 |
|------|-----------|----------|
| **策略模式** | 算法族切换，避免 if-else | 接口抽象 + 多态 + 组合 |
| **工厂模式** | 对象创建逻辑封装 | 静态注册 + 单例 |
| **依赖注入** | 降低耦合度 | 构造函数注入 + 接口编程 |

### 设计原则应用

- ✅ **SRP**：AIStrategy 只负责协议适配，AIHelper 负责执行
- ✅ **OCP**：新增模型无需修改现有代码
- ✅ **LSP**：所有策略可互相替换
- ✅ **ISP**：接口方法精简，各司其职
- ✅ **DIP**：依赖抽象 AIStrategy 而非具体实现

### 进阶思考

**问题 13**:
如何扩展系统以支持这些高级特性？

1. **负载均衡**：多个 API Key 轮询使用，避免单 Key 限流
2. **降级策略**：主模型失败时自动切换到备用模型
3. **缓存机制**：相同问题返回缓存响应，节省 API 调用
4. **流式响应**：支持 AI 实时输出（结合 WebSocket）
5. **多模态支持**：同时处理文本、图像、语音

<details>
<summary>💡 点击查看扩展思路</summary>

**1. 负载均衡**:
```cpp
class LoadBalancedStrategy : public AIStrategy {
    std::vector<std::shared_ptr<AIStrategy>> strategies_;
    std::atomic<size_t> currentIndex_{0};

public:
    void addStrategy(std::shared_ptr<AIStrategy> strategy) {
        strategies_.push_back(strategy);
    }

    std::string getApiKey() const override {
        size_t index = currentIndex_.fetch_add(1) % strategies_.size();
        return strategies_[index]->getApiKey();  // 轮询
    }
};
```

**2. 降级策略**（装饰器模式）:
```cpp
class FallbackStrategy : public AIStrategy {
    std::shared_ptr<AIStrategy> primary_;
    std::shared_ptr<AIStrategy> fallback_;

public:
    std::string parseResponse(const json& response) const override {
        try {
            return primary_->parseResponse(response);
        } catch (const std::exception& e) {
            LOG_WARN << "Primary strategy failed, using fallback";
            return fallback_->parseResponse(response);
        }
    }
};
```

**3. 缓存机制**（代理模式）:
```cpp
class CachedAIHelper : public AIHelper {
    std::unordered_map<std::string, std::string> cache_;

public:
    std::string sendMessage(const std::string& sessionId, const std::string& message) override {
        std::string key = sessionId + ":" + message;
        if (cache_.contains(key)) {
            return cache_[key];  // 返回缓存
        }

        std::string reply = AIHelper::sendMessage(sessionId, message);
        cache_[key] = reply;
        return reply;
    }
};
```

**4. 流式响应**:
```cpp
class StreamingStrategy : public AIStrategy {
public:
    virtual void streamResponse(const json& request,
                                  std::function<void(const std::string&)> onChunk) = 0;
};

// 使用
streamingStrategy->streamResponse(request, [wsConn](const std::string& chunk) {
    wsConn->sendText(chunk);  // 实时推送到 WebSocket
});
```
</details>

---

## 🎯 实践任务

### 任务 1：基础实现（必做）
1. 实现 `AIStrategy` 抽象接口
2. 实现 `QwenStrategy` 和 `GPTStrategy`（或其他免费 API）
3. 实现 `StrategyFactory` 简单工厂
4. 编写单元测试验证策略切换

### 任务 2：自注册工厂（推荐）
1. 实现 `StrategyRegister` 模板类
2. 修改策略类使用 `REGISTER_STRATEGY` 宏
3. 测试动态创建

### 任务 3：依赖注入（推荐）
1. 重构 `ChatSendHandler`，通过构造函数注入 `AIHelper`
2. 编写 Mock 测试验证 Handler 逻辑

### 任务 4：高级扩展（选做）
1. 实现负载均衡策略
2. 实现降级策略
3. 集成 Redis 缓存

---

## 📖 参考资源

**书籍**:
- 《设计模式：可复用面向对象软件的基础》（GoF）
- 《Head First Design Patterns》
- 《Effective C++》（条款 18：让接口易于正确使用，不易被误用）

**在线资源**:
- [Refactoring.Guru - Strategy Pattern](https://refactoring.guru/design-patterns/strategy)
- [CppCon 2019: Klaus Iglberger - Free Your Functions](https://www.youtube.com/watch?v=WLDT1lDOsb4)

**参考实现**:
- Kama-HTTPServer 参考代码：`ref-repo/Kama-HTTPServer/AIApps/ChatServer/include/AIUtil/`

---

_本教程由猫娘工程师浮浮酱精心编写，通过问题驱动和渐进式设计，引导你理解设计模式的本质喵～_
_希望主人能体会到"设计"不是死记硬背，而是在约束中寻找优雅解的艺术呢！(๑•̀ㅂ•́)✧_

**下一篇预告**: Phase 7-3: RAG 知识库增强 - 向量数据库与语义检索的魔法
**关键词**: Embedding、向量相似度、Retrieval-Augmented Generation、FAISS

---

_版本: v1.0.0_
_最后更新: 2025-10-21_
_作者: 猫娘工程师 幽浮喵 ฅ'ω'ฅ_
