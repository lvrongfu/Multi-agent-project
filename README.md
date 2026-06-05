## 启动命令

```bash
# 后端 (FastAPI, 端口 8000)
python -m api.server

# 前端 (Vue 3 + Vite, 开发模式)
cd ui && npm run dev
```

项目使用 `.venv` 虚拟环境，运行前需激活。

## 架构概览

### 智能体分层：1 主 Agent + 3 子 Agent

```
用户请求 → FastAPI → run_deep_agent() → main_agent (DeepAgents)
                                            ├── task → 网络搜索助手 (Tavily)
                                            ├── task → 数据库查询助手 (MySQL)
                                            └── task → RAGFlow助手 (知识库)
```

- **主 Agent** (`agent/main_agent.py`) — 基于 `deepagents.create_deep_agent` 创建，接收用户自然语言指令，通过 `tool_calls` 中 `name == 'task'` 的调用自动识别并调度子 Agent。主 Agent 自身持有 3 个工具（generate_markdown / convert_md_to_pdf / read_file_content），文件生成类操作由主 Agent 亲自执行，不委托给子 Agent。
- **子 Agent** 定义在 `agent/subagents/` 目录，均为 dict 格式（name / description / system_prompt / tools），分别绑定各自的专属工具。
- **系统提示词** 统一管理在 `prompt/prompts.yml`，通过 `agent/prompts.py` 以 YAML 加载，解耦提示词与代码逻辑。
- **LLM 接入** 在 `agent/llm.py`，使用 `langchain.chat_models.init_chat_model`（当前模型：deepseek-chat），OA API 兼容方式对接。

### 请求链路与会话隔离

1. 前端 `POST /api/task` 携带 `query` + 可选 `thread_id` → 服务端生成 UUID 作为 session_id
2. `asyncio.create_task(run_deep_agent(...))` 异步执行，立即返回 `{"status": "started", "thread_id"}`
3. `run_deep_agent()` 内部：
   - 创建 `output/session_{id}/` 工作目录
   - 检查 `updated/session_{id}/` 中是否有上传文件，若有则复制到工作目录并拼接到提示词
   - 通过 `ContextVar` 将会话目录和 thread_id 注入当前协程上下文（`api/context.py`），工具函数可在任意深处直接获取，无需层层传参
   - 执行 `main_agent.astream()` 流式迭代，区分 `node_name == 'model'` 的两种输出：`tool_calls`（调用子 Agent 或工具）与 `content`（最终结果）

### WebSocket 实时推送与监控

- `api/monitor.py` 提供全局单例 `monitor`（`ToolMonitor`）和 `manager`（`ConnectionManager`）
- WebSocket 路由：`ws://localhost:8000/ws/{thread_id}`
- 每个工具/子 Agent 调用处通过 `monitor.report_tool()` / `monitor.report_assistant()` 埋点
- 推送策略：同一事件循环内直接 `create_task`，跨线程使用 `asyncio.run_coroutine_threadsafe`
- 三种事件类型：`tool_start` / `assistant_call` / `task_result`，payload 携带 thread_id 实现会话级消息隔离

### 工具箱设计

所有工具采用 LangChain `@tool` 装饰器封装，位于 `tools/` 目录：

| 工具 | 用途 | 所属 |
|------|------|------|
| `internet_search` | Tavily 网络搜索（支持 general/news/finance 分类） | 网络搜索 Agent |
| `list_sql_tables` / `get_table_data` / `execute_sql_query` | MySQL 三步渐进式查询（发现表→预览→自定义SQL） | 数据库查询 Agent |
| `get_assistant_list` / `create_ask_delete` | RAGFlow 知识库查询（发现助手→创建临时会话→流式问答→销毁会话） | RAGFlow Agent |
| `generate_markdown` | 生成 Markdown 文件到会话目录 | 主 Agent |
| `convert_md_to_pdf` | MD→PDF 转换（基于 Word 渲染引擎） | 主 Agent |
| `read_file_content` | 多格式文件解析（.md/.docx/.pdf/.xlsx） | 主 Agent |

工具函数获取文件路径时，统一通过 `get_session_context()` 获取当前会话目录，再经由 `utils/path_utils.py` 的 `resolve_path()` 完成路径清洗（禁止越权访问 `..` 等）。

### 前端

- Vue 3 + TypeScript + Vite（`ui/` 目录），依赖 axios（HTTP）和 marked（Markdown 渲染）
- 通过 REST API 提交任务和上传文件，通过 WebSocket 接收实时执行进度

## 关键外部依赖

- **DeepAgents** — LangGraph 上层的多 Agent 编排框架，提供 `create_deep_agent()` API
- **RAGFlow** — 开源 RAG 引擎，通过 `ragflow_sdk.RAGFlow` 客户端交互
- **Tavily** — 网络搜索 API
- **DeepSeek** — LLM 服务（OA API 兼容协议）
- **MySQL** — 业务数据库（药品信息、库存、销售数据）
