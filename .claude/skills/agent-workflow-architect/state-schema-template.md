# State Schema 模板

## WorkflowState - 通用工作流状态

```python
from typing import TypedDict, Optional, List, Dict, Any
from datetime import datetime

class WorkflowState(TypedDict, total=False):
    """
    通用工作流状态定义

    使用 TypedDict 定义状态结构，LangGraph 会自动管理状态传递
    """

    # ========== 输入 ==========
    task_id: str
    """任务 ID"""

    input_data: Dict[str, Any]
    """
    输入数据

    包含：
    - query: 用户查询/任务描述
    - parameters: 任务参数
    - constraints: 约束条件
    """

    # ========== 上下文信息 ==========
    context: Dict[str, Any]
    """
    上下文信息

    包含：
    - history: 历史记录
    - documents: 相关文档
    - entities: 实体信息
    - metadata: 元数据
    """

    plan: Optional[Dict[str, Any]]
    """
    执行计划

    包含：
    - steps: 执行步骤
    - goals: 目标
    - constraints: 约束
    """

    # ========== 生成内容 ==========
    result: Optional[str]
    """生成的结果"""

    # ========== 检查报告 ==========
    validation_report: Dict[str, Any]
    """
    验证报告

    包含：
    - issues: 发现的问题
    - score: 质量评分
    - suggestions: 改进建议
    """

    # ========== 执行状态 ==========
    execution_history: List[str]
    """
    执行历史

    记录已执行的节点名称，用于追踪工作流进度
    """

    current_node: Optional[str]
    """当前正在执行的节点"""

    # ========== Checkpoint 相关 ==========
    checkpoint_id: Optional[str]
    """Checkpoint ID"""

    checkpoint_timestamp: Optional[datetime]
    """Checkpoint 时间戳"""

    # ========== 错误处理 ==========
    error: Optional[str]
    """错误信息（如果有）"""

    retry_count: int
    """重试次数"""

    # ========== 元数据 ==========
    metadata: Dict[str, Any]
    """
    元数据

    可用于存储额外信息，如：
    - model_name: 使用的模型
    - tokens_used: 使用的 Token 数
    - execution_time: 执行时间
    """
```

---

## 状态初始化示例

```python
def create_initial_state(
    task_id: str,
    input_data: Dict[str, Any]
) -> WorkflowState:
    """创建初始状态"""
    return WorkflowState(
        task_id=task_id,
        input_data=input_data,
        context={},
        plan=None,
        result=None,
        validation_report={},
        execution_history=[],
        current_node=None,
        checkpoint_id=None,
        checkpoint_timestamp=None,
        error=None,
        retry_count=0,
        metadata={}
    )
```

---

## 状态更新模式

### 1. 简单字段更新

```python
async def update_node(state: WorkflowState) -> WorkflowState:
    """更新节点示例"""
    # 创建新状态（不修改原状态）
    new_state = state.copy()

    # 更新字段
    new_state["current_node"] = "update_node"
    new_state["execution_history"] = state["execution_history"] + ["update_node"]

    return new_state
```

### 2. 复杂对象更新

```python
async def context_node(state: WorkflowState) -> WorkflowState:
    """上下文节点"""
    # 构建上下文
    context = {
        "history": await load_history(state["task_id"]),
        "documents": await load_documents(state["input_data"]),
        "entities": await extract_entities(state["input_data"]),
    }

    # 更新状态
    return {
        **state,
        "context": context,
        "execution_history": state["execution_history"] + ["context"]
    }
```

### 3. 部分更新

```python
async def partial_update_node(state: WorkflowState) -> Dict[str, Any]:
    """
    部分更新

    LangGraph 支持返回部分状态，会自动合并到完整状态中
    """
    return {
        "current_node": "partial_update_node",
        "execution_history": state["execution_history"] + ["partial_update_node"]
    }
```

---

## 状态验证

```python
from pydantic import BaseModel, validator

class WorkflowStateValidator(BaseModel):
    """状态验证器"""

    task_id: str
    input_data: dict

    @validator("task_id")
    def validate_task_id(cls, v):
        if not v or len(v) == 0:
            raise ValueError("task_id cannot be empty")
        return v


def validate_state(state: WorkflowState) -> bool:
    """验证状态"""
    try:
        WorkflowStateValidator(**state)
        return True
    except Exception as e:
        logger.error(f"State validation failed: {e}")
        return False
```

---

## 状态快照

```python
import json
from datetime import datetime

def create_state_snapshot(state: WorkflowState) -> Dict[str, Any]:
    """创建状态快照（用于 Checkpoint）"""
    return {
        "state": state.copy(),
        "timestamp": datetime.now().isoformat(),
        "version": "1.0"
    }


def restore_state_snapshot(snapshot: Dict[str, Any]) -> WorkflowState:
    """从快照恢复状态"""
    return snapshot["state"]
```

---

## 状态压缩

```python
def compress_state_for_checkpoint(state: WorkflowState) -> WorkflowState:
    """
    压缩状态用于 Checkpoint

    移除不必要的数据，减少存储空间
    """
    compressed = state.copy()

    # 只保留摘要，不保存完整文档内容
    if "context" in compressed:
        context = compressed["context"]
        if "documents" in context:
            # 只保存文档 ID，不保存完整内容
            context["documents"] = [
                {"id": doc["id"], "title": doc["title"]}
                for doc in context["documents"]
            ]

    return compressed
```

---

## 多工作流状态

### Workflow 1: 通用处理

```python
class ProcessingState(WorkflowState):
    """处理工作流状态"""
    pass  # 使用基础 WorkflowState
```

### Workflow 2: 规划

```python
class PlanningState(TypedDict, total=False):
    """规划工作流状态"""

    task_id: str
    goal: str  # 目标
    constraints: List[str]  # 约束

    plan: Dict[str, Any]  # 计划
    alternatives: List[Dict[str, Any]]  # 备选方案

    execution_history: List[str]
    checkpoint_id: Optional[str]
```

---

## 状态迁移

```python
def migrate_state_v1_to_v2(old_state: Dict[str, Any]) -> WorkflowState:
    """
    状态版本迁移

    当 State Schema 升级时，迁移旧版本状态
    """
    new_state = WorkflowState(
        task_id=old_state["task_id"],
        input_data=old_state.get("input", {}),  # 字段重命名
        context=old_state.get("context", {}),
        plan=old_state.get("plan"),
        result=old_state.get("result"),
        validation_report=old_state.get("validation", {}),
        execution_history=old_state.get("history", []),
        current_node=None,  # 新增字段
        checkpoint_id=old_state.get("checkpoint_id"),
        checkpoint_timestamp=None,
        error=None,
        retry_count=0,
        metadata={}
    )

    return new_state
```

---

## 状态调试

```python
def debug_state(state: WorkflowState, message: str = ""):
    """调试状态"""
    logger.debug(f"=== State Debug: {message} ===")
    logger.debug(f"Task ID: {state.get('task_id')}")
    logger.debug(f"Current Node: {state.get('current_node')}")
    logger.debug(f"Execution History: {state.get('execution_history')}")
    logger.debug(f"Error: {state.get('error')}")
    logger.debug(f"Retry Count: {state.get('retry_count')}")

    # 检查关键字段
    if not state.get("result"):
        logger.warning("Result is empty")

    if state.get("validation_report", {}).get("issues"):
        logger.warning(f"Validation issues found: {state['validation_report']['issues']}")
```

---

## 最佳实践

1. **使用 TypedDict** - 提供类型提示，便于 IDE 支持
2. **total=False** - 允许部分字段可选
3. **不可变更新** - 节点返回新状态，不修改输入
4. **文档注释** - 为每个字段添加说明
5. **状态验证** - 关键节点验证状态完整性
6. **状态压缩** - Checkpoint 前压缩状态
7. **版本管理** - 支持状态 Schema 升级迁移
