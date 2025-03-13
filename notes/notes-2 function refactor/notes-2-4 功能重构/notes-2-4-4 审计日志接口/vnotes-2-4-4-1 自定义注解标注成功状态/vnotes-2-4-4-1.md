自定义注解如何标注成功状态？

#### 3. **状态判断逻辑**

```java
status = result instanceof SuccessResult ? "SUCCESS" : "FAILED"
```

- 假设业务方法统一返回 `SuccessResult` 或 `FailureResult`
- 实际项目中建议使用自定义注解标注成功状态

## --- R1

### 一、自定义注解设计方案

#### 1. **定义状态判断注解**

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface AuditStatus {
    /**
     * SpEL表达式，用于判断操作是否成功
     * 默认检查返回对象的success字段
     */
    String condition() default "#result.success";
}
```

#### 2. **注解使用示例**

```java
@Service
public class DeviceProcessor {

    // 案例1：检查返回对象的success字段
    @AuditStatus
    public CommonResult allocate(DeviceDTO dto) {
        // 返回对象需包含success字段
    }

    // 案例2：自定义成功判断条件
    @AuditStatus(condition = "#result.code == 200")
    public ResponseDTO maintain(MaintainDTO dto) {
        // 根据HTTP状态码判断
    }

    // 案例3：结合方法参数判断
    @AuditStatus(condition = "#result.valid && #dto.priority == 'HIGH'")
    public OperationResult emergency(UrgentDTO dto) {
        // 复杂判断逻辑
    }
}
```

---

### 二、切面逻辑改造

#### 1. **增强后的切面实现**

```java
@Aspect
@Component
public class EnhancedAuditAspect {
    @Autowired
    private AuditLogRepository repository;

    @Autowired
    private ExpressionParser spelParser; // SpEL解析器

    @AfterReturning(
        pointcut = "@annotation(auditStatus)",
        returning = "result"
    )
    public void logSuccess(JoinPoint joinPoint, Object result, AuditStatus auditStatus) {
        EvaluationContext context = createContext(joinPoint, result);
        boolean success = evaluateCondition(auditStatus.condition(), context);

        saveAuditLog(joinPoint, success ? "SUCCESS" : "FAILED");
    }

    @AfterThrowing(pointcut = "@annotation(auditStatus)", throwing = "ex")
    public void logFailure(JoinPoint joinPoint, AuditStatus auditStatus, Exception ex) {
        saveAuditLog(joinPoint, "FAILED");
    }

    private EvaluationContext createContext(JoinPoint jp, Object result) {
        StandardEvaluationContext context = new StandardEvaluationContext();
        context.setVariable("result", result);
        context.setVariable("args", jp.getArgs());
        return context;
    }

    private boolean evaluateCondition(String condition, EvaluationContext context) {
        return spelParser.parseExpression(condition)
               .getValue(context, Boolean.class);
    }

    private void saveAuditLog(JoinPoint jp, String status) {
        // 日志保存逻辑
    }
}
```

---

### 三、关键技术解析

#### 1. **SpEL 上下文变量**

| 变量名    | 说明                     |
| --------- | ------------------------ |
| `#result` | 方法返回值对象           |
| `#args`   | 方法参数数组             |
| `#root`   | 根对象（方法执行上下文） |

#### 2. **条件表达式示例**

| 场景         | 表达式示例                                 |
| ------------ | ------------------------------------------ |
| 检查返回码   | `#result.code == 200`                      |
| 组合参数判断 | `#result.success && #args[0].priority > 5` |
| 集合结果校验 | `#result.items.size() > 0`                 |

---

### 四、配置与优化建议

#### 1. **SpEL 预编译优化**

```java
@Bean
public ExpressionParser spelParser() {
    return new SpelExpressionParser(
        new SpelParserConfiguration(
            SpelCompilerMode.IMMEDIATE, // 启用编译模式
            getClass().getClassLoader()
        )
    );
}
```

#### 2. **注解继承支持**

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Inherited
public @interface AuditStatus {
    // 允许类级别注解继承
}
```

#### 3. **缓存表达式解析**

```java
private final Map<String, Expression> expressionCache = new ConcurrentHashMap<>();

private boolean evaluateCondition(String condition, EvaluationContext context) {
    Expression expr = expressionCache.computeIfAbsent(condition,
        key -> spelParser.parseExpression(key));
    return expr.getValue(context, Boolean.class);
}
```

---

### 五、与传统方案的对比

| **评估维度** | **自定义注解方案**         | **原 instanceof 方案** |
| ------------ | -------------------------- | ---------------------- |
| **灵活性**   | 支持任意判断逻辑           | 仅支持固定类型判断     |
| **可维护性** | 业务规则集中管理           | 规则分散在切面代码中   |
| **学习成本** | 需了解 SpEL 语法           | 无需额外知识           |
| **性能**     | 经预编译优化后接近原生代码 | 直接类型检查性能更优   |
| **可测试性** | 可单元测试条件表达式       | 需构造特定返回值对象   |

---

### 六、生产环境实施步骤

1. **定义审计注解规范**

   ```markdown
   - 所有需要审计的方法必须添加 `@AuditStatus`
   - 默认检查 `#result.success` 字段
   - 复杂场景使用 SpEL 表达式声明规则
   ```

2. **渐进式迁移策略**

   ```java
   // 阶段1：兼容新旧方案
   @Around("@annotation(auditStatus) || execution(* *Processor.*(..))")
   public Object hybridAudit(ProceedingJoinPoint pjp) {
       // 先执行原有逻辑
       // 逐步迁移到注解方案
   }
   ```

3. **监控与告警配置**
   ```yaml
   # 监控指标示例
   metrics:
     audit:
       success_rate:
         tags: [service, method]
       condition_eval_time:
         description: "条件表达式计算耗时"
   ```

---

通过这种设计，系统实现了 **声明式审计状态管理**，业务方可以通过简单的注解配置灵活定义成功规则，同时保持审计逻辑与业务代码的解耦。
