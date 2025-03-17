我需要的是 BS 架构下的架构目录图哈，你这还是 DDD 架构下的架构目录图吧？

--- R1

以下是针对 **B/S 架构**（传统分层架构）的目录结构设计，结合你的具体业务场景，**重点突出文件解析、状态转换、扩展接口**等技术实现模块：

---

### B/S 架构目录图（经典分层）

```
src/main/java
├── com.example.sprinkler
│   │
│   ├── **controller**                      # 控制层（请求入口）
│   │   ├── SprinklerFileController.java     # 文件上传/解析接口
│   │   ├── SprinklerStatusController.java   # 状态变更接口
│   │   └── LogStreamController.java         # 实时日志推送接口
│   │
│   ├── **service**                         # 业务逻辑层（核心处理）
│   │   ├── impl
│   │   │   ├── FileParseServiceImpl.java   # 文件解析服务（Excel/TXT处理）
│   │   │   ├── StatusChangeServiceImpl.java# 状态机转换服务
│   │   │   └── LogExportServiceImpl.java    # 日志导出服务
│   │   └── converter                       # DTO转换器（如SprinklerConverter.java）
│   │
│   ├── **dao**                             # 数据访问层（数据库操作）
│   │   ├── SprinklerAllocateMapper.java     # 领用记录Mapper（MyBatis）
│   │   ├── SprinklerMaintainMapper.java     # 维修工单Mapper
│   │   └── LogMapper.java                   # 审计日志Mapper
│   │
│   ├── **entity**                          # 数据库实体（对应表结构）
│   │   ├── SprinklerAllocateEntity.java     # 领用记录表
│   │   ├── SprinklerMaintainEntity.java     # 维修工单表
│   │   └── LogEntity.java                   # 日志表
│   │
│   ├── **model**                           # 业务模型（DTO/VO）
│   │   ├── dto
│   │   │   ├── SprinklerAllocateDTO.java    # 领用传输对象
│   │   │   └── SprinklerMaintainDTO.java    # 维修工单对象
│   │   └── vo
│   │       └── ResultVO.java                # 统一响应对象
│   │
│   ├── **utils**                           # 工具类
│   │   ├── ExcelUtils.java                  # Excel解析工具（Apache POI）
│   │   ├── JsonUtils.java                   # JSON序列化工具
│   │   └── StateMachineUtils.java           # 状态机工具
│   │
│   ├── **config**                          # 配置类
│   │   ├── ElasticsearchConfig.java         # ES连接配置
│   │   ├── SwaggerConfig.java               # API文档配置
│   │   └── WebSocketConfig.java             # 实时日志推送配置
│   │
│   ├── **listener**                        # 监听器
│   │   └── LogEventListener.java           # 日志事件监听（如数据库操作日志）
│   │
│   └── **interfaces**                      # 扩展接口预留
│       ├── es                              # Elasticsearch扩展
│       │   ├── EsSprinklerService.java      # ES数据操作接口
│       │   └── impl/EsSprinklerServiceImpl.java
│       └── converter                       # 工单转换模块
│           ├── OrderConverter.java         # 工单格式转换接口
│           └── impl/ExcelOrderConverter.java
│
└── resources
    ├── mapper                              # MyBatis XML文件
    │   ├── SprinklerAllocateMapper.xml
    │   └── SprinklerMaintainMapper.xml
    ├── es-mappings                        # ES索引映射
    │   └── sprinkler-mapping.json
    └── templates                          # Excel模板文件
```

---

### 关键目录说明

#### 1. **文件解析流程**

- **技术实现**：

  - `FileParseServiceImpl` 调用 `ExcelUtils` 和自定义解析器处理 Excel/TXT，**通过策略模式实现多格式支持**：

    ```java
    public interface FileParser {
        List<SprinklerAllocateDTO> parse(InputStream inputStream);
    }

    @Component("excelParser")
    public class ExcelParser implements FileParser { ... }

    @Component("txtParser")
    public class TxtParser implements FileParser { ... }
    ```

#### 2. **状态转换与数据库操作**

- **状态机设计**：  
  使用 `StateMachineUtils` 工具类封装状态转换规则，避免硬编码：

  ```java
  public class StateMachineUtils {
      private static final Map<String, Map<Integer, Integer>> STATE_RULES = new HashMap<>();

      static {
          // 领用：0->1
          STATE_RULES.put("allocate", Map.of(0, 1));
          // 维修：1->2
          STATE_RULES.put("maintain", Map.of(1, 2));
      }

      public static boolean validateTransition(String operation, int currentStatus) {
          return STATE_RULES.getOrDefault(operation, Collections.emptyMap())
                            .containsKey(currentStatus);
      }
  }
  ```

#### 3. **扩展接口设计**

- **Elasticsearch 扩展**：  
  在 `interfaces/es` 中定义接口，通过 `@ConditionalOnClass` 实现 **ES 功能的按需加载**：

  ```java
  @Service
  @ConditionalOnClass(ElasticsearchOperations.class)
  public class EsSprinklerServiceImpl implements EsSprinklerService {
      @Autowired
      private ElasticsearchOperations esOperations;

      public void syncToEs(SprinklerAllocateDTO dto) { ... }
  }
  ```

- **工单转换**：  
  在 `interfaces/converter` 中定义转换接口，支持未来新增格式（如 XML、CSV）：

  ```java
  public interface OrderConverter {
      String convert(SprinklerMaintainDTO maintainDTO);
  }

  @Component("excelOrderConverter")
  public class ExcelOrderConverter implements OrderConverter { ... }
  ```

#### 4. **实时日志推送**

- **技术实现**：  
  通过 `WebSocketConfig` 配置实时推送，前端监听 `/topic/realtime-logs`：

  ```java
  @Controller
  public class LogStreamController {
      @Autowired
      private SimpMessagingTemplate messagingTemplate;

      @Async
      public void pushLog(String logMessage) {
          messagingTemplate.convertAndSend("/topic/realtime-logs", logMessage);
      }
  }
  ```

---

### 扩展性设计

| 需求                 | 实现方式                                | 扩展方法                         |
| -------------------- | --------------------------------------- | -------------------------------- |
| **新增文件格式**     | 在 `FileParser` 接口添加实现类          | 新增 `CSVParser.java`            |
| **新增状态规则**     | 修改 `StateMachineUtils` 状态映射表     | 无代码侵入，只需更新配置         |
| **接入其他搜索引擎** | 在 `interfaces` 下新增模块（如 `solr`） | 实现 `SolrSprinklerService.java` |
| **工单格式转换**     | 实现 `OrderConverter` 接口              | 新增 `XmlOrderConverter.java`    |

---

### 数据流示意图

```
[浏览器] → (HTTP上传文件) → [SprinklerFileController]
    → 调用 → [FileParseServiceImpl] → 解析为DTO
        → 调用 → [StatusChangeServiceImpl] → 状态校验
            → 调用 → [SprinklerAllocateMapper] → 写入数据库
                → 触发 → [LogEventListener] → 记录审计日志
                    → 推送 → [LogStreamController] → WebSocket实时日志
```

---

此目录结构清晰分离 **请求入口（Controller）→ 业务逻辑（Service）→ 数据操作（Dao）**，同时通过 `interfaces` 包预留扩展点，非常适合数据量小但需要快速迭代的 B/S 系统。
