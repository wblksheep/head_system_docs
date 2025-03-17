架构目录图应该是怎样的？

--- R1

以下是基于 DDD 和 Clean Architecture 的 **Spring Boot 项目目录结构**设计，充分体现分层架构并支持未来扩展：

---

### 架构目录图（Package Structure）

```
src/main/java
├── com.example.sprinkler                    # 项目根包
│   │
│   ├── **application**                      # 应用层（协调用例）
│   │   ├── service                          # 应用服务（用例实现）
│   │   │   ├── AllocateService.java         # 示例：喷头领用服务
│   │   │   └── MaintainService.java         # 示例：维修工单服务
│   │   ├── dto                              # 数据传输对象（DTO）
│   │   │   ├── request                      # 请求DTO（如SprinklerAllocateRequest.java）
│   │   │   └── response                     # 响应DTO（如ApiResponse.java）
│   │   └── event                            # 应用层事件（如SprinklerStatusChangedEvent.java）
│   │
│   ├── **domain**                           # 领域层（核心业务逻辑）
│   │   ├── model                            # 领域模型
│   │   │   ├── aggregate                    # 聚合根（如Sprinkler.java）
│   │   │   ├── vo                           # 值对象（如MaintenanceRecord.java）
│   │   │   └── enums                        # 枚举（如SprinklerStatus.java）
│   │   ├── service                          # 领域服务（如MaintenanceCheckService.java）
│   │   ├── repository                       # 仓储接口（如SprinklerRepository.java）
│   │   └── exception                        # 领域异常（如BusinessRuleException.java）
│   │
│   ├── **infrastructure**                   # 基础设施层（技术实现）
│   │   ├── persistence                      # 持久化实现
│   │   │   ├── jpa                          # JPA实现（如JpaSprinklerRepository.java）
│   │   │   └── es                           # Elasticsearch实现（如EsSprinklerRepository.java）
│   │   ├── file                             # 文件处理
│   │   │   ├── parser                       # 文件解析器（ExcelParser.java、TxtParser.java）
│   │   │   └── storage                      # 文件存储（MinioStorageService.java）
│   │   ├── mq                               # 消息队列（RabbitMQ/Kafka生产者消费者）
│   │   ├── audit                            # 审计日志（AOP切面+数据库存储）
│   │   └── config                           # 技术配置（如ElasticsearchConfig.java）
│   │
│   ├── **interfaces**                       # 接口层（Web入口）
│   │   ├── rest                             # REST API
│   │   │   ├── controller                   # 控制器（SprinklerController.java）
│   │   │   └── advice                       # 全局异常处理（GlobalExceptionHandler.java）
│   │   └── websocket                        # WebSocket实时日志推送（LogWebSocket.java）
│   │
│   └── **common**                           # 通用工具包
│       ├── util                             # 工具类（ExcelUtils.java、JsonConverter.java）
│       ├── constants                        # 全局常量（如ApiConstants.java）
│       └── aspect                           # 通用切面（如LoggingAspect.java）
│
└── resources
    ├── config                               # 配置文件
    │   ├── application.yml                  # 主配置
    │   └── es-mappings                     # Elasticsearch索引映射
    └── templates                           # 文件模板（如Excel导入模板）
```

---

### 关键目录说明

#### 1. **application 层**

- **职责**：编排领域服务，处理事务、权限、DTO 转换等跨领域协调工作。
- **扩展性设计**：  
  通过 `@ConditionalOnProperty` 实现功能模块开关，例如：
  ```java
  @Configuration
  @ConditionalOnProperty(name = "feature.audit.enabled", havingValue = "true")
  public class AuditConfig { ... }
  ```

#### 2. **domain 层**

- **核心原则**：保持纯净，不依赖任何外部框架（JPA/ES 注解通过适配器实现）。
- **状态机示例**：  
  使用 `Sprinkler` 聚合根内部方法实现状态转换：
  ```java
  public class Sprinkler {
      public void allocate() {
          if (this.status != SprinklerStatus.STOCK) {
              throw new IllegalStateException("仅库存中喷头可领用");
          }
          this.status = SprinklerStatus.IN_USE;
          this.domainEvents.add(new SprinklerAllocatedEvent(this.id));
      }
  }
  ```

#### 3. **infrastructure 层**

- **插件化设计**：  
  数据库和 Elasticsearch 通过不同 Repository 实现自由切换：

  ```java
  // 领域层接口
  public interface SprinklerRepository {
      Sprinkler findById(SprinklerId id);
  }

  // JPA实现
  @Repository
  public class JpaSprinklerRepository implements SprinklerRepository {
      @Override
      public Sprinkler findById(SprinklerId id) { ... }
  }

  // ES实现（可选）
  @Repository
  @ConditionalOnProperty(name = "persistence.mode", havingValue = "es")
  public class EsSprinklerRepository implements SprinklerRepository { ... }
  ```

#### 4. **interfaces 层**

- **统一响应格式**：  
  使用 `@RestControllerAdvice` 全局封装响应：
  ```java
  @RestControllerAdvice
  public class ApiResponseAdvice {
      @ResponseBody
      public ApiResponse<Object> handleSuccess() {
          return ApiResponse.success();
      }
  }
  ```

#### 5. **扩展性预留**

- **工单转换**：在 `infrastructure/file` 下添加 `converter` 包实现不同格式转换。
- **实时日志**：通过 `infrastructure/mq` 发送日志事件，`interfaces/websocket` 推送至前端。

---

### 技术栈对应关系

| 业务需求           | 技术实现                             | 对应目录位置                  |
| ------------------ | ------------------------------------ | ----------------------------- |
| Excel/TXT 解析     | Apache POI + 策略模式                | infrastructure/file/parser    |
| 状态转换           | 聚合根内部方法 + Spring StateMachine | domain/model/aggregate        |
| 审计日志           | AOP 切面 + 数据库/ES 存储            | infrastructure/audit          |
| 实时日志显示       | WebSocket + STOMP                    | interfaces/websocket          |
| Elasticsearch 扩展 | Spring Data ES                       | infrastructure/persistence/es |

---

该目录结构清晰体现 **DDD 分层架构**，同时通过模块化设计支持未来扩展。数据量较小时可先用 JPA，随着数据增长通过 `infrastructure/persistence/es` 平滑集成 Elasticsearch。
