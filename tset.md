# Phase 1: ToolProvider 解耦 — 工作总结

## 完成的改动

### 1. 创建了 ToolProvider 协议

#### [tool_provider.py](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/tool_provider.py)

定义了两个核心类型：
- **[ToolProviderContext](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/tool_provider.py#15-33)** — 封装 session 级别的上下文（`computer_use_runtime`, `sandbox_cfg`, `session_id`）
- **[ToolProvider](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/tool_provider.py#35-49)** — Protocol 接口，包含 [get_tools()](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/tool_provider.py#42-45) 和 [get_system_prompt_addon()](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/computer/computer_tool_provider.py#245-257) 两个方法

---

### 2. 实现了 ComputerToolProvider

#### [computer_tool_provider.py](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/computer/computer_tool_provider.py)

将原 [astr_main_agent.py](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/astr_main_agent.py) 中硬编码的工具注入逻辑全部迁移到了此处：

| 原位置 | 迁移到 |
|--------|--------|
| `_apply_local_env_tools()` | `ComputerToolProvider.get_tools()` (`runtime=="local"`) |
| [_apply_sandbox_tools()](file:///e:/Astrbot_branch/Node/AstrBot/tests/unit/test_astr_main_agent.py#1388-1401) | `ComputerToolProvider._sandbox_tools()` |
| [_build_local_mode_prompt()](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/computer/computer_tool_provider.py#141-154) | [_build_local_mode_prompt()](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/computer/computer_tool_provider.py#141-154) (模块内) |
| `SANDBOX_MODE_PROMPT` + Neo prompts | `ComputerToolProvider._sandbox_prompt_addon()` |

额外功能：
- **Lazy 工具单例缓存** — 工具实例按类别（sandbox/local/neo/browser）懒创建并缓存
- **[get_all_tools()](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/computer/computer_tool_provider.py#164-230)** — 为 WebUI 工具管理提供全量工具列表（`active=False`，仅展示不注入）

---

### 3. 拆分了 [astr_main_agent_resources.py](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/astr_main_agent_resources.py)

原来的 [astr_main_agent_resources.py](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/astr_main_agent_resources.py)（~485 行）被拆分成 4 个独立模块：

| 新模块 | 内容 |
|--------|------|
| [prompts.py](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/tools/prompts.py) | 所有系统 prompt 常量（LLM_SAFETY_MODE、TOOL_CALL_PROMPT、CHATUI、LIVE_MODE 等） |
| [kb_query.py](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/tools/kb_query.py) | [KnowledgeBaseQueryTool](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/tools/kb_query.py#22-58) + [retrieve_knowledge_base()](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/tools/kb_query.py#60-132) |
| [send_message.py](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/tools/send_message.py) | [SendMessageToUserTool](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/tools/send_message.py#26-211) |
| [cron_tools.py](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/tools/cron_tools.py) | 3 个 Cron 工具（create/delete/list） |

原文件已删除。

---

### 4. 重构了 [astr_main_agent.py](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/astr_main_agent.py)

- **删除了 20+ 个工具 import**（`EXECUTE_SHELL_TOOL`, `BROWSER_EXEC_TOOL`, `LOCAL_PYTHON_TOOL` 等）
- **删除了** `_apply_local_env_tools()`, [_build_local_mode_prompt()](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/computer/computer_tool_provider.py#141-154), [_apply_sandbox_tools()](file:///e:/Astrbot_branch/Node/AstrBot/tests/unit/test_astr_main_agent.py#1388-1401) 三个函数
- **[build_main_agent()](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/astr_main_agent.py#869-1118)** 中的工具注入改为调用 [ComputerToolProvider](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/computer/computer_tool_provider.py#161-309)

文件从 **1223 行** 缩减到 **1118 行**。

---

## 当前遗留问题

> [build_main_agent()](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/astr_main_agent.py#869-1118) 仍然直接 import 了 [ComputerToolProvider](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/computer/computer_tool_provider.py#161-309)（L21）

```python
from astrbot.core.computer.computer_tool_provider import ComputerToolProvider
...
_computer_provider = ComputerToolProvider()  # L1036
```

这意味着 **agent 模块仍然知道 computer provider 的具体类型**，没有完全实现依赖反转。

### 计划中的方案

在 [MainAgentBuildConfig](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/astr_main_agent.py#68-124) 上增加 `tool_providers: list[ToolProvider]` 字段，由上游（`InternalAgentSubStage.initialize()`）创建并注入 provider，[build_main_agent()](file:///e:/Astrbot_branch/Node/AstrBot/astrbot/core/astr_main_agent.py#869-1118) 只遍历 `config.tool_providers`：

```
InternalAgentSubStage (知道 ComputerToolProvider)
    ↓ 注入 tool_providers=[ComputerToolProvider()]
MainAgentBuildConfig
    ↓ 传递
build_main_agent (只遍历 tool_providers，不 import 具体类)
```

这一步尚未完成。