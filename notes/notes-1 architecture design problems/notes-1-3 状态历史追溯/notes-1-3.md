状态历史追溯有哪些手段，一定要在数据库中增加审计日志表吗？

```questions
   - 如何记录状态变更的完整历史（时间、操作人、原因）？
   - 是否需要通过 **事件溯源（Event Sourcing）** 或 **审计日志** 实现追溯？
```

--- R1
状态历史追溯的实现手段不强制依赖数据库审计日志表，但需要根据具体场景选择合适的技术方案。以下是多种实现方式及其适用场景：

---

### **一、核心追溯手段对比**

| **方案**                      | **优点**                     | **缺点**                   | **适用场景**                  |
| ----------------------------- | ---------------------------- | -------------------------- | ----------------------------- |
| **数据库审计表**              | 实时性强，查询方便           | 高并发下性能压力大         | 中小型系统，需实时审计        |
| **事件溯源 (Event Sourcing)** | 完整记录事件流，支持状态重建 | 架构复杂，存储成本高       | 复杂领域模型，需审计+业务逻辑 |
| **数据库触发器**              | 对业务代码无侵入             | 维护困难，跨数据库兼容性差 | 遗留系统改造                  |
| **日志文件**                  | 实现简单，存储成本低         | 查询和分析效率低           | 临时性调试，非关键审计        |
| **CDC (Change Data Capture)** | 实时同步，对业务无侵入       | 依赖数据库特性，架构复杂   | 大数据分析，跨系统同步        |
| **区块链存证**                | 数据不可篡改，可验证         | 性能低，成本高             | 金融、法律等高合规场景        |

---

### **二、主流方案详解**

#### **1. 数据库审计日志表**

**实现方式**：在业务库中创建专用审计表，记录关键字段变更。  
**示例表结构**：

```sql
CREATE TABLE status_audit (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    entity_id BIGINT NOT NULL,       -- 喷头ID
    old_status VARCHAR(50) NOT NULL,
    new_status VARCHAR(50) NOT NULL,
    operator VARCHAR(50) NOT NULL,  -- 操作人
    operation_time DATETIME NOT NULL,
    reason TEXT                      -- 变更原因
);
```

**代码示例（Spring Data JPA）**：

```java
@Entity
@EntityListeners(AuditListener.class)  // 监听实体变更
public class StatusAudit {
    @Id @GeneratedValue private Long id;
    private Long entityId;
    private String oldStatus;
    private String newStatus;
    private String operator;
    private LocalDateTime operationTime;
    private String reason;
}

// 审计监听器
public class AuditListener {
    @PreUpdate
    public void beforeUpdate(SprayHead entity) {
        StatusAudit audit = new StatusAudit();
        audit.setEntityId(entity.getId());
        audit.setOldStatus(entity.getStatus());
        audit.setNewStatus(entity.getStatus());
        auditRepository.save(audit);
    }
}
```

---

#### **2. 事件溯源 (Event Sourcing)**

**核心思想**：将状态变更记录为不可变事件流，通过回放事件重建状态。  
**实现方式**：

- **事件存储**：使用专用事件存储（如 [EventStoreDB](https://eventstore.com/)）或消息队列（如 Kafka）。
- **状态重建**：通过 `CQRS` 模式分离读写，从事件流生成当前状态。

**代码示例（Kafka 事件发布）**：

```java
// 状态变更事件
public record StatusChangedEvent(
    String sprayId,
    String oldStatus,
    String newStatus,
    String operator
) {}

// 发布事件
kafkaTemplate.send("status-events", new StatusChangedEvent(...));
```

---

#### **3. 数据库日志/CDC**

**实现方式**：通过解析数据库事务日志（如 MySQL Binlog、PostgreSQL WAL）捕获变更。  
**工具**：

- **[Debezium](https://debezium.io/)**：实时捕获数据库变更，输出到 Kafka。
- **[AWS DMS](https://aws.amazon.com/dms/)**：托管式数据库迁移和同步服务。

**架构图**：

```
MySQL → Debezium → Kafka → Elasticsearch（审计分析）
```

---

#### **4. 应用日志 + ELK**

**实现方式**：在业务代码中打印结构化日志，通过 ELK 栈（Elasticsearch + Logstash + Kibana）集中分析。  
**日志格式**：

```json
{
  "timestamp": "2023-10-05T14:23:01Z",
  "sprayId": "123",
  "oldStatus": "IN_USE",
  "newStatus": "UNDER_REPAIR",
  "operator": "user-456"
}
```

**Kibana 看板**：  
![](https://www.elastic.co/static-res/images/elastic-logo-200.png)

---

### **三、如何选择方案？**

#### **决策树**

1. **是否需要实时查询？**

   - 是 → 数据库审计表
   - 否 → 事件溯源或日志文件

2. **是否需防止数据篡改？**

   - 是 → 区块链存证
   - 否 → 常规方案

3. **系统复杂度容忍度？**

   - 高 → 事件溯源 + CQRS
   - 低 → 数据库触发器

4. **是否已有大数据平台？**
   - 是 → CDC + 数据湖
   - 否 → 应用日志 + ELK

---

### **四、混合方案示例**

**场景**：既要实时查询，又要长期归档。  
**架构**：

```
业务系统 → 数据库审计表（实时查询）
         → Kafka → 数据湖（长期存储+分析）
```

---

### **总结**

- **强制审计需求（如 SOX、GDPR）**：优先选择数据库审计表或区块链。
- **高并发系统**：事件溯源 + 消息队列（如 Kafka）。
- **低成本临时方案**：应用日志 + ELK。
- **无需侵入业务代码**：数据库触发器或 CDC。

无需局限于单一方案，可根据业务阶段和技术栈灵活组合。
