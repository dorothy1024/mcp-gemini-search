# `client.py` 代码分析

## 整体目标与架构

此脚本的核心是一个**协调器（Orchestrator）**。它本身不执行实际的业务逻辑（如搜索航班），而是利用两个强大的外部系统来完成任务：

1.  **Google Gemini Pro (大脑)**: 一个大型语言模型，负责理解自然语言指令，并决定使用哪个工具来完成任务。
2.  **MCP Flight Search Tool (手/执行者)**: 一个本地的、独立的程序，它暴露了一组可以被调用的功能（API），用于实际执行航班搜索。

该脚本通过 **Function Calling (或 Tool Use)** 模式将这两者连接起来。脚本首先询问“执行者”它能做什么，然后将这些能力告诉“大脑”，让“大脑”根据用户指令来决定调用哪个能力。

---

## 详细代码分析

### 1. 导入 (Imports)

```python
import asyncio
import os
import json
from google import genai
from google.genai import types
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
```

*   `asyncio`: Python 的标准异步 I/O 库。用于高效处理网络请求和进程间通信等耗时的 I/O 操作，避免程序阻塞。
*   `os`: 用于访问操作系统功能，这里通过 `os.getenv` 来安全地获取存储在环境变量中的 API 密钥。
*   `json`: 用于处理 JSON 数据。工具返回的结果是 JSON 格式的字符串，需要用此库来解析（`json.loads`）和格式化输出（`json.dumps`）。
*   `google.genai` 和 `google.genai.types`: Google Gemini 的官方 Python SDK。`genai.Client` 用于创建 API 客户端，`types` 则包含了构建 API 请求所需的特定数据结构，如 `Tool` 和 `GenerateContentConfig`。
*   `mcp` 和 `mcp.client.stdio`: `mcp` (Machine-to-Cloud Protocol) 库的组件。该脚本通过**标准输入/输出 (stdio)** 与本地工具进行通信。

### 2. 全局配置与初始化

```python
client = genai.Client(api_key=os.getenv("GEMINI_API_KEY"))

server_params = StdioServerParameters(
    command="mcp-flight-search",
    args=["--connection_type", "stdio"],
    env={"SERP_API_KEY": os.getenv("SERP_API_KEY")},
)
```

*   `client = genai.Client(...)`: 创建一个 Gemini API 的客户端实例，是与 Gemini 模型所有交互的入口点。
*   `server_params = StdioServerParameters(...)`: 这部分代码定义了如何运行本地工具。
    *   `command="mcp-flight-search"`: 指定了需要执行的命令。
    *   `args=[...]`: 传递给该命令的命令行参数，告知工具通过 `stdio` 模式进行通信。
    *   `env={...}`: 为子进程设置特定的环境变量，表明 `mcp-flight-search` 工具本身可能依赖于其他服务（如 SERP API）并需要其独立的 API 密钥。

### 3. `async def run()` - 核心业务逻辑

这是脚本的主函数，包含了从指令理解到结果返回的完整流程。

#### 步骤 3.1: 建立通信通道

```python
async with stdio_client(server_params) as (read, write):
    async with ClientSession(read, write) as session:
```

*   `stdio_client` 启动 `mcp-flight-search` 子进程，并建立到其标准输入/输出的管道。
*   `ClientSession` 是 `mcp` 库提供的高级抽象，它接管底层的数据流，提供一个方便的会话接口来与工具交互。

#### 步骤 3.2: 初始化与工具发现

```python
prompt = f"Find Flights from Atlanta to Las Vegas 2025-05-05"
await session.initialize()
mcp_tools = await session.list_tools()
```

*   `prompt`: 定义用户的原始请求。
*   `await session.initialize()`: 执行 `mcp` 协议的初始化握手。
*   `mcp_tools = await session.list_tools()`: **关键的“工具发现”步骤**。客户端向工具查询其所支持的功能列表。

#### 步骤 3.3: 格式转换 (为 Gemini 进行适配)

```python
tools = [
    types.Tool(
        function_declarations=[{
            "name": tool.name,
            "description": tool.description,
            "parameters": {
                k: v for k, v in tool.inputSchema.items()
                if k not in ["additionalProperties", "$schema"]
            },
        }]
    ) for tool in mcp_tools.tools
]
```

*   这是一个**至关重要的适配层**。它将 `mcp` 工具返回的自定义格式的工具定义，转换成 Gemini API 所要求的 `types.Tool` 格式，以便模型能够理解和使用。

#### 步骤 3.4: 调用 Gemini 进行推理

```python
response = client.models.generate_content(
    model="gemini-2.5-pro-exp-03-25",
    contents=prompt,
    config=types.GenerateContentConfig(
        temperature=0,
        tools=tools,
    ),
)
```

*   `client.models.generate_content(...)`: 向 Gemini API 发送核心请求。
*   `contents=prompt`: 用户的自然语言输入。
*   `temperature=0`: 将模型的“随机性”设置为零，确保对于工具调用场景，模型能做出最可靠、最确定的决策。
*   `tools=tools`: **将格式化后的工具列表提供给模型**，授权模型在需要时可以调用这些函数。

#### 步骤 3.5: 处理响应并执行工具

```python
if response.candidates[0].content.parts[0].function_call:
    function_call = response.candidates[0].content.parts[0].function_call
    result = await session.call_tool(
        function_call.name, arguments=dict(function_call.args)
    )
```

*   脚本检查模型的响应中是否包含 `function_call` 对象。
*   如果存在，说明 Gemini 决定调用一个工具。脚本会提取出函数名和参数。
*   `await session.call_tool(...)`: **执行工具调用**。使用 `mcp` 会话，将从 Gemini 获取的指令发送给 `mcp-flight-search` 子进程去执行。

#### 步骤 3.6: 结果处理与展示

```python
print("--- Formatted Result ---")
try:
    flight_data = json.loads(result.content[0].text)
    print(json.dumps(flight_data, indent=2))
except json.JSONDecodeError:
    # ... error handling ...
else:
    # ... handle no function call ...
```

*   `try...except` 块确保了代码的健壮性。
*   `json.loads(...)`: 尝试将工具返回的文本内容解析为 Python 对象。
*   `json.dumps(..., indent=2)`: 将解析后的对象转换成一个格式优美、带缩进的 JSON 字符串，并打印出来。
*   错误处理块和 `else` 块确保了在工具返回非预期内容或模型未调用工具时，程序也能给出清晰的反馈。

### 4. 脚本入口

```python
asyncio.run(run())
```

*   这是启动整个异步程序的标准方法。`asyncio.run()` 创建并管理一个事件循环，然后执行 `run` 协程函数。
