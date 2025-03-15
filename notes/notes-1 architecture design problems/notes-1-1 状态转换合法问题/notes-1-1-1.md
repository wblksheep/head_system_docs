用 java 实现
--- R1
以下是使用 Java 和 Spring Boot 实现喷头状态转换合法性保障的完整方案，包含 **状态机设计**、**乐观锁控制**、**审计日志** 和 **监控指标**。

---

### **1. 环境准备**

#### **Maven 依赖**

```xml
<!-- Spring Boot Starter -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.statemachine</groupId>
    <artifactId>spring-statemachine-core</artifactId>
    <version>3.2.0</version>
</dependency>
<!-- Prometheus 监控 -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
<!-- H2 数据库（示例用） -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

---

### **2. 状态机与实体定义**

#### **喷头状态枚举**

```java
public enum SprayHeadStatus {
    INVENTORY,      // 库存（状态0）
    IN_USE,         // 使用中（状态1）
    UNDER_REPAIR    // 维修中（状态2）
}
```

#### **喷头实体（含乐观锁）**

```java
@Entity
public class SprayHead {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Enumerated(EnumType.STRING)
    private SprayHeadStatus status = SprayHeadStatus.INVENTORY;

    @Version
    private Long version;  // 乐观锁版本号

    // Getters and Setters
}
```

#### **状态机配置**

```java
@Configuration
@EnableStateMachine
public class StateMachineConfig extends EnumStateMachineConfigurerAdapter<SprayHeadStatus, SprayHeadEvent> {

    @Override
    public void configure(StateMachineStateConfigurer<SprayHeadStatus, SprayHeadEvent> states) throws Exception {
        states.withStates()
                .initial(SprayHeadStatus.INVENTORY)
                .states(EnumSet.allOf(SprayHeadStatus.class));
    }

    @Override
    public void configure(StateMachineTransitionConfigurer<SprayHeadStatus, SprayHeadEvent> transitions) throws Exception {
        transitions
                .withExternal()
                    .source(SprayHeadStatus.INVENTORY).target(SprayHeadStatus.IN_USE)
                    .event(SprayHeadEvent.ALLOCATE)
                .and()
                .withExternal()
                    .source(SprayHeadStatus.IN_USE).target(SprayHeadStatus.UNDER_REPAIR)
                    .event(SprayHeadEvent.START_MAINTENANCE)
                .and()
                .withExternal()
                    .source(SprayHeadStatus.UNDER_REPAIR).target(SprayHeadStatus.IN_USE)
                    .event(SprayHeadEvent.COMPLETE_MAINTENANCE)
                .and()
                .withExternal()
                    .source(SprayHeadStatus.UNDER_REPAIR).target(SprayHeadStatus.INVENTORY)
                    .event(SprayHeadEvent.SCRAP);
    }
}
```

---

### **3. 服务层与事务控制**

#### **状态转换服务**

```java
@Service
@Transactional
public class SprayHeadService {

    @Autowired
    private SprayHeadRepository sprayHeadRepository;

    @Autowired
    private StateMachine<SprayHeadStatus, SprayHeadEvent> stateMachine;

    @Autowired
    private MeterRegistry meterRegistry;

    public void changeStatus(Long sprayHeadId, SprayHeadEvent event, String operator) {
        SprayHead sprayHead = sprayHeadRepository.findById(sprayHeadId)
                .orElseThrow(() -> new ResourceNotFoundException("喷头不存在"));

        // 校验状态转换合法性
        if (!stateMachine.transitionExits(event, sprayHead.getStatus())) {
            throw new IllegalStateException("非法状态转换");
        }

        // 记录审计日志
        SprayHeadStatus oldStatus = sprayHead.getStatus();
        stateMachine.sendEvent(event);
        sprayHead.setStatus(stateMachine.getState().getId());
        sprayHeadRepository.save(sprayHead);

        logAudit(sprayHeadId, oldStatus, sprayHead.getStatus(), operator);
        meterRegistry.counter("sprayhead.status.change",
                "from", oldStatus.name(),
                "to", sprayHead.getStatus().name()).increment();
    }

    private void logAudit(Long sprayHeadId, SprayHeadStatus oldStatus,
                         SprayHeadStatus newStatus, String operator) {
        // 实现审计日志存储（如写入数据库或发送到消息队列）
    }
}
```

#### **乐观锁异常处理**

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OptimisticLockingFailureException.class)
    public ResponseEntity<String> handleOptimisticLockingFailure(OptimisticLockingFailureException ex) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
                .body("并发冲突，请重试");
    }

    @ExceptionHandler(IllegalStateException.class)
    public ResponseEntity<String> handleIllegalState(IllegalStateException ex) {
        return ResponseEntity.badRequest().body(ex.getMessage());
    }
}
```

---

### **4. REST API 层**

#### **维修操作接口**

```java
@RestController
@RequestMapping("/api/spray-heads")
public class SprayHeadController {

    @Autowired
    private SprayHeadService sprayHeadService;

    @PostMapping("/{id}/maintenance")
    public ResponseEntity<?> startMaintenance(
            @PathVariable Long id,
            @RequestParam String operator) {

        sprayHeadService.changeStatus(id, SprayHeadEvent.START_MAINTENANCE, operator);
        return ResponseEntity.ok().build();
    }
}
```

---

### **5. 监控配置**

#### **Prometheus 端点**

```java
@Configuration
public class MetricsConfig {

    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config().commonTags("application", "spray-head-service");
    }
}
```

在 `application.properties` 中启用 Prometheus：

```properties
management.endpoints.web.exposure.include=prometheus,health,info
management.metrics.export.prometheus.enabled=true
```

---

### **6. 测试用例**

#### **状态转换测试**

```java
@SpringBootTest
public class SprayHeadServiceTest {

    @Autowired
    private SprayHeadService service;

    @Autowired
    private SprayHeadRepository repository;

    @Test
    public void testValidStatusTransition() {
        SprayHead head = new SprayHead();
        head.setStatus(SprayHeadStatus.IN_USE);
        head = repository.save(head);

        service.changeStatus(head.getId(), SprayHeadEvent.START_MAINTENANCE, "operator");

        SprayHead updated = repository.findById(head.getId()).get();
        assertEquals(SprayHeadStatus.UNDER_REPAIR, updated.getStatus());
    }

    @Test
    public void testInvalidTransition() {
        SprayHead head = new SprayHead();
        head.setStatus(SprayHeadStatus.INVENTORY);
        head = repository.save(head);

        assertThrows(IllegalStateException.class, () -> {
            service.changeStatus(head.getId(), SprayHeadEvent.START_MAINTENANCE, "operator");
        });
    }
}
```

---

### **关键机制说明**

1. **状态合法性校验**

   - 通过 Spring State Machine 明确定义合法转换路径。
   - 在 `changeStatus` 方法中校验当前状态是否允许触发事件。

2. **乐观锁控制**

   - 实体类中的 `@Version` 字段自动实现乐观锁。
   - 捕获 `OptimisticLockingFailureException` 返回 HTTP 409 冲突。

3. **监控与审计**

   - 通过 Micrometer 统计状态变更次数。
   - 自定义审计日志记录每次状态变更的详细信息。

4. **事务边界**
   - `@Transactional` 注解保证数据库操作的原子性。
   - 状态变更与日志记录在同一事务中完成。

---

### **运行验证**

1. 启动应用后访问 Prometheus 指标端点：
   ```bash
   curl http://localhost:8080/actuator/prometheus
   ```
2. 调用维修接口触发状态变更：
   ```bash
   curl -X POST "http://localhost:8080/api/spray-heads/1/maintenance?operator=tech001"
   ```

该方案通过 **状态机 + 乐观锁 + 监控** 三重机制，确保喷头状态转换的合法性与数据一致性。
