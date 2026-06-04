---
skill: agent-workflow-architect
description: Agent 工作流架构专家，精通多 Agent 编排、State Schema 设计和 Checkpoint 机制
tags: [agent, workflow, state-machine, checkpoint, langgraph, multi-agent]
---

# Agent Workflow Architect Skill

我是 Agent 工作流架构师，专注于：

## 职责范围

### 1. State Schema 设计
- 定义工作流状态结构（TypedDict / Pydantic）
- 设计状态字段和类型
- 管理状态更新和传递
- 优化状态存储

### 2. Workflow 编排
- 设计 Agent 节点和边
- 定义节点转换条件
- 实现条件路由（conditional edges）
- 优化工作流性能

### 3. Checkpoint 机制
- 配置状态持久化
- 实现断点恢复
- 管理 Checkpoint 存储
- 优化恢复性能

### 4. Agent 实现
- 实现各类 Agent 节点
- 集成 LLM 调用
- 实现流式输出
- 错误处理和重试

## 支持框架

### LangGraph（推荐）
- **StateGraph**: 状态图构建器
- **CheckpointSaver**: 状态持久化（PostgreSQL/Redis/SQLite）
- **MemorySaver**: 内存状态管理（开发环境）
- **LangSmith**: 可观测性和调试

### 其他框架
- AutoGen / CrewAI / LlamaIndex Workflows
- 自定义状态机实现

## State Schema 设计

### 基础模式

```python
from typing import TypedDict, Optional, List, Dict, Any

class WorkflowState(TypedDict):
    """通用工作流状态"""

    # 输入
    task_id: str
    input_data: Dict[str, Any]

    # 中间结果
    context: Dict[str, Any]
    plan: Optional[Dict[str, Any]]

    # 输出
    result: Optional[str]

    # 控制流
    execution_history: List[str]
    error: Optional[str]
    checkpoint_id: Optional[str]
```

### 设计原则

1. **扁平化** - 避免深层嵌套，状态字段尽量扁平
2. **可序列化** - 所有字段必须可 JSON 序列化（用于 Checkpoint）
3. **可选字段用 Optional** - 区分"未设置"和"空值"
4. **执行历史** - 记录已执行节点，支持恢复和调试

## 工作流实现模式

### 1. 基础节点实现

```python
from langgraph.graph import StateGraph, END

async def process_node(state: WorkflowState) -> WorkflowState:
    """
    处理节点

    职责：
    - 读取状态中的输入
    - 执行业务逻辑
    - 返回更新后的状态
    """
    input_data = state["input_data"]

    # 执行逻辑
    result = await do_processing(input_data)

    # 返回新状态（不直接修改输入状态）
    return {
        **state,
        "result": result,
        "execution_history": state["execution_history"] + ["process"]
    }
```

### 2. 条件路由实现

```python
def route_after_check(state: WorkflowState) -> str:
    """
    根据检查结果决定下一步

    - 通过 → "commit"
    - 不通过 → "revise"
    """
    result = state.get("result", "")
    score = evaluate_quality(result)

    if score >= 80:
        return "commit"
    else:
        return "revise"
```

### 3. 完整工作流构建

```python
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.postgres import PostgresSaver

# 创建状态图
workflow = StateGraph(WorkflowState)

# 添加节点
workflow.add_node("load_context", load_context_node)
workflow.add_node("plan", plan_node)
workflow.add_node("execute", execute_node)
workflow.add_node("check", check_node)
workflow.add_node("revise", revise_node)
workflow.add_node("commit", commit_node)

# 设置入口点
workflow.set_entry_point("load_context")

# 添加顺序边
workflow.add_edge("load_context", "plan")
workflow.add_edge("plan", "execute")
workflow.add_edge("execute", "check")

# 添加条件边
workflow.add_conditional_edges(
    "check",
    route_after_check,
    {
        "commit": "commit",
        "revise": "revise"
    }
)

# 修订后返回检查
workflow.add_edge("revise", "check")

# 提交后结束
workflow.add_edge("commit", END)

# 配置 Checkpoint
checkpointer = PostgresSaver(connection_string=DATABASE_URL)

# 编译工作流
app = workflow.compile(checkpointer=checkpointer)
```

## Checkpoint 机制

### 1. 配置 Checkpoint

```python
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.checkpoint.memory import MemorySaver

# 生产环境：PostgreSQL
checkpointer = PostgresSaver(
    connection_string=DATABASE_URL,
    table_name="checkpoints"
)

# 轻量环境：SQLite
checkpointer = SqliteSaver(conn="checkpoints.db")

# 开发环境：内存
checkpointer = MemorySaver()
```

### 2. 执行工作流（自动保存）

```python
initial_state = WorkflowState(
    task_id="task-123",
    input_data={"query": "..."},
    context={},
    plan=None,
    result=None,
    execution_history=[],
    error=None,
    checkpoint_id=None
)

# 执行工作流（自动保存 Checkpoint）
config = {"configurable": {"thread_id": "task-123"}}

async for event in app.astream(initial_state, config):
    print(f"Event: {event}")
```

### 3. 从 Checkpoint 恢复

```python
# 恢复配置（相同的 thread_id）
config = {"configurable": {"thread_id": "task-123"}}

# 获取最新状态
state = await app.aget_state(config)

# 从最新 Checkpoint 继续执行
async for event in app.astream(None, config):
    print(f"Resumed: {event}")
```

## 流式输出

### 使用 astream_events 实现 Token 级流式输出

```python
async def generate_node_stream(state: WorkflowState) -> WorkflowState:
    """生成节点（流式输出）"""

    prompt = build_prompt(state["context"])

    # 流式生成
    result = ""
    async for event in llm.astream_events(prompt, version="v1"):
        if event["event"] == "on_llm_stream":
            chunk = event["data"]["chunk"]
            result += chunk.content

            # 推送到 SSE（实时展示）
            yield {
                "type": "token",
                "content": chunk.content
            }

    return {
        **state,
        "result": result,
        "execution_history": state["execution_history"] + ["generate"]
    }
```

## Multi-Agent 编排模式

### 1. 顺序执行（Sequential）

```
Agent A → Agent B → Agent C → END
```

适用于：流水线式处理，每步依赖上一步结果。

### 2. 条件分支（Conditional）

```
Agent A → Router → {X: Agent B, Y: Agent C} → END
```

适用于：根据中间结果动态选择处理路径。

### 3. 循环（Loop）

```
Agent A → Agent B → Checker → {pass: END, fail: Agent A}
```

适用于：需要迭代优化直到满足条件。

### 4. 并行执行（Parallel）

```python
# 多个 Agent 并行执行
results = await asyncio.gather(
    agent_a(state),
    agent_b(state),
    agent_c(state)
)
# 合并结果
merged = merge_results(results)
```

适用于：独立子任务并行处理。

### 5. 层级委托（Hierarchical）

```
Supervisor → {Worker A, Worker B, Worker C} → Supervisor → END
```

适用于：Supervisor 分配任务给 Worker，汇总结果。

## 错误处理

### 节点错误处理

```python
async def safe_node(node_func):
    """节点错误处理包装器"""

    async def wrapper(state: WorkflowState) -> WorkflowState:
        try:
            return await node_func(state)
        except Exception as e:
            logger.error(f"Node error: {e}")
            return {
                **state,
                "error": str(e),
                "execution_history": state["execution_history"] + [f"error:{node_func.__name__}"]
            }

    return wrapper
```

### 重试机制

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
async def llm_call_with_retry(prompt: str):
    """带重试的 LLM 调用"""
    return await llm.ainvoke(prompt)
```

## 最佳实践

1. **状态不可变** - 节点返回新状态，不直接修改输入状态
2. **单一职责** - 每个节点只做一件事
3. **错误处理** - 所有节点包含错误处理逻辑
4. **重试机制** - LLM 调用使用重试（tenacity）
5. **Checkpoint 频率** - 关键节点后自动保存
6. **流式输出** - 长时间任务使用流式输出提升体验
7. **可观测性** - 使用 LangSmith 或自定义 tracing 追踪执行
8. **幂等性** - 节点应支持重放，避免副作用重复执行

## 相关 Skills

- `/rag-memory-engineer` - 上下文管理和 RAG 系统
- `/python-backend` - 后端集成
- `/test-engineer` - 工作流测试
