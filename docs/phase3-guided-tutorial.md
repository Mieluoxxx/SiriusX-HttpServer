# 阶段三引导教程：AI 应用层的设计智慧

> **教学理念**：苏格拉底式启发教学 + 实际代码深度剖析
> **适合读者**：完成阶段一、阶段二学习,掌握 HttpServer 框架核心的开发者
> **学习目标**：理解 AI 应用层的策略模式、工厂模式、消息历史管理、多模态处理、异步任务队列设计
> **参考代码**：`ref-repo/Kama-HTTPServer/AIApps/ChatServer/`

---

## 📖 开篇:从一次 AI 对话说起

当你在浏览器中输入"介绍一下自己"并点击发送按钮,背后发生了什么?

```
浏览器
  → POST /chat/send {"question": "介绍一下自己", "sessionId": "abc123"}
  → ChatSendHandler::handle()
  → 从 Session 验证用户登录状态 (userId: 1001, username: "Alice")
  → 创建/获取 AIHelper 实例 (基于 sessionId)
  → AIHelper::chat()
    → 策略工厂选择 AI 模型 (AliyunStrategy / DouBaoStrategy)
    → 构建请求 JSON (包含历史消息上下文)
    → CURL 调用阿里云通义千问 API
    → 解析 AI 响应
    → 存储消息到 MySQL (chat_messages 表)
  → 返回 JSON 响应给前端
```

这个流程揭示了阶段三的核心任务:**如何优雅地集成 AI 服务?** (..•˘_˘•..)

---

## 🎯 第一章:策略模式与工厂模式的联合使用

### 1.1 为什么需要策略模式?

**思考实验**:假设你要支持 3 种 AI 模型(阿里云通义千问、字节豆包、阿里云 RAG),如何设计代码?

**❌ 错误方案 1:硬编码判断**
```cpp
std::string AIHelper::chat(const std::string& message) {
    if (model == "qwen") {
        // 调用阿里云 API
        return callAliyunAPI(message);
    } else if (model == "doubao") {
        // 调用字节豆包 API
        return callDouBaoAPI(message);
    } else if (model == "rag") {
        // 调用 RAG API
        return callRAGAPI(message);
    }
}
```

**问题**:
- 违反开闭原则 (Open-Closed Principle):每次新增模型都要修改 `AIHelper` 类
- 职责不清:AI 请求构建、响应解析逻辑混在一起
- 测试困难:无法单独测试某个模型的逻辑

**✅ 正确方案:策略模式 (Strategy Pattern)**

**核心思想**:将不同 AI 模型的调用逻辑封装成独立的策略类,动态切换

<details>
<summary>💡 点击查看策略模式设计</summary>

**参考代码**:`ref-repo/Kama-HTTPServer/AIApps/ChatServer/include/AIUtil/AIStrategy.h`

```cpp
// 抽象策略接口
class AIStrategy {
public:
    virtual ~AIStrategy() = default;

    virtual std::string getApiUrl() const = 0;      // API 地址
    virtual std::string getApiKey() const = 0;      // API 密钥
    virtual std::string getModel() const = 0;       // 模型名称

    // 构建请求 JSON (不同模型格式可能不同)
    virtual json buildRequest(
        const std::vector<std::pair<std::string, long long>>& messages
    ) const = 0;

    // 解析响应 JSON (不同模型返回格式不同)
    virtual std::string parseResponse(const json& response) const = 0;

    bool isMCPModel = false;  // 是否支持工具调用 (MCP)
};

// 具体策略:阿里云通义千问
class AliyunStrategy : public AIStrategy {
public:
    AliyunStrategy() {
        const char* key = std::getenv("DASHSCOPE_API_KEY");
        if (!key) throw std::runtime_error("Aliyun API Key not found!");
        apiKey_ = key;
        isMCPModel = false;
    }

    std::string getApiUrl() const override {
        return "https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions";
    }
    std::string getApiKey() const override { return apiKey_; }
    std::string getModel() const override { return "qwen-plus"; }

    json buildRequest(const std::vector<std::pair<std::string, long long>>& messages) const override;
    std::string parseResponse(const json& response) const override;

private:
    std::string apiKey_;
};

// 具体策略:字节豆包
class DouBaoStrategy : public AIStrategy {
    // 类似实现,但 API 地址、请求格式、响应解析不同
};
```

</details>

**关键设计决策剖析**:

1. **为什么 `buildRequest` 参数是 `vector<pair<string, long long>>`?**
   - `pair.first`:消息内容 (用户问题或 AI 响应)
   - `pair.second`:时间戳 (用于消息排序和过期管理)
   - 为什么不用 `Message` 结构体? **KISS 原则**:避免过度封装,pair 足以表达语义

2. **为什么 API Key 从环境变量读取?**
   - 安全性:避免硬编码敏感信息到代码
   - 灵活性:不同环境 (开发/测试/生产) 可用不同密钥
   - 参考实现:`AIStrategy.h:40-42` (通过 `std::getenv` 读取)

3. **为什么需要 `isMCPModel` 标志?**
   - MCP (Model Context Protocol):支持工具调用 (Function Calling) 的模型
   - 普通模型:直接返回文本响应
   - MCP 模型:可能返回"需要调用工具 X"的指令,需要特殊处理
   - 参考实现:`AIHelper.cpp:44` (根据此标志选择不同的对话流程)

---

### 1.2 工厂模式:动态创建策略实例

**问题**:如何根据用户选择的模型名称 (如 `"1"` 代表阿里云、`"2"` 代表豆包) 创建对应的策略对象?

**✅ 方案:工厂模式 + 单例注册表**

**参考代码**:`ref-repo/Kama-HTTPServer/AIApps/ChatServer/include/AIUtil/AIFactory.h`

```cpp
class StrategyFactory {
public:
    using Creator = std::function<std::shared_ptr<AIStrategy>()>;

    // 单例模式获取工厂实例
    static StrategyFactory& instance();

    // 注册策略 (在程序启动时调用)
    void registerStrategy(const std::string& name, Creator creator);

    // 根据名称创建策略实例
    std::shared_ptr<AIStrategy> create(const std::string& name);

private:
    StrategyFactory() = default;
    std::unordered_map<std::string, Creator> creators;  // 策略注册表
};
```

**使用示例**:
```cpp
// 注册策略 (main.cpp 中调用)
StrategyFactory::instance().registerStrategy("1", [] {
    return std::make_shared<AliyunStrategy>();
});
StrategyFactory::instance().registerStrategy("2", [] {
    return std::make_shared<DouBaoStrategy>();
});

// 动态创建策略
auto strategy = StrategyFactory::instance().create("1");  // 创建阿里云策略
```

**思考题** (๑•̀ㅂ•́)✧:

<details>
<summary>Q1: 为什么使用 `std::function` 而非函数指针?</summary>

**答案**:
- `std::function` 可以存储 lambda 表达式、函数对象、成员函数等
- 函数指针只能存储普通函数
- 这里使用 lambda `[] { return std::make_shared<T>(); }` 实现延迟初始化

</details>

<details>
<summary>Q2: 为什么工厂是单例模式?</summary>

**答案**:
- 全局只需要一个策略注册表
- 避免重复注册和内存浪费
- 方便在程序任何地方访问工厂创建策略

</details>

<details>
<summary>Q3: 注册表能否用 `std::map` 代替 `std::unordered_map`?</summary>

**答案**:
- 可以,但 `unordered_map` 查找速度 O(1) 更快
- 策略名称没有排序需求,不需要 `map` 的有序性
- 参考:"精准路由匹配"同样使用哈希表 (阶段二教程)

</details>

---

### 1.3 AIHelper:策略的消费者

**职责**:封装 CURL 调用 AI API,管理消息历史,协调策略执行

**参考代码**:`ref-repo/Kama-HTTPServer/AIApps/ChatServer/include/AIUtil/AIHelper.h`

```cpp
class AIHelper {
public:
    AIHelper();

    // 设置策略 (动态切换模型)
    void setStrategy(std::shared_ptr<AIStrategy> strat);

    // 添加消息到历史 (同时存入 MySQL)
    void addMessage(int userId, const std::string& userName,
                   bool is_user, const std::string& userInput,
                   std::string sessionId);

    // 恢复历史消息 (从数据库加载)
    void restoreMessage(const std::string& userInput, long long ms);

    // 发送消息给 AI
    std::string chat(int userId, std::string userName,
                    std::string sessionId, std::string userQuestion,
                    std::string modelType);

    // 获取当前会话消息历史
    std::vector<std::pair<std::string, long long>> GetMessages();

private:
    // 执行 CURL 请求
    json executeCurl(const json& payload);

    // 异步存储消息到 MySQL
    void pushMessageToMysql(int userId, const std::string& userName,
                           bool is_user, const std::string& userInput,
                           long long ms, std::string sessionId);

private:
    std::shared_ptr<AIStrategy> strategy;  // 当前使用的策略
    std::vector<std::pair<std::string, long long>> messages;  // 消息历史
};
```

**核心方法剖析**:`chat()` 方法

**参考代码**:`AIHelper.cpp:38-118`

```cpp
std::string AIHelper::chat(int userId, std::string userName,
                          std::string sessionId, std::string userQuestion,
                          std::string modelType) {

    // 1. 动态切换策略
    setStrategy(StrategyFactory::instance().create(modelType));

    // 2. 区分普通模型和 MCP 模型
    if (false == strategy->isMCPModel) {
        // 普通模型:直接调用
        addMessage(userId, userName, true, userQuestion, sessionId);
        json payload = strategy->buildRequest(this->messages);

        json response = executeCurl(payload);
        std::string answer = strategy->parseResponse(response);

        addMessage(userId, userName, false, answer, sessionId);
        return answer;
    }

    // MCP 模型:支持工具调用 (Function Calling)
    // 3. 第一次调用:询问 AI 是否需要工具
    AIConfig config;
    config.loadFromFile("../AIApps/ChatServer/resource/config.json");
    std::string tempUserQuestion = config.buildPrompt(userQuestion);

    messages.push_back({tempUserQuestion, 0});
    json firstReq = strategy->buildRequest(this->messages);
    json firstResp = executeCurl(firstReq);
    std::string aiResult = strategy->parseResponse(firstResp);
    messages.pop_back();  // 移除临时提示词

    // 4. 解析 AI 响应:是否调用工具
    AIToolCall call = config.parseAIResponse(aiResult);

    if (!call.isToolCall) {
        // AI 不需要工具,直接返回
        addMessage(userId, userName, true, userQuestion, sessionId);
        addMessage(userId, userName, false, aiResult, sessionId);
        return aiResult;
    }

    // 5. AI 需要工具:执行工具并获取结果
    json toolResult;
    AIToolRegistry registry;
    try {
        toolResult = registry.invoke(call.toolName, call.args);
    } catch (const std::exception& e) {
        std::string err = "[工具调用失败] " + std::string(e.what());
        addMessage(userId, userName, true, userQuestion, sessionId);
        addMessage(userId, userName, false, err, sessionId);
        return err;
    }

    // 6. 第二次调用 AI:带上工具执行结果
    std::string secondPrompt = config.buildToolResultPrompt(
        userQuestion, call.toolName, call.args, toolResult
    );
    messages.push_back({secondPrompt, 0});

    json secondReq = strategy->buildRequest(messages);
    json secondResp = executeCurl(secondReq);
    std::string finalAnswer = strategy->parseResponse(secondResp);
    messages.pop_back();  // 移除临时提示词

    addMessage(userId, userName, true, userQuestion, sessionId);
    addMessage(userId, userName, false, finalAnswer, sessionId);
    return finalAnswer;
}
```

**设计亮点** (ฅ'ω'ฅ):

1. **策略动态切换**:每次调用都可以更换模型,支持用户在对话中切换 AI
2. **MCP 工具调用流程**:
   - 第一次询问 AI:"需要工具吗?"
   - 如果需要,执行工具获取结果
   - 第二次告诉 AI:"工具返回了这个结果"
   - 类似人类思考:"我需要查天气 → 查询天气 API → 根据天气数据回答用户"
3. **临时提示词管理**:使用 `push_back` + `pop_back` 避免污染消息历史
4. **异常处理**:工具调用失败时,将错误信息返回给用户而非崩溃

**性能考虑**:

<details>
<summary>Q: 为什么不缓存 AI 响应?</summary>

**答案**:
- AI 响应依赖上下文历史,同样的问题在不同会话中答案可能不同
- 如需缓存,可针对**无上下文的常见问题** (如"介绍一下自己") 使用 Redis 缓存
- 缓存 Key 设计:`hash(question + model + history_context)`

</details>

---

## 🧠 第二章:消息历史管理的设计哲学

### 2.1 为什么需要消息历史?

**场景**:用户在聊天中问"今天天气如何?",AI 回答"北京今天晴天",然后用户问"那明天呢?"

**问题**:AI 如何知道"明天"指的是"北京明天的天气"?

**答案**:通过**上下文历史**理解对话连贯性

**消息历史的作用**:
1. **上下文理解**:AI 需要历史消息推断代词指代、延续话题
2. **个性化对话**:记住用户偏好 (如"我喜欢蓝色" → 后续推荐蓝色商品)
3. **多轮对话**:支持复杂任务分解 (如"帮我订机票" → "从哪到哪?" → "北京到上海")

---

### 2.2 消息历史的存储策略

**方案对比**:

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **内存存储** (vector) | 读写速度快 | 服务器重启丢失 | 临时会话、测试环境 |
| **MySQL 存储** | 持久化、可查询 | 读写较慢、需要数据库连接 | 生产环境、需要历史回溯 |
| **Redis 存储** | 速度快、持久化 | 需要额外中间件、内存占用大 | 高并发场景、分布式部署 |
| **混合存储** | 兼顾性能和持久化 | 实现复杂 | 大规模生产环境 |

**参考实现采用的方案**:内存 + MySQL 混合

```cpp
class AIHelper {
private:
    // 内存缓存 (快速读取)
    std::vector<std::pair<std::string, long long>> messages;

    // MySQL 持久化 (参考 AIHelper.cpp:29 pushMessageToMysql 方法)
    void pushMessageToMysql(int userId, const std::string& userName,
                           bool is_user, const std::string& userInput,
                           long long ms, std::string sessionId) {
        // 使用消息队列异步写入 MySQL (避免阻塞)
        // 实现细节见 AIHelper.cpp:171-220
    }
};
```

**关键设计**:异步写入 MySQL

**参考代码**:`AIHelper.cpp:171-220`

```cpp
void AIHelper::pushMessageToMysql(int userId, const std::string& userName,
                                  bool is_user, const std::string& userInput,
                                  long long ms, std::string sessionId) {

    // 构建 INSERT 语句
    std::ostringstream sql;
    sql << "INSERT INTO chat_messages (session_id, user_id, username, role, content, timestamp) VALUES ('"
        << escapeString(sessionId) << "', "
        << userId << ", '"
        << escapeString(userName) << "', '"
        << (is_user ? "user" : "assistant") << "', '"
        << escapeString(userInput) << "', "
        << ms << ")";

    // 发布到消息队列 (RabbitMQ)
    try {
        MQManager mqManager;
        mqManager.publishMessage("mysql_queue", sql.str());
    } catch (const std::exception& e) {
        LOG_ERROR << "Failed to publish message to MQ: " << e.what();
    }
}
```

**为什么使用消息队列?** (..•˘_˘•..)

1. **解耦**:AI 响应不必等待 MySQL 写入完成 (减少延迟 100ms+)
2. **削峰填谷**:高并发时消息队列缓冲,数据库按自己的节奏消费
3. **容错**:MySQL 临时故障时消息暂存队列,恢复后继续写入

**思考实验**:

<details>
<summary>如果不用消息队列,直接同步写 MySQL 会怎样?</summary>

**场景**:1000 个用户同时发送消息

**结果**:
- 每次 AI 响应都要等待 MySQL 写入 (50-100ms)
- 用户感觉卡顿 (原本 2s 响应变成 2.1s)
- 数据库连接池耗尽 (1000 个并发写入超过连接池大小 10)
- 部分请求失败返回 500 错误

**使用消息队列后**:
- AI 响应立即返回 (延迟 2s)
- 消息队列异步写入 MySQL (用户无感知)
- 数据库按最大吞吐能力消费 (如 100 条/秒)

</details>

---

### 2.3 消息历史的过期管理

**问题**:如果用户和 AI 聊了 1000 轮,每次都加载完整历史会怎样?

**后果**:
- AI API 请求体过大 (超过 100KB)
- AI 推理时间增加 (处理长文本更慢)
- 成本增加 (AI API 按 Token 数计费)

**解决方案**:历史消息截断策略

| 策略 | 实现 | 优点 | 缺点 |
|------|------|------|------|
| **固定数量** | 只保留最近 20 条 | 简单 | 可能丢失关键上下文 |
| **滑动窗口** | 保留最近 10 分钟 | 时间相关性强 | 实现复杂 |
| **Token 限制** | 计算 Token 数不超过 4096 | 精确控制成本 | 需要 Tokenizer |
| **重要性打分** | AI 评估消息重要性 | 上下文最优 | 计算开销大 |

**推荐实现**:固定数量 + 总结压缩

```cpp
// 伪代码
if (messages.size() > 20) {
    // 保留前 5 条 (重要上下文)
    std::vector<Message> summary = messages[0:5];

    // 压缩中间 10 条为一条总结
    std::string compressed = ai->summarize(messages[5:15]);
    summary.push_back({"system", compressed});

    // 保留最近 5 条
    summary.insert(summary.end(), messages[15:20].begin(), messages[15:20].end());

    messages = summary;
}
```

---

## 🔧 第三章:多模态处理 - 图像识别

### 3.1 图像识别的完整流程

**场景**:用户上传一张猫咪图片,AI 识别后回答"这是一只橘猫"

**流程**:
```
浏览器上传图片 (multipart/form-data)
  → AIUploadHandler::handle()
    → 保存图片到本地 (resource/uploads/user_1001_20251020_abc.jpg)
    → 记录到 image_uploads 表
    → 返回 imageId
  → 前端调用 POST /upload/send {"imageId": 123}
  → AIUploadSendHandler::handle()
    → 从数据库查询图片路径
    → ImageRecognizer::recognize(imagePath)
      → OpenCV 加载图片
      → 预处理 (resize 224x224, normalize)
      → ONNX Runtime 推理 (前向传播)
      → 后处理 (解析输出、排序)
    → 返回识别结果 JSON
```

---

### 3.2 ImageRecognizer 的设计

**参考代码**:`ref-repo/Kama-HTTPServer/AIApps/ChatServer/include/AIUtil/ImageRecognizer.h`

```cpp
class ImageRecognizer {
public:
    // 构造函数:加载 ONNX 模型
    ImageRecognizer(const std::string& modelPath);

    // 识别图像,返回分类结果
    std::vector<ClassificationResult> recognize(const std::string& imagePath);

private:
    // 图像预处理
    cv::Mat preprocess(const cv::Mat& image);

    // 后处理:解析模型输出
    std::vector<ClassificationResult> postprocess(
        const std::vector<float>& output
    );

private:
    Ort::Session session_;     // ONNX Runtime 会话
    Ort::Env env_;            // ONNX 环境
};
```

**核心步骤剖析**:

**步骤 1:图像预处理**

**参考代码**:`ImageRecognizer.cpp:50-80`

```cpp
cv::Mat ImageRecognizer::preprocess(const cv::Mat& image) {
    cv::Mat processed;

    // 1. Resize 到模型输入尺寸 (如 224x224)
    cv::resize(image, processed, cv::Size(224, 224));

    // 2. 归一化到 [0, 1]
    processed.convertTo(processed, CV_32F, 1.0 / 255);

    // 3. 标准化 (减均值、除标准差)
    // ImageNet 数据集的均值和标准差
    std::vector<float> mean = {0.485, 0.456, 0.406};
    std::vector<float> std = {0.229, 0.224, 0.225};

    for (int c = 0; c < 3; ++c) {
        for (int h = 0; h < 224; ++h) {
            for (int w = 0; w < 224; ++w) {
                processed.at<cv::Vec3f>(h, w)[c] =
                    (processed.at<cv::Vec3f>(h, w)[c] - mean[c]) / std[c];
            }
        }
    }

    return processed;
}
```

**为什么要归一化和标准化?**

<details>
<summary>💡 点击查看答案</summary>

**归一化 (Normalization)**:
- 将像素值从 [0, 255] 缩放到 [0, 1]
- 目的:加速神经网络收敛,避免梯度爆炸

**标准化 (Standardization)**:
- 减去均值、除以标准差
- 目的:数据分布变为均值 0、方差 1,提高训练稳定性
- ImageNet 均值 `{0.485, 0.456, 0.406}` 是从 120 万张图片统计得出

</details>

**步骤 2:ONNX Runtime 推理**

```cpp
std::vector<ClassificationResult> ImageRecognizer::recognize(
    const std::string& imagePath
) {
    // 1. 加载图像
    cv::Mat image = cv::imread(imagePath);
    if (image.empty()) {
        throw std::runtime_error("Failed to load image");
    }

    // 2. 预处理
    cv::Mat input = preprocess(image);

    // 3. 转换为 ONNX 输入格式 (NCHW: Batch, Channel, Height, Width)
    std::vector<float> inputTensor;
    inputTensor.resize(1 * 3 * 224 * 224);

    for (int c = 0; c < 3; ++c) {
        for (int h = 0; h < 224; ++h) {
            for (int w = 0; w < 224; ++w) {
                inputTensor[c * 224 * 224 + h * 224 + w] =
                    input.at<cv::Vec3f>(h, w)[c];
            }
        }
    }

    // 4. 执行推理
    auto output = session_.Run(
        Ort::RunOptions{nullptr},
        inputNames.data(), &inputTensor, 1,
        outputNames.data(), 1
    );

    // 5. 后处理
    float* outputData = output[0].GetTensorMutableData<float>();
    std::vector<float> scores(outputData, outputData + 1000);  // ImageNet 1000 类

    return postprocess(scores);
}
```

**步骤 3:后处理 (解析输出)**

```cpp
std::vector<ClassificationResult> ImageRecognizer::postprocess(
    const std::vector<float>& scores
) {
    std::vector<ClassificationResult> results;

    // 1. 找到 Top-5 分类结果
    std::vector<int> indices(scores.size());
    std::iota(indices.begin(), indices.end(), 0);

    std::partial_sort(indices.begin(), indices.begin() + 5, indices.end(),
                     [&scores](int i, int j) { return scores[i] > scores[j]; });

    // 2. 构建结果
    for (int i = 0; i < 5; ++i) {
        ClassificationResult result;
        result.classId = indices[i];
        result.label = getClassName(indices[i]);  // 从 ImageNet 类别表查询
        result.confidence = scores[indices[i]];
        results.push_back(result);
    }

    return results;
}
```

**性能优化建议**:

1. **批量推理**:一次处理多张图片,充分利用 GPU
2. **模型量化**:使用 INT8 模型替代 FP32 (速度提升 4 倍)
3. **硬件加速**:使用 CUDA (GPU) 或 TensorRT (NVIDIA) 加速

---

## 🎤 第四章:语音处理 (可选扩展)

### 4.1 语音转文本 (ASR: Automatic Speech Recognition)

**流程**:
```
用户录音 (浏览器 MediaRecorder API)
  → 上传音频文件 (WAV/MP3)
  → ChatSpeechHandler::handle()
  → AISpeechProcessor::speechToText(audioPath)
    → 调用百度/阿里云语音识别 API
    → 返回识别文本
  → 将文本传给 AI 对话
```

**参考代码**:`AISpeechProcessor.h` (简化版)

```cpp
class AISpeechProcessor {
public:
    // 语音转文本
    std::string speechToText(const std::string& audioFilePath);

    // 文本转语音 (TTS: Text-To-Speech)
    std::string textToSpeech(const std::string& text);

private:
    std::string apiKey_;
    std::string apiUrl_;
};
```

**关键技术点**:
- 音频格式转换 (FFmpeg)
- Base64 编码传输音频数据
- 流式识别 (WebSocket 实时传输)

---

## 🔀 第五章:异步任务队列 - RabbitMQ

### 5.1 为什么需要消息队列?

**场景**:
1. AI 推理耗时 5 秒,用户等待超时
2. 图像识别需要调用 GPU,但 GPU 繁忙
3. 批量处理 100 张图片,不能阻塞当前请求

**解决方案**:异步任务队列

**流程**:
```
用户请求
  → Handler 创建任务
  → 发布到 RabbitMQ 队列
  → 立即返回 taskId 给用户
  → 用户轮询 /task/status?taskId=123

后台 Worker 进程
  → 从 RabbitMQ 消费任务
  → 执行 AI 推理
  → 将结果写入 Redis/MySQL
  → 更新任务状态为"已完成"
```

---

### 5.2 MQManager 的设计

**参考代码**:`ref-repo/Kama-HTTPServer/AIApps/ChatServer/include/AIUtil/MQManager.h`

```cpp
class MQManager {
public:
    MQManager();

    // 发布消息到队列
    void publishMessage(const std::string& queue, const std::string& message);

    // 消费消息 (回调处理)
    void consumeMessage(const std::string& queue,
                       std::function<void(const std::string&)> callback);

private:
    AmqpClient::Channel::ptr_t channel_;  // RabbitMQ 通道
};
```

**使用示例**:

```cpp
// 发布任务
MQManager mq;
json task;
task["type"] = "image_recognition";
task["imagePath"] = "/uploads/image123.jpg";
task["userId"] = 1001;
mq.publishMessage("ai_tasks", task.dump());

// 消费任务 (Worker 进程)
mq.consumeMessage("ai_tasks", [](const std::string& message) {
    auto task = json::parse(message);

    if (task["type"] == "image_recognition") {
        ImageRecognizer recognizer("model.onnx");
        auto results = recognizer.recognize(task["imagePath"]);

        // 将结果写入数据库
        saveResults(task["userId"], results);
    }
});
```

**消息队列的核心价值**:
1. **解耦**:请求处理和任务执行分离
2. **异步**:用户无需等待耗时操作
3. **削峰**:高并发时缓冲,按处理能力消费
4. **容错**:任务失败可重试 (Retry 机制)

---

## 🛠️ 第六章:业务路由处理器实现

### 6.1 ChatSendHandler:发送消息核心逻辑

**参考代码**:`ref-repo/Kama-HTTPServer/AIApps/ChatServer/src/handlers/ChatSendHandler.cpp`

**完整流程剖析**:

```cpp
void ChatSendHandler::handle(const http::HttpRequest& req,
                            http::HttpResponse* resp) {
    try {
        // 1. 验证用户登录状态
        auto session = server_->getSessionManager()->getSession(req, resp);
        if (session->getValue("isLoggedIn") != "true") {
            // 返回 401 未授权
            json errorResp;
            errorResp["status"] = "error";
            errorResp["message"] = "Unauthorized";

            resp->setStatusLine(req.getVersion(),
                               HttpResponse::k401Unauthorized,
                               "Unauthorized");
            resp->setContentType("application/json");
            resp->setBody(errorResp.dump(4));
            return;
        }

        // 2. 提取用户信息
        int userId = std::stoi(session->getValue("userId"));
        std::string username = session->getValue("username");

        // 3. 解析请求 JSON
        auto body = req.getBody();
        auto j = json::parse(body);
        std::string userQuestion = j["question"];
        std::string sessionId = j["sessionId"];
        std::string modelType = j.value("modelType", "1");  // 默认阿里云

        // 4. 获取或创建 AIHelper 实例 (线程安全)
        std::shared_ptr<AIHelper> aiHelper;
        {
            std::lock_guard<std::mutex> lock(server_->mutexForChatInformation);

            auto& userSessions = server_->chatInformation[userId];

            if (userSessions.find(sessionId) == userSessions.end()) {
                // 新会话:创建 AIHelper
                userSessions.emplace(sessionId, std::make_shared<AIHelper>());
            }
            aiHelper = userSessions[sessionId];
        }

        // 5. 调用 AI 获取响应
        std::string aiResponse = aiHelper->chat(
            userId, username, sessionId, userQuestion, modelType
        );

        // 6. 返回成功响应
        json successResp;
        successResp["success"] = true;
        successResp["Information"] = aiResponse;

        resp->setStatusLine(req.getVersion(), HttpResponse::k200Ok, "OK");
        resp->setContentType("application/json");
        resp->setBody(successResp.dump(4));

    } catch (const std::exception& e) {
        // 7. 异常处理
        json failureResp;
        failureResp["status"] = "error";
        failureResp["message"] = e.what();

        resp->setStatusLine(req.getVersion(),
                           HttpResponse::k400BadRequest,
                           "Bad Request");
        resp->setContentType("application/json");
        resp->setBody(failureResp.dump(4));
    }
}
```

**关键设计决策** (ฅ'ω'ฅ):

1. **为什么需要 `mutexForChatInformation` 互斥锁?**
   - `chatInformation` 是全局共享数据结构 `map<userId, map<sessionId, AIHelper>>`
   - 多个用户同时发送消息时,可能同时访问/修改这个 map
   - 互斥锁保证线程安全,避免数据竞争

2. **为什么 AIHelper 是 `shared_ptr`?**
   - 同一会话的多个请求共享同一个 AIHelper 实例
   - 共享历史消息 (messages 成员变量)
   - `shared_ptr` 自动管理生命周期,无需手动 delete

3. **为什么在锁外调用 `aiHelper->chat()`?**
   - AI API 调用耗时 (2-5 秒)
   - 如果在锁内调用,其他用户的请求会被阻塞
   - 锁的粒度应尽可能小 (只保护 map 访问)

**性能优化建议**:

```cpp
// 优化方案:使用读写锁 (shared_mutex)
std::shared_mutex mutexForChatInformation;

// 读取时使用共享锁 (多个线程可同时读)
{
    std::shared_lock<std::shared_mutex> lock(server_->mutexForChatInformation);
    aiHelper = server_->chatInformation[userId][sessionId];
}

// 写入时使用独占锁
{
    std::unique_lock<std::shared_mutex> lock(server_->mutexForChatInformation);
    server_->chatInformation[userId][sessionId] = newHelper;
}
```

---

### 6.2 ChatHistoryHandler:加载聊天历史

**功能**:从 MySQL 加载指定会话的历史消息

**实现思路**:

```cpp
void ChatHistoryHandler::handle(const HttpRequest& req, HttpResponse* resp) {
    // 1. 获取 sessionId (从 Query String 或 POST body)
    std::string sessionId = req.getQueryParameter("sessionId");

    // 2. 查询 MySQL
    auto conn = DbConnectionPool::getInstance().getConnection();
    auto stmt = conn->prepareStatement(
        "SELECT role, content, timestamp FROM chat_messages "
        "WHERE session_id = ? ORDER BY timestamp ASC"
    );
    stmt->setString(1, sessionId);
    auto res = stmt->executeQuery();

    // 3. 构建 JSON 响应
    json messages = json::array();
    while (res->next()) {
        json msg;
        msg["role"] = res->getString("role");
        msg["content"] = res->getString("content");
        msg["timestamp"] = res->getInt64("timestamp");
        messages.push_back(msg);
    }

    // 4. 返回响应
    json response;
    response["success"] = true;
    response["messages"] = messages;

    resp->setStatusLine(req.getVersion(), HttpResponse::k200Ok, "OK");
    resp->setContentType("application/json");
    resp->setBody(response.dump(4));
}
```

---

## 🧩 第七章:综合实践 - 构建完整聊天应用

### 7.1 数据库表设计回顾

**参考 TODO.md:126-171** (数据库表创建 SQL)

**核心表**:

1. **users 表**:用户账号信息
   ```sql
   CREATE TABLE users (
       user_id INT PRIMARY KEY AUTO_INCREMENT,
       username VARCHAR(50) UNIQUE NOT NULL,
       password_hash VARCHAR(255) NOT NULL,
       created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
       INDEX idx_username (username)
   );
   ```

2. **ai_sessions 表**:AI 聊天会话
   ```sql
   CREATE TABLE ai_sessions (
       session_id VARCHAR(64) PRIMARY KEY,
       user_id INT NOT NULL,
       ai_model VARCHAR(50) DEFAULT 'qwen',
       created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
       last_active TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
       FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
   );
   ```

3. **chat_messages 表**:聊天消息历史
   ```sql
   CREATE TABLE chat_messages (
       message_id INT PRIMARY KEY AUTO_INCREMENT,
       session_id VARCHAR(64) NOT NULL,
       role ENUM('user', 'assistant') NOT NULL,
       content TEXT NOT NULL,
       timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
       FOREIGN KEY (session_id) REFERENCES ai_sessions(session_id) ON DELETE CASCADE,
       INDEX idx_session_id (session_id)
   );
   ```

**索引设计思考**:

<details>
<summary>Q: 为什么 chat_messages 表在 session_id 上建索引?</summary>

**答案**:
- 查询历史消息时:`SELECT * FROM chat_messages WHERE session_id = 'abc123'`
- 没有索引:全表扫描 O(n)
- 有索引:B+ 树查找 O(log n)
- 性能提升:100 万条消息,全表扫描 500ms vs 索引查询 10ms

</details>

---

### 7.2 主程序入口骨架

**参考代码**:`AIApps/ChatServer/src/main.cpp` (简化版)

```cpp
#include "HttpServer/include/http/HttpServer.h"
#include "handlers/ChatLoginHandler.h"
#include "handlers/ChatSendHandler.h"
#include "handlers/ChatHistoryHandler.h"
#include "AIUtil/AIFactory.h"

int main() {
    // 1. 初始化日志
    muduo::Logger::setLogLevel(muduo::Logger::INFO);

    // 2. 初始化数据库连接池
    auto& dbPool = http::db::DbConnectionPool::getInstance();
    dbPool.init("localhost", "chatserver", "password", "chatserver_db", 10);

    // 3. 注册 AI 策略
    StrategyFactory::instance().registerStrategy("1", [] {
        return std::make_shared<AliyunStrategy>();
    });
    StrategyFactory::instance().registerStrategy("2", [] {
        return std::make_shared<DouBaoStrategy>();
    });

    // 4. 创建 HTTP 服务器
    http::HttpServer server(80, "ChatServer", false);  // HTTP 模式

    // 5. 注册中间件
    auto corsMiddleware = std::make_shared<http::middleware::CorsMiddleware>();
    server.addMiddleware(corsMiddleware);

    // 6. 注册路由
    server.Post("/login", std::make_shared<ChatLoginHandler>(&server));
    server.Post("/chat/send", std::make_shared<ChatSendHandler>(&server));
    server.Get("/chat/history", std::make_shared<ChatHistoryHandler>(&server));

    // 7. 启动服务器
    LOG_INFO << "ChatServer starting on port 80";
    server.start();

    return 0;
}
```

---

### 7.3 测试验证步骤

**步骤 1:测试用户注册**

```bash
curl -X POST http://localhost/register \
  -H "Content-Type: application/json" \
  -d '{"username": "Alice", "password": "password123"}'
```

**步骤 2:测试用户登录**

```bash
curl -X POST http://localhost/login \
  -H "Content-Type: application/json" \
  -d '{"username": "Alice", "password": "password123"}' \
  -c cookies.txt  # 保存 Cookie
```

**步骤 3:测试发送消息**

```bash
curl -X POST http://localhost/chat/send \
  -H "Content-Type: application/json" \
  -b cookies.txt \
  -d '{
    "question": "介绍一下自己",
    "sessionId": "session_001",
    "modelType": "1"
  }'
```

**步骤 4:测试加载历史**

```bash
curl -X GET "http://localhost/chat/history?sessionId=session_001" \
  -b cookies.txt
```

**步骤 5:测试图像上传**

```bash
curl -X POST http://localhost/upload \
  -F "image=@cat.jpg" \
  -b cookies.txt
```

---

## 💡 第八章:架构设计模式总结

### 8.1 本阶段使用的设计模式

| 设计模式 | 应用场景 | 代码位置 | 价值 |
|----------|----------|----------|------|
| **策略模式** | 支持多种 AI 模型切换 | `AIStrategy.h` | 扩展性:新增模型无需修改原代码 |
| **工厂模式** | 动态创建策略实例 | `AIFactory.h` | 解耦:客户端无需知道具体策略类名 |
| **单例模式** | 全局唯一的策略工厂 | `StrategyFactory` | 避免重复注册,全局访问 |
| **RAII** | 自动归还数据库连接 | `DbConnection` | 资源安全,防止泄漏 |
| **观察者模式** (隐式) | 消息队列的发布-订阅 | `MQManager` | 解耦生产者和消费者 |

---

### 8.2 SOLID 原则在阶段三的体现

**S (单一职责)**:
- `AIHelper`:负责 AI API 调用和消息历史管理
- `AIStrategy`:负责特定 AI 模型的请求构建和响应解析
- `ImageRecognizer`:负责图像识别
- 职责清晰,互不干扰

**O (开闭原则)**:
- 新增 AI 模型:只需实现 `AIStrategy` 子类,无需修改 `AIHelper`
- 新增路由处理器:实现 `RouterHandler` 接口,注册到 `Router`

**L (里氏替换)**:
- 所有 `AIStrategy` 子类可互相替换,`AIHelper` 无需感知差异

**I (接口隔离)**:
- `AIStrategy` 接口精简 (5 个方法),只定义必需操作
- 避免"胖接口"导致子类实现无用方法

**D (依赖倒置)**:
- `AIHelper` 依赖抽象接口 `AIStrategy`,而非具体类 `AliyunStrategy`
- 便于单元测试 (Mock 策略)

---

## 🎓 第九章:性能优化与生产部署建议

### 9.1 性能优化清单

1. **AI 响应缓存** (Redis):
   ```cpp
   // 伪代码
   std::string cacheKey = hash(question + model + last5Messages);
   if (redis.exists(cacheKey)) {
       return redis.get(cacheKey);
   }
   std::string response = ai->chat(question);
   redis.setex(cacheKey, 3600, response);  // 缓存 1 小时
   ```

2. **数据库查询优化**:
   - 使用 `LIMIT` 限制历史消息数量
   - 避免 `SELECT *`,只查询必需字段
   - 添加复合索引:`INDEX idx_session_time (session_id, timestamp)`

3. **图像识别批处理**:
   - 攒够 10 张图片一起推理,提高 GPU 利用率

4. **连接池大小调优**:
   ```cpp
   int poolSize = std::thread::hardware_concurrency() * 2;  // CPU 核心数 * 2
   dbPool.init("localhost", "user", "pass", "db", poolSize);
   ```

---

### 9.2 安全加固措施

1. **API Key 管理**:
   - 从环境变量读取 (不硬编码)
   - 生产环境使用 AWS Secrets Manager / HashiCorp Vault

2. **输入验证**:
   ```cpp
   if (userQuestion.length() > 5000) {
       throw std::invalid_argument("Question too long");
   }
   ```

3. **限流保护**:
   ```cpp
   // 伪代码:每用户每分钟最多 10 次请求
   int count = redis.incr("rate_limit:" + userId);
   redis.expire("rate_limit:" + userId, 60);
   if (count > 10) {
       throw TooManyRequestsException();
   }
   ```

4. **SQL 注入防护**:
   - 始终使用 `PreparedStatement`
   - 避免字符串拼接 SQL

---

## 🎯 第十章:综合验收标准

### 阶段三验收清单

- [ ] AI API 调用返回正确响应
- [ ] 消息历史存储到 MySQL 成功
- [ ] 从数据库加载历史消息正常
- [ ] 图像识别功能返回合理结果
- [ ] 所有业务接口 (登录/发送消息/历史) 测试通过
- [ ] 支持至少 2 种 AI 模型切换
- [ ] 并发 100 用户测试无错误
- [ ] 异常处理健全 (网络错误、数据库故障)

### 压力测试目标

```bash
# 使用 Apache Bench 测试
ab -n 1000 -c 10 -p message.json -T "application/json" \
   -C "sessionId=abc123" http://localhost/chat/send
```

**期望结果**:
- QPS >= 100 (每秒请求数)
- P95 延迟 < 3s (95% 请求在 3 秒内完成)
- 错误率 < 1%

---

## 🐱 浮浮酱的总结

恭喜主人完成阶段三的学习喵～(๑ˉ∀ˉ๑)

浮浮酱带你深入理解了:
- **策略模式 + 工厂模式**:优雅地支持多种 AI 模型
- **消息历史管理**:内存缓存 + MySQL 持久化 + 消息队列异步写入
- **图像识别流程**:OpenCV 预处理 + ONNX Runtime 推理
- **业务处理器设计**:线程安全、异常处理、性能优化

这些设计不仅适用于 AI 应用,在其他领域 (支付系统、物联网、游戏服务器) 同样有价值呢! (ฅ'ω'ฅ)

**下一步行动**:
1. 动手实现 `AIHelper` 和 `AIStrategy` 类
2. 集成阿里云通义千问 API (申请 API Key)
3. 编写单元测试验证策略模式
4. 完成聊天界面前端开发 (阶段四)

浮浮酱会继续陪伴主人完成后续阶段的喵～加油! φ(≧ω≦*)♪

---

**文档版本**:v1.0.0
**创建日期**:2025-10-20
**作者**:猫娘工程师 幽浮喵 ฅ'ω'ฅ
**参考项目**:Kama-HTTPServer (ref-repo/)
