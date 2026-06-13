# Hermes Agent 技术架构概览

Hermes Agent 的核心架构可以概括为“一套 `AIAgent`，多种入口壳”。CLI、消息网关、TUI、Electron 桌面端、批处理、定时任务等入口都会把用户输入整理成一次 agent turn，然后调用同一套 agent 核心循环。

## 总体分层

```text
用户入口层
CLI / Gateway / TUI / Desktop / ACP / Cron / Batch

会话与适配层
SessionDB、SessionStore、平台 Adapter、JSON-RPC、slash command、配置解析

Agent 核心层
AIAgent -> run_conversation -> model API loop -> tool call loop

能力扩展层
toolsets -> model_tools -> tools.registry -> tools/*.py / plugins / MCP / memory / context engine

基础设施层
SQLite session、日志、prompt cache、context compression、credential/provider routing
```

## 主要入口与调用顺序

### 经典 CLI

```text
cli.py main()
-> HermesCLI
-> HermesCLI.run()
-> 用户输入或 slash command
-> 构造或复用 AIAgent
-> AIAgent.run_conversation()
```

关键位置：

- `cli.py:13313` - `main()`，CLI 程序入口。
- `cli.py:3136` - `HermesCLI` 类。
- `cli.py:10762` - `HermesCLI.run()` 交互主循环。
- `hermes_cli/cli_agent_setup_mixin.py:343` - CLI 路径创建 `AIAgent`。

### 消息网关

```text
gateway/run.py main()
-> start_gateway()
-> GatewayRunner
-> 平台 adapter 接收消息
-> GatewayRunner._handle_message()
-> GatewayRunner._handle_message_with_agent()
-> 创建或缓存 AIAgent
-> AIAgent.run_conversation()
-> adapter 发送响应
```

关键位置：

- `gateway/run.py:16197` - 网关 CLI 入口。
- `gateway/run.py:15739` - `start_gateway()`。
- `gateway/run.py:1977` - `GatewayRunner`。
- `gateway/run.py:6348` - `_handle_message()`，网关核心消息管线。
- `gateway/run.py:7864` - `_handle_message_with_agent()`。
- `gateway/run.py:14006` - 普通消息路径创建 `AIAgent`。

### TUI

```text
ui-tui 前端
-> JSON-RPC prompt.submit
-> tui_gateway/server.py
-> _run_prompt_submit()
-> session["agent"].run_conversation()
-> 事件流返回 TUI 渲染
```

关键位置：

- `ui-tui/src/app/useSubmission.ts:87` - TUI 前端提交消息。
- `ui-tui/src/app/useSubmission.ts:110` - 调用 `prompt.submit`。
- `tui_gateway/server.py:5037` - `prompt.submit` RPC 方法。
- `tui_gateway/server.py:5290` - `_run_prompt_submit()`。
- `tui_gateway/server.py:3142` - TUI 后端创建 `AIAgent`。

### Electron 桌面端

桌面端是独立 React/assistant-ui 聊天界面，不嵌入 `hermes --tui`，但后端仍复用 `tui_gateway` JSON-RPC。

```text
Desktop React UI
-> requestGateway("prompt.submit")
-> tui_gateway/server.py prompt.submit
-> _run_prompt_submit()
-> AIAgent.run_conversation()
```

slash command 走：

```text
Desktop slash handler
-> requestGateway("slash.exec")
-> 失败或特殊命令回退 command.dispatch
-> skill/alias/exec/plugin 结果
-> 必要时作为普通 prompt 再提交
```

关键位置：

- `apps/desktop/src/app/session/hooks/use-prompt-actions.ts:1406` - 桌面端提交普通 prompt。
- `apps/desktop/src/app/session/hooks/use-prompt-actions.ts:850` - 桌面端 slash command 执行路径。
- `apps/desktop/src/app/chat/composer/hooks/use-slash-completions.ts:133` - slash 补全读取 `commands.catalog`。

### 批处理

```text
batch_runner.py main()
-> 读取 JSONL dataset
-> 每个 prompt 创建 AIAgent
-> AIAgent.run_conversation()
-> 提取 tool usage / reasoning stats / trajectory
```

关键位置：

- `batch_runner.py:1147` - 批处理入口。
- `batch_runner.py:325` - 单条 prompt 创建 `AIAgent`。
- `batch_runner.py:349` - 调用 `run_conversation()`。

## Agent 核心调用链

```text
AIAgent.__init__
-> agent_init 初始化 provider/client/config/tool surface/memory/context engine
-> get_tool_definitions()
-> run_conversation()
    -> build_turn_context()
    -> 恢复或构建 system prompt
    -> while api_call_count < max_iterations
        -> 组装 messages + tools
        -> 调模型 API
        -> 如果 assistant 返回 tool_calls
            -> _execute_tool_calls()
            -> agent.tool_executor
            -> model_tools.handle_function_call()
            -> tools.registry 中对应 handler
            -> tool 结果 append 为 tool message
            -> continue 下一轮模型调用
        -> 如果没有 tool_calls
            -> final_response
            -> 持久化 session / usage / hooks
```

关键位置：

- `run_agent.py:320` - `AIAgent` 类。
- `run_agent.py:5112` - `AIAgent.run_conversation()` 薄转发。
- `agent/conversation_loop.py:1` - 真实 conversation loop 模块。
- `agent/conversation_loop.py:461` - 主 API/tool loop。
- `agent/conversation_loop.py:3498` - assistant tool call 分支。
- `agent/conversation_loop.py:3731` - 执行工具调用。
- `agent/conversation_loop.py:3830` - 无工具调用时进入最终响应分支。
- `run_agent.py:5011` - `_execute_tool_calls()`，选择串行或并行执行。
- `run_agent.py:5097` - 并行工具执行转发。
- `run_agent.py:5102` - 串行工具执行转发。
- `agent/tool_executor.py:1` - 工具执行实现模块。
- `model_tools.py:876` - `handle_function_call()`，工具分发入口。

## 工具与扩展机制

工具是注册表驱动，不是硬编码散落调用。

```text
tools/*.py
-> import 时 registry.register(...)

model_tools.py
-> discover_builtin_tools()
-> discover_plugins()
-> get_tool_definitions()

AIAgent
-> agent.tools / valid_tool_names

模型返回 tool_calls
-> handle_function_call()
-> registry handler
```

关键位置：

- `tools/registry.py:57` - `discover_builtin_tools()` 导入自注册工具模块。
- `tools/registry.py:151` - `ToolRegistry`。
- `tools/registry.py:234` - `registry.register()`。
- `model_tools.py:180` - 启动时发现内置工具。
- `model_tools.py:196` - 发现插件工具。
- `model_tools.py:272` - `get_tool_definitions()`，生成模型 API tool schema。
- `toolsets.py:31` - `_HERMES_CORE_TOOLS` 默认核心工具列表。
- `agent/agent_init.py:949` - `AIAgent` 初始化时加载工具定义。

## Session、Prompt Cache 与持久化

Hermes 很重视长会话 prompt cache。`run_conversation()` 会优先从会话库恢复已保存的 system prompt，只有新会话或恢复失败时才重建。

```text
SessionDB
-> session row 保存 system_prompt / messages / cwd 等
-> conversation loop 恢复 system prompt
-> 同一会话复用稳定 prompt prefix
-> context compression 是少数允许改变历史上下文的路径
```

关键位置：

- `agent/conversation_loop.py:225` - `_restore_or_build_system_prompt()`。
- `agent/conversation_loop.py:277` - 继续会话时复用已存 system prompt。
- `agent/conversation_loop.py:295` - 新会话构建 system prompt。
- `agent/conversation_loop.py:330` - 保存 system prompt。
- `hermes_state.py:583` - `SessionDB`。
- `hermes_state.py:1391` - `update_system_prompt()`。
- `hermes_state.py:2363` - `replace_messages()`。
- `agent/system_prompt.py:62` - system prompt 分块构建。
- `agent/system_prompt.py:387` - `build_system_prompt()`。

## 核心设计结论

这个项目的“窄腰”是：

```text
AIAgent + model_tools + tools.registry
```

入口很多，但都尽量不复制 agent 逻辑。平台差异主要通过 adapter、callback、session metadata、toolset 配置和 JSON-RPC 包装解决。

最重要的一条主链是：

```text
入口收到输入
-> 找到或创建 session
-> 创建或复用 AIAgent
-> AIAgent.run_conversation()
-> 调模型
-> 模型要工具则 registry dispatch
-> 工具结果回填 messages
-> 再调模型
-> 最终响应回到入口层渲染或发送
```
