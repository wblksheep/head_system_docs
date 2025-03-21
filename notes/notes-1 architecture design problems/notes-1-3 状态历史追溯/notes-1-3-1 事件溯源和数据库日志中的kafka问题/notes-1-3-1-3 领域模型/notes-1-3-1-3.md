什么是领域模型，为什么 CQRS 需要完整历史？
--- R1

### **领域模型（Domain Model）**

**定义**：领域模型是软件系统中对业务领域概念的抽象表示，它通过对象、属性、关系和规则，精确反映业务逻辑和操作流程。其核心目标是 **将复杂业务需求转化为可维护的代码结构**。

#### **核心要素**

1. **实体（Entity）**

   - 具有唯一标识的对象（如 `订单`、`用户`）。
   - 示例：订单的 `订单号` 是唯一标识，即使订单内容相同，不同订单号代表不同实体。

2. **值对象（Value Object）**

   - 无唯一标识，通过属性值定义的对象（如 `地址`、`金额`）。
   - 示例：地址 `北京市海淀区` 和 `北京市朝阳区` 是两个不同的值对象。

3. **聚合根（Aggregate Root）**

   - 控制一组相关对象访问的根实体（如 `购物车` 管理商品项和优惠）。
   - 规则：外部只能通过聚合根修改内部对象。

4. **领域服务（Domain Service）**

   - 处理跨多个实体的业务逻辑（如 `支付服务` 处理订单和账户的交互）。

5. **领域事件（Domain Event）**
   - 记录业务状态变更的事件（如 `OrderPaidEvent`、`UserRegisteredEvent`）。

#### **示例：电商订单领域模型**

```text
Order（聚合根）
├── OrderId（唯一标识）
├── User（实体）
├── Items（值对象列表）
├── Status（枚举：已创建、已支付、已发货）
└── applyPayment()（领域方法：更新状态并触发 OrderPaidEvent）
```

---

### **为什么 CQRS 需要完整历史？**

CQRS（Command Query Responsibility Segregation）通过分离命令（写）和查询（读）模型优化系统性能，但其完整性和灵活性高度依赖 **完整历史记录**。以下是关键原因：

#### **1. 事件溯源（Event Sourcing）的依赖**

- **原理**：系统状态通过 **按顺序应用历史事件** 重建。
- **示例**：订单状态从 `已创建` → `已支付` → `已发货`，需记录每个事件：
  ```text
  OrderCreatedEvent → OrderPaidEvent → OrderShippedEvent
  ```
- **必要性**：
  - 无完整事件流，无法追溯状态变化路径。
  - 无法支持 **时间旅行调试**（如查看订单在某个时间点的状态）。

#### **2. 业务审计与合规**

- **需求**：金融、医疗等行业需记录所有操作痕迹以满足法规（如 GDPR、SOX）。
- **示例**：银行交易需记录 `谁` 在 `何时` 修改了账户余额，完整事件流提供不可篡改的审计日志。

#### **3. 解决并发冲突**

- **问题**：多个用户同时修改同一实体可能导致数据覆盖。
- **方案**：通过事件版本号（如 `version`）实现乐观锁：
  ```java
  // 更新时校验版本号
  UPDATE orders SET status = 'PAID', version = version + 1
  WHERE id = 123 AND version = 5;
  ```
  - 若版本不匹配，拒绝操作并提示冲突。

#### **4. 支持复杂查询与分析**

- **场景**：生成用户行为报告需分析历史事件（如用户下单频率、支付失败原因）。
- **技术实现**：将事件流同步到数据仓库（如 BigQuery）进行离线分析。

#### **5. 业务逻辑回滚与补偿**

- **需求**：错误操作需回滚到之前状态（如取消误支付的订单）。
- **实现**：反向应用事件（如生成 `OrderCancelledEvent`）恢复状态。

---

### **CQRS + 事件溯源架构示例**

```text
┌───────────────┐       Command       ┌───────────────┐
│   客户端        │ ------------------> │  命令处理器     │
└───────────────┘                      └───────┬───────┘
                                                │
                                                ▼
                                         ┌───────────────┐
                                         │  事件存储      │
                                         │（EventStoreDB）│
                                         └───────┬───────┘
                                                 │
                                                 │ 异步订阅
                                                 ▼
┌───────────────┐       Query        ┌───────────────┐
│   客户端        │ <------------------ │  读模型存储     │
└───────────────┘                    （如Elasticsearch）
```

#### **流程说明**

1. **命令处理**：

   - 客户端发送 `UpdateOrderStatusCommand` 到命令处理器。
   - 校验业务规则后，生成 `OrderStatusUpdatedEvent` 并存入事件存储。

2. **读模型更新**：

   - 读模型订阅事件流，将 `OrderStatusUpdatedEvent` 转换为非规范化结构（如更新 Elasticsearch 中的订单视图）。

3. **历史追溯**：
   - 通过重放事件流到任意时间点，重建订单历史状态。

---

### **总结**

- **领域模型** 是业务逻辑的代码映射，强调实体、规则和关系的精确表达。
- **CQRS 依赖完整历史** 实现审计、并发控制、复杂查询等功能，其核心是通过事件溯源记录所有状态变更。
- **技术选型建议**：
  - 高一致性场景 → 使用 EventStoreDB 或 Kafka（持久化日志）。
  - 高查询需求 → 同步事件到 Elasticsearch 或 ClickHouse。
