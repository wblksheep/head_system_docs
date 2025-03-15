那么如何保证状态转换是合法的呢？
--- R1
为了保证喷头状态转换的合法性，需要从 **状态机设计**、**数据一致性**、**并发控制** 和 **业务规则校验** 四个维度进行系统化设计。以下是具体方案及实现示例：

---

### **一、状态机建模：明确合法转换路径**

#### **1. 定义状态转换规则**

| 当前状态    | 允许转换的目标状态 | 触发条件（操作） |
| ----------- | ------------------ | ---------------- |
| 0（库存）   | → 1（使用中）      | 领用 API         |
| 1（使用中） | → 2（维修中）      | 维修 API         |
| 2（维修中） | → 1（使用中）      | 维修完成 API     |
| 2（维修中） | → 0（库存）        | 报废 API         |

#### **2. 状态机实现（Python 伪代码）**

```python
from transitions import Machine

class SprayHead:
    states = ['0_库存', '1_使用中', '2_维修中']

    transitions = [
        {'trigger': 'allocate', 'source': '0_库存', 'dest': '1_使用中'},
        {'trigger': 'maintain', 'source': '1_使用中', 'dest': '2_维修中'},
        {'trigger': 'complete_maintenance', 'source': '2_维修中', 'dest': '1_使用中'},
        {'trigger': 'scrap', 'source': '2_维修中', 'dest': '0_库存'}
    ]

    def __init__(self):
        self.machine = Machine(model=self, states=SprayHead.states, transitions=SprayHead.transitions, initial='0_库存')
```

---

### **二、数据库层：事务与锁机制**

#### **1. 乐观锁实现（基于版本号）**

```sql
-- 喷头表结构
CREATE TABLE spray_head (
    id INT PRIMARY KEY,
    status INT NOT NULL,
    version INT NOT NULL DEFAULT 0
);

-- 维修操作SQL（原子性更新）
UPDATE spray_head
SET status = 2, version = version + 1
WHERE id = ? AND status = 1 AND version = ?;
```

#### **2. 事务脚本（Python + SQLAlchemy）**

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine('sqlite:///spray.db')
Session = sessionmaker(bind=engine)

def start_maintenance(spray_id, current_version):
    with Session() as session:
        # 悲观锁：SELECT ... FOR UPDATE
        spray = session.query(SprayHead).filter(
            SprayHead.id == spray_id
        ).with_for_update().first()

        if not spray:
            raise ValueError("喷头不存在")
        if spray.status != 1:
            raise IllegalStateError("当前状态不允许维修")

        spray.status = 2
        spray.version += 1
        session.commit()
```

---

### **三、API 层：业务规则校验**

#### **1. 维修接口设计（FastAPI 示例）**

```python
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel

router = APIRouter()

class MaintenanceRequest(BaseModel):
    spray_id: int
    operator: str
    reason: str

@router.post("/spray/maintenance")
async def start_maintenance(request: MaintenanceRequest):
    try:
        # 1. 查询当前状态
        spray = db.query_spray(request.spray_id)
        if not spray:
            raise HTTPException(status_code=404, detail="喷头不存在")

        # 2. 状态机校验
        if not spray.can_maintain():  # 内部调用状态机逻辑
            raise HTTPException(status_code=409, detail="状态冲突")

        # 3. 执行状态变更
        db.update_spray_status(spray.id, new_status=2, version=spray.version)

        # 4. 记录审计日志
        log_audit(
            spray_id=spray.id,
            old_status=1,
            new_status=2,
            operator=request.operator,
            reason=request.reason
        )

        return {"message": "维修状态已更新"}

    except ConcurrentModificationError:
        raise HTTPException(status_code=409, detail="并发冲突，请重试")
```

---

### **四、防并发冲突方案对比**

| **方案**     | **适用场景**                | **实现复杂度** | **性能影响** |
| ------------ | --------------------------- | -------------- | ------------ |
| **乐观锁**   | 低冲突率场景（如 C 端应用） | 低             | 低           |
| **悲观锁**   | 高冲突率场景（如库存扣减）  | 中             | 中           |
| **分布式锁** | 分布式系统跨服务状态管理    | 高             | 高           |

---

### **五、审计与追溯设计**

#### **1. 状态变更事件表**

```sql
CREATE TABLE status_audit (
    id INT PRIMARY KEY AUTO_INCREMENT,
    spray_id INT NOT NULL,
    old_status INT NOT NULL,
    new_status INT NOT NULL,
    operator VARCHAR(50),
    reason TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### **2. 事件溯源查询**

```python
def get_status_history(spray_id: int):
    return db.query(
        "SELECT old_status, new_status, operator, created_at "
        "FROM status_audit WHERE spray_id = ? ORDER BY created_at DESC",
        (spray_id,)
    )
```

---

### **六、防御性编程：异常处理策略**

1. **自定义异常类型**

```python
class IllegalStateError(Exception):
    """非法状态转换异常"""

class ConcurrentModificationError(Exception):
    """并发修改异常"""
```

2. **全局异常拦截（FastAPI）**

```python
@app.exception_handler(IllegalStateError)
async def illegal_state_handler(request, exc):
    return JSONResponse(
        status_code=400,
        content={"detail": f"非法操作: {str(exc)}"}
    )
```

---

### **七、全链路监控**

#### **1. Prometheus 指标**

```python
from prometheus_client import Counter

STATUS_CHANGE_COUNTER = Counter(
    'spray_status_changes_total',
    '喷头状态变更总次数',
    ['from_status', 'to_status']
)

# 在状态变更成功后埋点
STATUS_CHANGE_COUNTER.labels(from_status=1, to_status=2).inc()
```

#### **2. 日志格式规范**

```python
import logging
import json_log_formatter

formatter = json_log_formatter.JSONFormatter()

logger = logging.getLogger('spray')
logger.setLevel(logging.INFO)
handler = logging.StreamHandler()
handler.setFormatter(formatter)
logger.addHandler(handler)

# 示例日志
logger.info("状态变更成功", extra={
    'spray_id': 123,
    'old_status': 1,
    'new_status': 2
})
```

---

### **总结**

通过 **状态机模型 + 数据库锁 + 审计日志 + 监控告警** 的组合方案，可系统化解决喷头状态转换的合法性问题。关键要点：

1. **状态机驱动**：所有变更必须通过预定义规则。
2. **原子性保证**：通过乐观锁/悲观锁避免并发冲突。
3. **可追溯性**：完整记录每次变更的上下文信息。
4. **实时监控**：快速发现异常转换行为。
