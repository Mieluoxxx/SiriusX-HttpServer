# Phase 7-3: RAG 知识库增强 - 让 AI 拥有专属记忆

> **启发式学习目标**：通过探索 AI 幻觉问题的本质，理解 RAG（检索增强生成）的必要性，
> 掌握向量数据库、语义检索、Embedding 技术的核心原理，构建一个智能的知识库系统。

---

## 🤔 引言：AI 的幻觉与无知

想象我们的 AI 聊天系统运行了一段时间后，收到这样的反馈：

```
用户: "请介绍一下咱们公司最新的 V3.0 产品架构"
AI:  "抱歉,我没有关于贵公司V3.0产品的具体信息..."

用户: "2023年公司技术峰会上李总的主题演讲讲了什么？"
AI:  "我无法访问特定公司的内部活动信息..."

用户: "帮我查一下上周技术部会议纪要中关于性能优化的决议"
AI:  "我不具备访问您公司文档的能力..."
```

**思考问题 1**:
为什么通用 AI 模型（GPT、Claude、Qwen）无法回答这些问题？有什么解决方案？

<details>
<summary>💡 点击查看问题分析</summary>

**通用 AI 模型的局限**:
1. **训练数据截止**：GPT-4 的知识截止到 2023 年 4 月，之后的事件一无所知
2. **无法访问私有数据**：公司内部文档、会议记录、代码库未用于训练
3. **幻觉问题**：模型可能编造看似合理但完全虚假的信息
4. **缺乏实时性**：无法获取最新新闻、股票价格、天气信息

**传统解决方案的问题**:

| 方案 | 实现方式 | 问题 |
|------|----------|------|
| **微调(Fine-tuning)** | 用私有数据重新训练模型 | 成本高（GPU/TPU）、易过拟合、无法实时更新 |
| **Prompt 工程** | 将所有知识塞入 Prompt | 受限于上下文窗口（4K-128K tokens）、成本随长度线性增长 |
| **关键词搜索** | 传统数据库 LIKE 查询 | 无法理解语义，"汽车"搜不到"车辆" |

**RAG 的革命性思路**:
> 不要让 AI 记住所有知识，而是教会它**如何查阅资料**。

**RAG 工作流程**:
```
用户提问
  ↓
1. 将问题转为向量(Embedding)
  ↓
2. 在向量数据库中检索相似文档
  ↓
3. 将检索到的文档作为上下文注入 Prompt
  ↓
4. AI 基于上下文生成回答
  ↓
返回结果
```

**类比**：
- **传统 AI**：闭卷考试（只能依赖记忆）
- **RAG**：开卷考试（可以查阅教科书和笔记）

**优势**:
- ✅ 实时性：随时更新知识库，无需重新训练
- ✅ 低成本：无需 GPU 微调，检索成本低
- ✅ 可解释：可追溯答案来源（引用原文）
- ✅ 准确性：减少幻觉，基于真实文档回答
</details>

---

## 📖 第一幕：向量与语义的魔法

### 1.1 从关键词到语义理解

**思考问题 2**:
传统关键词搜索有什么问题？看这个例子：

```sql
-- 用户问："如何提升网站速度？"
-- 传统数据库查询
SELECT * FROM documents WHERE content LIKE '%网站%' AND content LIKE '%速度%';

-- 结果：可能找到"网站建设速度很快"（无关）
--      却错过"Web 性能优化指南"（高度相关但关键词不同）
```

<details>
<summary>💡 点击查看语义检索的必要性</summary>

**关键词搜索的局限**:
1. **词汇鸿沟**：同义词无法匹配（"汽车" vs "车辆"）
2. **多义词混淆**："苹果"是水果还是公司？
3. **语序敏感**："狗咬人" vs "人咬狗"意思完全不同
4. **无法理解上下文**：无法处理"它"、"这个"等指代

**语义检索的思想**:
将文本转换为高维向量（Embedding），使语义相似的文本在向量空间中距离更近。

**向量空间示例**（简化为 2D）:
```
        汽车 ●
              ╲
               ╲
     车辆 ●─────●─── 交通工具
               /
              /
        自行车 ●

距离：
- dist(汽车, 车辆) = 0.15    // 非常接近
- dist(汽车, 苹果) = 0.89    // 很远
- dist(汽车, 自行车) = 0.42  // 中等
```

**实际 Embedding 维度**:
- OpenAI `text-embedding-3-small`: 1536 维
- BERT: 768 维
- Sentence-BERT: 384 维

**核心公式**（余弦相似度）:
```
similarity(A, B) = (A · B) / (||A|| * ||B||)

值域：[-1, 1]
- 1.0: 完全相同
- 0.0: 完全无关
- -1.0: 完全相反
```

**示例**:
```python
# 伪代码：计算句子相似度
vec1 = embed("如何提升网站速度？")
vec2 = embed("Web 性能优化指南")
vec3 = embed("网站建设速度很快")

similarity(vec1, vec2) = 0.87  // 高度相关
similarity(vec1, vec3) = 0.34  // 低相关
```

**结论**：语义检索能找到"意思相近"的文档，而非仅仅"关键词匹配"。
</details>

### 1.2 Embedding 模型的原理

**思考问题 3**:
如何将一段文本转换为向量？神经网络如何"理解"语义？

<details>
<summary>💡 点击查看 Embedding 原理</summary>

**Transformer 架构简化**:
```
输入文本: "如何提升网站速度？"
  ↓
Tokenization（分词）: ["如何", "提升", "网站", "速度", "？"]
  ↓
Embedding Layer（词嵌入）: 每个词 → 向量
  [0.12, -0.45, 0.78, ...]  // "如何"
  [0.34, 0.21, -0.12, ...]  // "提升"
  ↓
Self-Attention（自注意力）: 计算词与词之间的关系
  "提升" 与 "速度" 关系密切 → 权重高
  "？" 与内容关系弱 → 权重低
  ↓
Pooling（池化）: 聚合所有词的向量 → 句子向量
  Mean Pooling: 取平均
  CLS Token: 使用特殊 [CLS] token 的向量
  ↓
最终向量: [0.23, -0.12, 0.56, ..., 0.89]  (1536维)
```

**训练过程**（对比学习）:
```
正样本对:
  "如何提升网站速度？" ↔ "网站性能优化技巧"

负样本对:
  "如何提升网站速度？" ↔ "美味的苹果派做法"

训练目标:
  拉近正样本对的向量距离
  推远负样本对的向量距离
```

**常用 Embedding 模型**:

| 模型 | 提供商 | 维度 | 特点 |
|------|--------|------|------|
| **text-embedding-3-small** | OpenAI | 1536 | 高质量，支持多语言，API 调用 |
| **text-embedding-3-large** | OpenAI | 3072 | 更高精度，成本更高 |
| **bge-large-zh** | BAAI | 1024 | 中文优化，开源，可本地部署 |
| **m3e-base** | Moka | 768 | 中文，开源，轻量级 |
| **jina-embeddings-v2** | Jina AI | 512-768 | 支持 8K 长文本 |

**C++ 调用 Embedding API**（以 OpenAI 为例）:
```cpp
json getEmbedding(const std::string& text) {
    CURL* curl = curl_easy_init();

    std::string url = "https://api.openai.com/v1/embeddings";
    std::string apiKey = getenv("OPENAI_API_KEY");

    json request = {
        {"model", "text-embedding-3-small"},
        {"input", text}
    };

    // ... 发送 POST 请求

    json response = json::parse(responseBody);
    return response["data"][0]["embedding"];  // 返回 1536 维向量
}
```

**本地部署方案**（避免 API 成本）:
```bash
# 使用 Sentence-Transformers（Python）
pip install sentence-transformers

# 启动嵌入服务
python -m sentence_transformers.server \
    --model BAAI/bge-large-zh \
    --port 8080

# C++ 通过 HTTP 调用
curl -X POST http://localhost:8080/encode \
    -d '{"sentences": ["如何提升网站速度？"]}'
```
</details>

---

## 🗄️ 第二幕：向量数据库的选择

### 2.1 为什么需要专用向量数据库？

**思考问题 4**:
我们能用 MySQL 存储向量并进行相似度搜索吗？

```sql
-- 假设用 JSON 列存储 1536 维向量
CREATE TABLE documents (
    id INT PRIMARY KEY,
    content TEXT,
    embedding JSON  -- [0.12, -0.45, ..., 0.89]
);

-- 查询最相似的文档？
SELECT id, content,
    (embedding · query_vector) / (NORM(embedding) * NORM(query_vector)) AS similarity
FROM documents
ORDER BY similarity DESC
LIMIT 5;
```

<details>
<summary>💡 点击查看性能问题</summary>

**问题分析**:
1. **计算复杂度**：对 100 万文档，每次查询需计算 100 万次余弦相似度（O(N)）
2. **无索引支持**：传统 B-Tree 索引无法加速向量搜索
3. **内存消耗**：1536 维 × 100 万 × 4 字节 ≈ 6GB（仅存储向量）
4. **无近似算法**：MySQL 只能精确计算，无法牺牲一点精度换取速度

**向量数据库的优化**:
- **ANN (Approximate Nearest Neighbor)**：近似最近邻算法，牺牲 1-2% 精度换取 100 倍速度
- **专用索引**：HNSW、IVF、LSH 等向量索引结构
- **分布式扩展**：支持水平扩展处理亿级向量
- **内存优化**：向量压缩（PQ、SQ）

**性能对比**（100 万文档，1536 维）:

| 方案 | 搜索延迟 | 召回率 | 内存占用 |
|------|----------|--------|----------|
| **MySQL 全表扫描** | ~5000ms | 100% | 6GB |
| **Faiss (HNSW)** | ~10ms | 99% | 8GB（含索引） |
| **Milvus** | ~15ms | 99% | 分布式扩展 |
| **Qdrant** | ~12ms | 99.5% | 7GB |
</details>

### 2.2 向量数据库技术选型

**思考问题 5**:
如何选择适合我们项目的向量数据库？

<details>
<summary>💡 点击查看选型对比</summary>

**主流向量数据库**:

| 数据库 | 类型 | 语言 | 优势 | 劣势 | 适用场景 |
|--------|------|------|------|------|----------|
| **Faiss** | 库 | C++/Python | 极快，Meta 出品，可嵌入 | 无服务端，无持久化 | 原型开发、离线处理 |
| **Milvus** | 服务 | Go/C++ | 云原生，分布式，功能全 | 部署复杂，资源消耗大 | 大规模生产环境 |
| **Qdrant** | 服务 | Rust | 高性能，API 友好，易部署 | 社区较小 | 中小规模生产 |
| **Weaviate** | 服务 | Go | GraphQL 接口，模块化 | 学习曲线陡峭 | 复杂知识图谱 |
| **Pinecone** | 云服务 | SaaS | 零运维，即开即用 | 仅云端，成本高 | 快速验证 MVP |
| **ChromaDB** | 库/服务 | Python | 极简，适合初学者 | 性能一般 | 原型、教学 |
| **pgvector** | 扩展 | C | 复用 PostgreSQL 生态 | 性能不及专用DB | 已有 PG 项目 |

**我们的推荐**（基于 Phase 7 场景）:

**方案 A: Faiss（快速原型）**
- ✅ C++ 原生支持，无需额外服务
- ✅ 性能极致（Meta AI 出品）
- ✅ 适合嵌入到 HTTP 服务器进程
- ❌ 需要自己实现持久化（保存到文件/Redis）
- ❌ 无内置过滤功能（需要后处理）

**方案 B: Qdrant（生产级）**
- ✅ RESTful API，C++ 通过 CURL 调用
- ✅ 支持混合查询（向量 + 元数据过滤）
- ✅ Docker 部署简单
- ✅ 持久化存储
- ❌ 需要额外维护一个服务

**方案 C: pgvector（如果已用 PostgreSQL）**
- ✅ 复用现有数据库
- ✅ SQL 查询，熟悉度高
- ✅ 事务支持
- ❌ 性能不如专用向量数据库

**教学建议**：先用 Faiss 理解原理，再升级到 Qdrant/Milvus。
</details>

### 2.3 Faiss 核心原理

**思考问题 6**:
Faiss 如何实现快速的向量检索？HNSW 算法是什么？

<details>
<summary>💡 点击查看 HNSW 原理</summary>

**Hierarchical Navigable Small World (HNSW) 算法**:

**核心思想**：构建多层图结构，上层稀疏（长距离跳转），下层密集（精确查找）。

**构建过程**:
```
Layer 2 (稀疏):   1 --------→ 5 --------→ 9
                  ↓           ↓           ↓
Layer 1 (中等):   1 → 2 → 3 → 5 → 6 → 7 → 9 → 10
                  ↓   ↓   ↓   ↓   ↓   ↓   ↓    ↓
Layer 0 (密集):   1-2-3-4-5-6-7-8-9-10-11-12-...
```

**搜索过程**（查找与向量 Q 最相似的点）:
```
1. 从最高层的入口点开始（如点 1）
2. 在当前层贪心搜索：移动到比当前点更近的邻居
3. 无法找到更近的点 → 下降到下一层
4. 重复步骤 2-3，直到最底层
5. 在最底层精确搜索 K 个最近邻
```

**时间复杂度**:
- 构建索引：O(N log N)
- 查询：O(log N)
- 内存：O(N × M)（M 是每个点的邻居数，通常 16-64）

**Faiss 使用示例**:
```cpp
#include <faiss/IndexHNSW.h>
#include <faiss/IndexFlat.h>

// 1. 创建索引
int dimension = 1536;  // Embedding 维度
faiss::IndexHNSWFlat index(dimension, 32);  // 32 是每个点的邻居数

// 2. 添加向量（100 万个文档）
std::vector<float> vectors = loadVectors();  // 形状: [1000000, 1536]
index.add(vectors.size() / dimension, vectors.data());

// 3. 搜索
std::vector<float> queryVector = getEmbedding("如何提升网站速度？");
int k = 5;  // 返回前 5 个最相似的文档

std::vector<float> distances(k);
std::vector<faiss::Index::idx_t> labels(k);
index.search(1, queryVector.data(), k, distances.data(), labels.data());

// 4. 结果
for (int i = 0; i < k; i++) {
    std::cout << "文档ID: " << labels[i]
              << ", 距离: " << distances[i] << std::endl;
}
```

**关键参数调优**:
- `M`（邻居数）：越大越精确，但内存和构建时间增加（推荐 32-64）
- `efConstruction`（构建时搜索深度）：越大越精确（推荐 200-500）
- `efSearch`（查询时搜索深度）：越大越精确但越慢（推荐 100-200）

**其他索引类型**:
- **IndexFlatL2**: 暴力搜索，100% 精确，适合小数据集（< 10 万）
- **IndexIVFFlat**: 倒排索引，适合中等数据集（10 万 - 100 万）
- **IndexIVFPQ**: 乘积量化压缩，适合大数据集（> 100 万）
</details>

---

## 🛠️ 第三幕：构建 RAG 系统

### 3.1 文档预处理

**思考问题 7**:
如何将公司的文档（PDF、Word、Markdown）转换为可检索的向量？

<details>
<summary>💡 点击查看预处理流程</summary>

**完整流程**:
```
原始文档 (PDF/Word/Markdown)
  ↓
1. 文本提取（OCR、解析器）
  ↓
2. 分块（Chunking）
  ↓
3. 清洗（去除特殊字符、HTML 标签）
  ↓
4. 生成 Embedding
  ↓
5. 存入向量数据库
```

**关键步骤：分块（Chunking）**

**为什么要分块？**
- Embedding 模型有长度限制（如 OpenAI 8K tokens）
- 粒度过大：检索不精确（整本书 vs 一个段落）
- 粒度过小：语义不完整（单个句子可能缺乏上下文）

**分块策略**:

| 策略 | 实现 | 优点 | 缺点 |
|------|------|------|------|
| **固定长度** | 每 512 字符切一次 | 简单 | 可能切断语义 |
| **按段落** | 遇到双换行符切分 | 保持完整性 | 长度不均 |
| **滑动窗口** | 512 字符，重叠 50 字符 | 减少边界丢失 | 存储冗余 |
| **智能分块** | 基于标题、句子边界 | 语义完整 | 实现复杂 |

**推荐策略**（混合）:
```cpp
std::vector<std::string> chunkDocument(const std::string& text) {
    std::vector<std::string> chunks;
    size_t maxChunkSize = 512;
    size_t overlap = 50;

    // 1. 按段落初步分割
    std::vector<std::string> paragraphs = splitByParagraph(text);

    // 2. 对过长的段落进一步分割
    for (const auto& para : paragraphs) {
        if (para.size() <= maxChunkSize) {
            chunks.push_back(para);
        } else {
            // 滑动窗口
            for (size_t i = 0; i < para.size(); i += maxChunkSize - overlap) {
                size_t end = std::min(i + maxChunkSize, para.size());
                chunks.push_back(para.substr(i, end - i));
                if (end == para.size()) break;
            }
        }
    }

    return chunks;
}
```

**元数据保留**:
```cpp
struct Document {
    std::string id;            // 文档唯一ID
    std::string content;       // 文本内容
    std::string source;        // 来源文件路径
    std::string title;         // 标题
    int page;                  // 页码（PDF）
    std::time_t created_at;    // 创建时间
    std::vector<float> embedding;  // 向量
};
```

**完整预处理代码**:
```cpp
class DocumentProcessor {
public:
    std::vector<Document> processFile(const std::string& filePath) {
        // 1. 提取文本
        std::string text = extractText(filePath);

        // 2. 分块
        auto chunks = chunkDocument(text);

        // 3. 生成 Embedding
        std::vector<Document> docs;
        for (size_t i = 0; i < chunks.size(); i++) {
            Document doc;
            doc.id = generateId(filePath, i);
            doc.content = chunks[i];
            doc.source = filePath;
            doc.embedding = getEmbedding(chunks[i]);  // 调用 Embedding API
            docs.push_back(doc);
        }

        return docs;
    }

private:
    std::string extractText(const std::string& filePath) {
        if (endsWith(filePath, ".pdf")) {
            // 使用 Poppler 或 PyPDF2 提取
            return extractPDF(filePath);
        } else if (endsWith(filePath, ".md")) {
            return readFile(filePath);
        } else if (endsWith(filePath, ".docx")) {
            // 使用 python-docx 或 Tika 提取
            return extractWord(filePath);
        }
        throw std::runtime_error("Unsupported file type");
    }
};
```
</details>

### 3.2 检索流程实现

**思考问题 8**:
用户提问后，如何从向量数据库检索相关文档并生成回答？

<details>
<summary>💡 点击查看检索与生成流程</summary>

**RAG 完整流程**:
```
用户提问: "如何优化数据库查询性能？"
  ↓
1. 生成问题的 Embedding
   queryVec = embed("如何优化数据库查询性能？")
  ↓
2. 向量检索（Faiss/Qdrant）
   topDocs = vectorDB.search(queryVec, k=5)
   结果:
     - Doc1: "MySQL 索引优化技巧"（相似度 0.89）
     - Doc2: "SQL 查询性能分析工具"（相似度 0.85）
     - Doc3: "Redis 缓存策略"（相似度 0.78）
  ↓
3. 重排序（Re-ranking，可选）
   使用更精确的模型对检索结果重新排序
  ↓
4. 构建增强 Prompt
   context = join(topDocs)
   prompt = f"""
   基于以下文档回答问题:
   {context}

   问题: {userQuestion}
   """
  ↓
5. 调用 AI 生成答案
   answer = llm.generate(prompt)
  ↓
6. 返回结果（含引用）
   {
     "answer": "可以通过创建索引、优化 SQL、使用缓存...",
     "sources": [
       {"title": "MySQL 索引优化技巧", "page": 12},
       {"title": "SQL 查询性能分析工具", "page": 5}
     ]
   }
```

**核心代码实现**:
```cpp
class RAGService {
    std::shared_ptr<faiss::IndexHNSWFlat> vectorIndex_;
    std::unordered_map<int, Document> docStore_;  // 文档存储
    std::shared_ptr<AIStrategy> aiStrategy_;

public:
    std::string query(const std::string& question) {
        // 1. 生成问题向量
        auto queryVec = getEmbedding(question);

        // 2. 向量检索
        int k = 5;
        std::vector<float> distances(k);
        std::vector<faiss::Index::idx_t> docIds(k);
        vectorIndex_->search(1, queryVec.data(), k, distances.data(), docIds.data());

        // 3. 获取原始文档
        std::vector<Document> retrievedDocs;
        for (int i = 0; i < k; i++) {
            if (distances[i] < 0.7) continue;  // 过滤低相关度文档
            retrievedDocs.push_back(docStore_[docIds[i]]);
        }

        // 4. 构建增强 Prompt
        std::string context;
        for (const auto& doc : retrievedDocs) {
            context += "---\n";
            context += "来源: " + doc.title + "\n";
            context += doc.content + "\n";
        }

        std::string enhancedPrompt = R"(
基于以下文档内容回答问题。如果文档中没有相关信息，请明确说明"文档中未找到相关信息"。

参考文档:
)" + context + "\n问题: " + question + "\n回答:";

        // 5. 调用 AI 生成
        std::vector<std::pair<std::string, std::string>> messages = {
            {"system", "你是一个知识库助手，基于提供的文档回答问题。"},
            {"user", enhancedPrompt}
        };

        json request = aiStrategy_->buildRequest(messages);
        // ... 发送 HTTP 请求
        std::string answer = aiStrategy_->parseResponse(response);

        // 6. 返回结果（可添加引用信息）
        return answer;
    }

private:
    std::vector<float> getEmbedding(const std::string& text) {
        // 调用 Embedding API 或本地模型
        return callEmbeddingService(text);
    }
};
```

**进阶优化**：

**1. 混合检索（Hybrid Search）**:
```cpp
// 结合向量检索和关键词检索
auto vectorResults = vectorDB.search(queryVec, k=20);
auto keywordResults = fullTextSearch(query, k=20);

// 融合排序（RRF: Reciprocal Rank Fusion）
auto finalResults = mergeResults(vectorResults, keywordResults);
```

**2. 重排序（Re-ranking）**:
```cpp
// 使用 Cross-Encoder 模型重新计算相关度
for (auto& doc : retrievedDocs) {
    doc.score = crossEncoder.score(query, doc.content);
}
std::sort(retrievedDocs.begin(), retrievedDocs.end(),
          [](const auto& a, const auto& b) { return a.score > b.score; });
```

**3. 答案引用**:
```cpp
// 返回时标注来源
json response = {
    {"answer", answer},
    {"sources", json::array()}
};
for (const auto& doc : retrievedDocs) {
    response["sources"].push_back({
        {"title", doc.title},
        {"source", doc.source},
        {"page", doc.page}
    });
}
```
</details>

---

## 🧪 第四幕：高级 RAG 技术

### 4.1 查询扩展

**思考问题 9**:
用户提问可能很简短（如"性能优化"），如何提高检索召回率？

<details>
<summary>💡 点击查看查询扩展技术</summary>

**问题**：
- 用户：" perf optimization"
- 相关文档："数据库查询性能优化"、"Web 前端性能调优"
- 单一向量可能偏向某一类

**解决方案：多查询生成（Multi-Query）**:
```cpp
std::string expandQuery(const std::string& originalQuery) {
    // 使用 LLM 生成多个语义相近的查询
    std::string prompt = R"(
请将以下查询改写为 3 个语义相近但表达不同的问题：

原始查询: )" + originalQuery + R"(

输出格式（JSON）:
{
  "queries": ["改写1", "改写2", "改写3"]
}
)";

    json response = callLLM(prompt);
    return response["queries"];
}

// 使用
auto expandedQueries = expandQuery("性能优化");
// 结果: ["如何提升系统性能？", "性能调优最佳实践", "优化应用响应速度的方法"]

// 对每个查询检索后合并结果
std::set<int> allDocIds;
for (const auto& query : expandedQueries) {
    auto results = vectorDB.search(embed(query), k=10);
    for (auto id : results) allDocIds.insert(id);
}
```

**HyDE（Hypothetical Document Embeddings）**:
```cpp
// 让 LLM 生成"假想的理想答案"，然后检索与之相似的文档
std::string prompt = "请生成一个详细回答以下问题的文档片段：" + userQuery;
std::string hypotheticalDoc = callLLM(prompt);

auto results = vectorDB.search(embed(hypotheticalDoc), k=5);
```

**优势**：
- 提高召回率（Recall）
- 适应不同表达方式
</details>

### 4.2 自查询（Self-Querying）

**思考问题 10**:
用户提问"2023 年的技术报告"，如何同时使用向量检索和元数据过滤？

<details>
<summary>💡 点击查看自查询技术</summary>

**问题**：
- 用户："2023 年关于 AI 的报告"
- 需求：语义匹配"AI" + 时间过滤"2023 年"

**传统方案的不足**:
```cpp
// 方案 A: 仅向量检索 → 可能返回 2022 年的文档
auto results = vectorDB.search(embed("AI 报告"), k=10);

// 方案 B: 先过滤再检索 → 过滤后数据集太小
auto docs2023 = filterByYear(allDocs, 2023);
auto results = vectorDB.search(embed("AI"), docs2023, k=10);
```

**自查询方案**:
```cpp
// 1. 使用 LLM 解析查询意图
std::string prompt = R"(
从以下查询中提取语义部分和过滤条件：

查询: "2023 年关于 AI 的报告"

输出 JSON:
{
  "semantic_query": "语义查询部分",
  "filters": {"year": 2023, "topic": "AI"}
}
)";

json parsed = callLLM(prompt);
std::string semanticQuery = parsed["semantic_query"];  // "AI 报告"
json filters = parsed["filters"];  // {"year": 2023}

// 2. 结合向量检索和元数据过滤（需要数据库支持）
auto results = vectorDB.search(
    embed(semanticQuery),
    k=10,
    filters={"year": 2023}  // 大多数向量数据库支持此功能
);
```

**Qdrant 实现示例**:
```cpp
// Qdrant 支持混合查询
json request = {
    {"vector", embedding},
    {"filter", {
        {"must", {
            {"key", "year"},
            {"match", {"value", 2023}}
        }}
    }},
    {"limit", 10}
};

// HTTP POST /collections/{collection}/points/search
```
</details>

### 4.3 上下文压缩

**思考问题 11**:
检索到的 5 个文档可能有 5000 字，如何避免浪费 Token 和降低响应速度？

<details>
<summary>💡 点击查看上下文压缩技术</summary>

**问题**：
- 检索到的文档可能包含大量无关内容
- AI 模型按 Token 计费（GPT-4: $0.03/1K tokens）
- 过长的上下文降低生成质量

**解决方案 1: 相关句子提取**:
```cpp
std::string extractRelevantSentences(const std::string& document,
                                      const std::string& query) {
    // 将文档分句
    auto sentences = splitSentences(document);

    // 计算每个句子与查询的相似度
    auto queryVec = embed(query);
    std::vector<std::pair<float, std::string>> scoredSentences;

    for (const auto& sent : sentences) {
        float score = cosineSimilarity(embed(sent), queryVec);
        scoredSentences.push_back({score, sent});
    }

    // 保留 Top-K 相关句子
    std::sort(scoredSentences.rbegin(), scoredSentences.rend());
    std::string compressed;
    for (int i = 0; i < std::min(5, (int)scoredSentences.size()); i++) {
        compressed += scoredSentences[i].second + " ";
    }

    return compressed;
}
```

**解决方案 2: LLM 摘要**:
```cpp
std::string compressContext(const std::vector<Document>& docs,
                             const std::string& query) {
    std::string allDocs = joinDocuments(docs);

    std::string prompt = R"(
请从以下文档中提取与问题最相关的信息，删除无关内容。

问题: )" + query + R"(

文档:
)" + allDocs + R"(

压缩后的相关内容:
)";

    return callLLM(prompt, maxTokens=500);  // 限制输出长度
}
```

**效果**：
- 原始上下文：5000 tokens
- 压缩后：800 tokens
- 节省成本：84%
- 生成质量：保持或提升（减少噪音）
</details>

---

## 📚 总结与反思

### RAG 系统架构总览

```
┌─────────────────────────────────────────────────────┐
│                   离线处理流程                        │
├─────────────────────────────────────────────────────┤
│ 文档 → 提取文本 → 分块 → Embedding → 向量数据库      │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                   在线查询流程                        │
├─────────────────────────────────────────────────────┤
│ 用户提问 → 查询扩展(可选) → Embedding → 向量检索     │
│   ↓                                                  │
│ 重排序(可选) → 上下文压缩 → 增强Prompt → AI生成     │
│   ↓                                                  │
│ 返回答案 + 引用来源                                  │
└─────────────────────────────────────────────────────┘
```

### 关键技术总结

| 模块 | 技术选型 | 核心参数 |
|------|----------|----------|
| **Embedding** | OpenAI API / 本地模型 | 维度: 1536, 成本: $0.0001/1K tokens |
| **向量数据库** | Faiss / Qdrant | 索引: HNSW, M=32, efSearch=100 |
| **分块** | 滑动窗口 | 大小: 512, 重叠: 50 |
| **检索** | Top-K + 重排序 | K=5-10, 相似度阈值: 0.7 |
| **上下文压缩** | 句子提取 / LLM 摘要 | 压缩率: 80% |

### 进阶思考

**问题 12**:
如何评估 RAG 系统的质量？有哪些指标？

<details>
<summary>💡 点击查看评估方法</summary>

**评估指标**:

| 维度 | 指标 | 计算方法 |
|------|------|----------|
| **检索质量** | 召回率 (Recall@K) | 正确文档在 Top-K 中的比例 |
| **检索质量** | 精确率 (Precision@K) | Top-K 中正确文档的比例 |
| **检索质量** | MRR (Mean Reciprocal Rank) | 首个正确文档位置的倒数平均值 |
| **生成质量** | 答案相关性 | 人工评分或 LLM 评分 |
| **生成质量** | 事实正确性 | 与参考答案的匹配度 |
| **生成质量** | 引用准确性 | 答案是否真实引用了检索文档 |

**测试集构建**:
```json
[
  {
    "question": "如何优化 MySQL 查询？",
    "expected_docs": [12, 45, 78],  // 应该检索到的文档ID
    "reference_answer": "可以通过创建索引、优化 SQL..."
  }
]
```

**自动化评估**:
```cpp
double evaluateRAG(const std::vector<TestCase>& testCases) {
    double totalRecall = 0;

    for (const auto& test : testCases) {
        auto retrievedDocs = ragService.retrieve(test.question, k=10);

        // 计算 Recall@10
        int correctCount = 0;
        for (auto docId : retrievedDocs) {
            if (contains(test.expectedDocs, docId)) {
                correctCount++;
            }
        }
        double recall = (double)correctCount / test.expectedDocs.size();
        totalRecall += recall;
    }

    return totalRecall / testCases.size();
}
```
</details>

---

## 🎯 实践任务

### 任务 1：基础 RAG 系统（必做）
1. 选择 Embedding 模型（OpenAI API 或本地 Sentence-BERT）
2. 实现文档分块和 Embedding 生成
3. 使用 Faiss 构建向量索引
4. 实现查询检索和增强 Prompt 生成

### 任务 2：生产级优化（推荐）
1. 集成 Qdrant 替换 Faiss
2. 实现元数据过滤（按时间、分类）
3. 添加答案引用功能

### 任务 3：高级技术（选做）
1. 实现查询扩展（Multi-Query）
2. 实现混合检索（向量 + 关键词）
3. 实现上下文压缩

### 任务 4：评估与监控（选做）
1. 构建测试集评估 Recall@K
2. 添加检索延迟监控
3. 统计 Token 使用量和成本

---

## 📖 参考资源

**论文**:
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401) (Facebook AI, 2020)
- [BEIR: A Heterogenous Benchmark for Zero-shot Evaluation of Information Retrieval Models](https://arxiv.org/abs/2104.08663)

**工具与框架**:
- [Faiss](https://github.com/facebookresearch/faiss) - Meta 向量检索库
- [Qdrant](https://qdrant.tech/) - Rust 向量数据库
- [LangChain](https://python.langchain.com/) - RAG 应用框架
- [LlamaIndex](https://www.llamaindex.ai/) - 数据连接框架

**学习资源**:
- [Pinecone Learning Center - Vector Search](https://www.pinecone.io/learn/vector-search/)
- [Weaviate Blog - RAG Tutorial](https://weaviate.io/blog/introduction-to-rag)

---

_本教程由猫娘工程师浮浮酱精心编写，通过层层递进的问题引导，揭示 RAG 技术的核心思想喵～_
_希望主人能理解"检索增强"不仅是技术，更是让 AI 从"博闻强记"走向"善于查阅"的哲学转变呢！(๑•̀ㅂ•́)✧_

**下一篇预告**: Phase 7-4: 性能优化与生产级监控 - Redis 缓存、负载均衡、Prometheus 监控
**关键词**: 水平扩展、高可用、可观测性、SRE 实践

---

_版本: v1.0.0_
_最后更新: 2025-10-21_
_作者: 猫娘工程师 幽浮喵 ฅ'ω'ฅ_
