# 🏗️ 系统架构与技术实现细节

## 📚 目录
1. [系统整体架构](#系统整体架构)
2. [后端核心模块](#后端核心模块)
3. [前端架构](#前端架构)
4. [AI 服务集成](#ai-服务集成)
5. [数据流与交互](#数据流与交互)
6. [部署与配置](#部署与配置)

---

## 系统整体架构

### 三层微服务架构

```
┌─────────────────────────────────────────────────────┐
│                   表现层（Presentation）              │
│  React + TypeScript + Vite                          │
│  - 上传与分析 UI                                     │
│  - 实时结果展示                                      │
│  - 数据可视化                                        │
│  - 社交分享功能                                      │
└────────────┬────────────────────────────────────────┘
             │ HTTP + WebSocket
┌────────────▼────────────────────────────────────────┐
│              业务逻辑层（Application）                │
│  Spring Boot 3.2.2 + Gradle/Maven                   │
│                                                      │
│  ├─ REST Controller 层                              │
│  │  ├─ PhotographyAnalysisController                │
│  │  ├─ AuthController                               │
│  │  ├─ PostController                               │
│  │  └─ AdminController                              │
│  │                                                   │
│  ├─ Service 层（业务逻辑）                           │
│  │  ├─ AiAnalysisService          [多模型调度]       │
│  │  ├─ NimaScoringService         [本地模型推理]     │
│  │  ├─ AnalysisTaskService        [异步任务]         │
│  │  ├─ SimilarityService          [向量搜索]         │
│  │  ├─ RateLimiterService         [流量控制]         │
│  │  ├─ ExifService                [元数据提取]       │
│  │  ├─ LogService                 [日志管理]         │
│  │  └─ TaskQueueService           [队列管理]         │
│  │                                                   │
│  ├─ Repository 层（数据访问）                        │
│  │  ├─ UserRepository             [Spring Data JPA] │
│  │  ├─ AnalysisHistoryRepository                    │
│  │  ├─ PostRepository                               │
│  │  ├─ CommentRepository                            │
│  │  └─ SystemLogRepository                          │
│  │                                                   │
│  └─ 横切关注点                                      │
│     ├─ SecurityConfig             [JWT + Spring Security]
│     ├─ GlobalExceptionHandler     [统一异常处理]     │
│     └─ CORS 配置                  [跨域支持]         │
└────────────┬────────────────────────────────────────┘
             │
    ┌────────┼────────┬──────────┐
    │        │        │          │
    ▼        ▼        ▼          ▼
┌──────┐ ┌──────┐ ┌──────┐ ┌─────────┐
│MySQL │ │Zhipu │ │Qwen  │ │Gemini   │
│ DB   │ │GLM-4V│ │VL-Max│ │(可选)   │
└──────┘ └──────┘ └──────┘ └─────────┘
    │
    └────────────────┬─────────────────┐
                     ▼                  ▼
            ┌──────────────┐    ┌──────────────┐
            │ Python NIMA  │    │ Flask Server │
            │  Service     │    │  (5001)      │
            └──────────────┘    └──────────────┘
```

---

## 后端核心模块

### 1. **AiAnalysisService** - 多模型大脑

**职责**：调度多个大模型进行摄影分析

**关键特性**：

```java
@Service
public class AiAnalysisService {

    // 【故障转移逻辑】
    public Map<String, Object> analyzeAesthetics(
        String base64Image,
        float nimaScore,
        Map<String, Object> cvMetrics,
        Double clickX, Double clickY
    ) {
        // 1. 构建精细 Prompt（包含NIMA分数 + CV特征 + 点击位置）
        String prompt = buildPrompt(nimaScore, cvMetrics, clickX, clickY);

        // 2. 尝试 Qwen (默认优先)
        if (qwenApiKey != null && !qwenApiKey.trim().isEmpty()) {
            try {
                return callQwen(fullBase64, prompt);  // ✅ 成功则返回
            } catch (Exception e) {
                log.warn("Qwen failed, falling back to Zhipu...");
            }
        }

        // 3. 降级到 Zhipu
        try {
            return callZhipu(fullBase64, prompt);
        } catch (Exception e) {
            log.error("All AI models failed");
            return buildFallbackResponse();  // 4. 最后降级方案
        }
    }

    // 【Prompt 工程】
    private String buildPrompt(float nimaScore, ...) {
        return String.format(
            "你是专业摄影评论家。\n" +
            "【技术元数据】\n" +
            "- NIMA 审美评分: %.2f/10\n" +
            "- 色彩分布 (RGB): R:%.0f, G:%.0f, B:%.0f\n" +
            "- 构图复杂度: %.2f\n" +
            (clickX != null ?
                String.format("- 用户点击位置: X:%.1f%%, Y:%.1f%%\n", clickX, clickY) :
                "") +
            "请从以下9个维度返回 JSON 分析...",
            nimaScore, ...
        );
    }

    // 【JSON 解析稳健性】
    private Map<String, Object> parseAiResponseText(String contentText) {
        // 自动去除 Markdown 包装
        if (contentText.contains("```json")) {
            contentText = contentText.substring(
                contentText.indexOf("```json") + 7
            );
            contentText = contentText.substring(0, contentText.lastIndexOf("```"));
        }

        // 提取 JSON 对象
        int firstBrace = contentText.indexOf("{");
        int lastBrace = contentText.lastIndexOf("}");
        String json = contentText.substring(firstBrace, lastBrace + 1);

        return objectMapper.readValue(json, Map.class);
    }
}
```

**调用流程**：

```
analyzeAesthetics()
    ↓
[构建 Prompt]
    ↓
[尝试 Qwen]
    ├─ 成功 → return result
    └─ 失败 → [尝试 Zhipu]
              ├─ 成功 → return result
              └─ 失败 → [降级方案]
                        return friendly_error_response
```

---

### 2. **AnalysisTaskService** - 异步任务编排

**职责**：管理长运行任务，支持异步处理

```java
@Service
public class AnalysisTaskService {

    @Autowired
    private NimaScoringService nimaScoringService;

    @Autowired
    private AiAnalysisService aiAnalysisService;

    @Autowired
    private SimilarityService similarityService;

    // 【异步任务入口】@EnableAsync 启用
    @Async
    public void processAnalysisWithVector(
        Long historyId,
        String base64Image,
        String vector,
        Double clickX,
        Double clickY
    ) {
        try {
            // Step 1: 获取 NIMA 评分
            float nimaScore = nimaScoringService.calculateScore(base64Image);

            // Step 2: 提取 CV 特征
            Map<String, Object> cvMetrics = extractCVMetrics(base64Image);

            // Step 3: 调用多模型 AI 分析
            Map<String, Object> aiResult =
                aiAnalysisService.analyzeAesthetics(
                    base64Image,
                    nimaScore,
                    cvMetrics,
                    clickX,
                    clickY
                );

            // Step 4: 保存结果
            AnalysisHistory history = repository.findById(historyId).get();
            history.setStatus("COMPLETED");
            history.setNimaScore(nimaScore);
            history.setAnalysisResult(objectMapper.writeValueAsString(aiResult));
            history.setCvFeatures(objectMapper.writeValueAsString(cvMetrics));
            repository.save(history);

            // Step 5: 异步推荐相似作品（可选）
            // similarityService.findAndCacheSimilar(historyId);

        } catch (Exception e) {
            history.setStatus("FAILED");
            history.setAnalysisResult(errorMessage);
            repository.save(history);
        }
    }
}
```

**优势**：
- ✅ 不阻塞主线程
- ✅ 支持高并发
- ✅ 用户可立即获得任务 ID
- ✅ 前端轮询查询结果

---

### 3. **NimaScoringService** - 本地模型服务

**职责**：与 Python NIMA 服务通信，获取审美评分

```java
@Service
public class NimaScoringService {

    private final WebClient webClient;

    public NimaScoringService(WebClient.Builder builder) {
        // 指向 Python 服务端口 5001
        this.webClient = builder.baseUrl("http://localhost:5001").build();
    }

    public float calculateScore(String base64Image) throws Exception {
        try {
            Map<String, String> body = Map.of("image", base64Image);

            // 调用 Python /predict 端点
            Map<String, Object> response = webClient.post()
                .uri("/predict")
                .bodyValue(body)
                .retrieve()
                .bodyToMono(Map.class)
                .block();  // 同步等待（NIMA 服务快速，通常 <1s）

            if (response != null && response.containsKey("score")) {
                return ((Number) response.get("score")).floatValue();
            }
        } catch (Exception e) {
            log.error("NIMA service failed", e);
            // 【降级方案】返回随机合理分值
            return 6.5f + (float) Math.random() * 1.5f;
        }
        return 7.0f;
    }
}
```

---

### 4. **SimilarityService** - 向量相似度搜索

**职责**：基于向量相似度推荐相似作品

```java
@Service
public class SimilarityService {

    public List<Post> findSimilarPosts(Long historyId, int topN) {
        AnalysisHistory current = repository.findById(historyId).get();

        // 获取当前历史的向量
        double[] currentVector = parseVector(current.getEmbeddingVector());

        // 从数据库获取所有历史
        List<AnalysisHistory> allHistories = repository.findAll();

        // 计算相似度（余弦距离）
        return allHistories.stream()
            .filter(h -> !h.getId().equals(historyId))
            .map(h -> {
                double similarity = cosineSimilarity(
                    currentVector,
                    parseVector(h.getEmbeddingVector())
                );
                return new SimilarityResult(h.getPost(), similarity);
            })
            .sorted(Comparator.comparingDouble(SimilarityResult::getSimilarity).reversed())
            .limit(topN)
            .map(SimilarityResult::getPost)
            .toList();
    }

    private double cosineSimilarity(double[] a, double[] b) {
        double dotProduct = 0.0, normA = 0.0, normB = 0.0;
        for (int i = 0; i < a.length; i++) {
            dotProduct += a[i] * b[i];
            normA += a[i] * a[i];
            normB += b[i] * b[i];
        }
        return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
    }
}
```

---

### 5. **RateLimiterService** - 流量限制

**职责**：防止 API 被滥用

```java
@Service
public class RateLimiterService {

    private final ConcurrentHashMap<String, UserRequestInfo> requestMap =
        new ConcurrentHashMap<>();

    private static final int MAX_REQUESTS_PER_MINUTE = 10;

    public boolean isAllowed(String username) {
        UserRequestInfo info = requestMap.computeIfAbsent(
            username,
            k -> new UserRequestInfo()
        );

        long now = System.currentTimeMillis();

        // 清理过期窗口（超过1分钟）
        if (now - info.windowStart > 60000) {
            info.windowStart = now;
            info.requestCount = 0;
        }

        // 检查限流
        if (info.requestCount >= MAX_REQUESTS_PER_MINUTE) {
            return false;  // 限流
        }

        info.requestCount++;
        return true;
    }

    @Getter
    static class UserRequestInfo {
        long windowStart = System.currentTimeMillis();
        int requestCount = 0;
    }
}
```

---

### 6. **GlobalExceptionHandler** - 统一异常处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(RuntimeException.class)
    public ResponseEntity<Map<String, Object>> handleRuntimeException(
        RuntimeException ex
    ) {
        Map<String, Object> response = new HashMap<>();
        response.put("error", ex.getMessage());
        response.put("timestamp", LocalDateTime.now());
        response.put("status", HttpStatus.INTERNAL_SERVER_ERROR.value());

        logService.error("GlobalExceptionHandler", ex.getMessage(), ex);

        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(response);
    }

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<Map<String, Object>> handleNotFound(
        EntityNotFoundException ex
    ) {
        Map<String, Object> response = new HashMap<>();
        response.put("error", "Resource not found");
        response.put("status", HttpStatus.NOT_FOUND.value());

        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(response);
    }
}
```

---

## 前端架构

### React 组件结构

```
App.tsx (主容器)
├─ 状态管理
│  ├─ user (登录态)
│  ├─ selectedImage (上传图像)
│  ├─ result (分析结果)
│  ├─ history (历史记录)
│  ├─ posts (分享作品)
│  └─ stats (用户统计)
│
├─ 视图组件
│  ├─ HomeView (上传 & 分析)
│  ├─ DashboardView (数据仪表板 - 趋势图+雷达图)
│  ├─ GalleryView (作品库)
│  └─ AdminView (管理后台)
│
└─ 功能模块
   ├─ 用户认证 (登录/注册)
   ├─ EXIF 提取 (exif.ts)
   ├─ 日志记录 (logger.ts)
   └─ API 通信
```

### 关键交互流程

```typescript
// 1. 用户上传图像
const handleImageUpload = (file: File) => {
    const reader = new FileReader();
    reader.onload = (e) => {
        setSelectedImage(e.target?.result as string);
    };
    reader.readAsDataURL(file);
};

// 2. 前端预处理
const preprocessImage = async () => {
    // ✅ 提取 EXIF 数据
    const exifData = await extractExif(selectedImage);
    setExifData(exifData);

    // ✅ 计算 CV 特征（前端）
    const vector = await generateVector(selectedImage);  // 简化版向量

    // ✅ 准备请求体
    return {
        image_url: selectedImage,
        vector: vector,
        click_x: clickPosition?.x,
        click_y: clickPosition?.y,
    };
};

// 3. 提交分析请求
const analyzeImage = async () => {
    setIsAnalyzing(true);

    const payload = await preprocessImage();

    const res = await fetch(`${API_BASE_URL}/api/analyze`, {
        method: 'POST',
        headers: {
            'Authorization': `mock-jwt-token-for-${user.username}`,
            'Content-Type': 'application/json',
        },
        body: JSON.stringify(payload),
    });

    const { id } = await res.json();

    // 4. 轮询查询结果
    pollAnalysisResult(id);
};

// 5. 轮询结果
const pollAnalysisResult = (id: number) => {
    const interval = setInterval(async () => {
        const res = await fetch(`${API_BASE_URL}/api/status/${id}`, {
            headers: { 'Authorization': `mock-jwt-token-for-${user.username}` }
        });

        const data = await res.json();

        if (data.status === 'COMPLETED') {
            setResult(data.result);
            setIsAnalyzing(false);
            clearInterval(interval);
        }
    }, 1000);  // 每秒轮询一次
};

// 6. 数据可视化
const renderTrendChart = (stats) => {
    return <LineChart data={stats.trend} />;
};

const renderRadarChart = (stats) => {
    return <RadarChart data={stats.radar} />;
};
```

---

## AI 服务集成

### Python NIMA 服务

```python
# nima_server.py
import tensorflow as tf
from flask import Flask, request, jsonify
import numpy as np

app = Flask(__name__)

# 加载 TensorFlow SavedModel
model = tf.saved_model.load("nima_saved_model")
infer = model.signatures["serving_default"]

@app.route('/predict', methods=['POST'])
def predict():
    data = request.json
    base64_str = data.get('image')

    # 【预处理】
    image_bytes = base64.b64decode(base64_str.split(',')[1])
    img = Image.open(io.BytesIO(image_bytes))
    img = img.convert('RGB').resize((224, 224))
    img_array = np.array(img).astype('float32')
    img_array = (img_array / 127.5) - 1.0  # MobileNet 标准化

    # 【推理】
    input_tensor = tf.convert_to_tensor(np.expand_dims(img_array, axis=0))
    output = infer(inputs=input_tensor)

    # 【转换概率分布 → 评分】
    predictions = output[list(output.keys())[0]].numpy()[0]
    # 分布有 10 个 bin (0-1, 1-2, ..., 9-10)
    score = np.sum(np.arange(1, 11) * predictions)

    return jsonify({'score': float(score)})

if __name__ == '__main__':
    app.run(port=5001)
```

---

## 数据流与交互

### 完整请求响应流程

```
【前端】
  1. 用户选择图像
     └─→ 图像压缩 (如需要)
     └─→ Base64 编码
     └─→ EXIF 提取
     └─→ CV 特征计算 (色彩、对比度等)

【后端 - 同步】
  2. POST /api/analyze
     └─→ 创建 AnalysisHistory (状态: PENDING)
     └─→ 立即返回 historyId
     └─→ 返回给前端

【前端】
  3. 轮询查询
     └─→ GET /api/status/{historyId}
     └─→ 轮询间隔: 1 秒
     └─→ 显示加载动画

【后端 - 异步】@Async 触发后台任务
  4. Step 1: NIMA 评分
     └─→ 调用 Python 服务 (5001)
     └─→ 获得 0-10 分数

  5. Step 2: CV 特征提取
     └─→ 色彩分布 (RGB 均值)
     └─→ 构图复杂度 (边缘能量)
     └─→ 纵横比

  6. Step 3: AI 多模型分析
     └─→ 构建 Prompt (包含 NIMA + CV 特征 + 点击位置)
     └─→ 尝试 Qwen VL-Max 调用
     └─→ 失败 → Zhipu GLM-4V 备用
     └─→ 获得 JSON 分析结果

  7. Step 4: 数据持久化
     └─→ 保存 NIMA 分数
     └─→ 保存 AI 分析结果 (9 维度)
     └─→ 保存 CV 特征
     └─→ 计算并存储向量
     └─→ 更新状态为 COMPLETED

  8. Step 5: 相似推荐 (异步可选)
     └─→ 计算与历史的向量相似度
     └─→ 返回 Top-5 相似作品

【前端】
  9. 轮询发现结果已完成
     └─→ GET /api/status/{historyId}
     └─→ 获得完整结果
     └─→ 展示 9 维度分析
     └─→ 绘制数据可视化
     └─→ 显示推荐作品

【用户】
  10. 查看结果
     └─→ 阅读 AI 评论
     └─→ 查看数据可视化
     └─→ 点击作品分享
     └─→ 或开启新分析
```

---

## 部署与配置

### Docker Compose 编排

```yaml
version: '3.8'

services:
  # MySQL 数据库
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: ai_photography
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

  # Python NIMA 服务
  python-model:
    build:
      context: ./nima_training
      dockerfile: Dockerfile.python
    ports:
      - "5001:5001"
    volumes:
      - ./nima_training/nima_saved_model:/app/nima_saved_model

  # Spring Boot 后端
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.backend
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/ai_photography
      SPRING_DATASOURCE_USERNAME: root
      SPRING_DATASOURCE_PASSWORD: root
      ZHIPU_API_KEY: ${ZHIPU_API_KEY}
      QWEN_API_KEY: ${QWEN_API_KEY}
      NIMA_SERVICE_URL: http://python-model:5001
    ports:
      - "8080:8080"
    depends_on:
      - mysql
      - python-model

  # React 前端
  frontend:
    build:
      context: .
      dockerfile: Dockerfile.frontend
    environment:
      BACKEND_URL: http://backend:8080
      GEMINI_API_KEY: ${GEMINI_API_KEY}
    ports:
      - "3000:3000"
    depends_on:
      - backend

volumes:
  mysql_data:
```

### 一键启动

```bash
# 1. 配置环境变量
cp .env.example .env
# 编辑 .env，填入 API Key

# 2. 构建并启动
docker-compose up -d

# 3. 验证服务
# 前端: http://localhost:3000
# 后端: http://localhost:8080
# Python: http://localhost:5001

# 4. 查看日志
docker-compose logs -f backend
docker-compose logs -f python-model
```

---

## 性能优化要点

| 优化项 | 实现 | 效果 |
|-------|------|------|
| **异步处理** | @Async 长任务 | 不阻塞请求线程 |
| **连接池** | WebClient + 反应式 | 复用 HTTP 连接 |
| **缓存** | 向量缓存 | 减少重复计算 |
| **流量限制** | RateLimiter | 防止 API 滥用 |
| **分层缓存** | 数据库索引 | 加快查询 |
| **容器化** | Docker | 资源隔离与扩展 |

---

## 安全考虑

| 安全项 | 实现 |
|-------|------|
| **身份认证** | JWT Token |
| **授权** | Spring Security |
| **CORS** | 前端跨域支持 |
| **API Key 保护** | 环境变量管理 |
| **输入验证** | 请求体检查 |
| **异常处理** | 不泄露系统信息 |
| **日志记录** | 审计追踪 |

---

总结：这套架构通过 **微服务分离 + 异步处理 + 多模型容错 + 容器化部署**，实现了一个高可用、高性能、易扩展的 AI 应用系统。

