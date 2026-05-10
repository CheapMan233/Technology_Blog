# Cursor MCP：推荐、配置与使用说明
---

## 1. MCP 在 Cursor 里是什么

MCP 是一套 **Agent / IDE 与外部工具、数据源通信** 的协议。Cursor 作为 MCP 客户端，可以连接多个 **MCP Server**，从而在对话中调用：

- **工具（Tools）**：例如查文档、分步推理、交互反馈等  
- **资源（Resources）**：部分服务器暴露的只读数据  

启用后，模型会在合适时机 **自动选用** 已连接服务器提供的工具（具体行为取决于 Cursor 版本与对话上下文）。

---

## 2. 配置文件位置与优先级

| 范围 | 典型路径（Windows） | 说明 |
|------|---------------------|------|
| **用户级（全局）** | `C:\Users\<用户名>\.cursor\mcp.json` | 对所有工作区生效（除非你改用项目级覆盖策略） |
| **项目级** | `<项目根>\.cursor\mcp.json` | 仅在该仓库/文件夹内生效，便于团队共享同一套 MCP |

编辑 JSON 后一般需要 **重启 Cursor 或重新加载 MCP**，新服务器才会稳定生效。

---

## 3. 配置结构概览

顶层键为 `mcpServers`，其下每个键名为 **服务器在 Cursor 内的标识**，值为该服务器的连接方式与可选参数。

常见两类连接方式：

1. **远程 HTTP(S)**：使用 `url` 指向 MCP 端点（由 Cursor 按 SSE/Streamable HTTP 等与远端协商，具体以 Cursor 文档为准）。  
2. **本地进程（stdio）**：使用 `command` + `args` 启动本地可执行文件，通过标准输入输出与 MCP 通信。

可选字段示例（以你当前配置为准）：

- **`timeout`**：工具调用超时（秒），适合可能长时间等待用户操作的交互类 MCP。  
- **`autoApprove`**：所列工具名可在无需每次手动确认的情况下调用（降低打断，需权衡安全）。  

---

## 4. mcp推荐

### 4.1 `context7`（远程文档）

```json
"context7": {
  "url": "https://mcp.context7.com/mcp"
}
```

| 项目 | 说明 |
|------|------|
| **类型** | 远程 URL，无需本地安装守护进程 |
| **推荐用途** | 查询 **库 / 框架 / SDK / API / CLI / 云服务** 的 **最新官方文档片段**，例如 React、Next.js、Prisma、云厂商 API 等 |
| **不适合** | 纯业务逻辑调试、大范围重构、从零写脚本、代码评审、泛泛的编程概念讲解（这些应主要靠仓库内代码与通用推理） |

**使用建议**：当问题涉及「某个库的用法、版本迁移、参数含义」时，优先通过 Context7 拉文档，而不是仅凭模型记忆。

---

### 4.2 `sequential-thinking`（本地 uv + Python）

```json
"sequential-thinking": {
  "command": "C:\\Users\\Administrator\\.local\\bin\\uv",
  "args": [
    "--directory",
    "D:\\Program Files (x86)\\MCP\\mcp-sequential-thinking",
    "run",
    "run_server.py"
  ]
}
```

| 项目 | 说明 |
|------|------|
| **类型** | 本地进程；用 **uv** 在指定目录运行 `run_server.py` |
| **推荐用途** | 需要 **显式分步** 的复杂任务：需求模糊、多方案权衡、调试假设较多、架构决策前先结构化推理 |
| **典型工具能力** | 例如记录分步思考（`process_thought`）、会话导入导出、清理历史、生成摘要等（以服务器实际暴露的工具为准） |

**使用建议**：把「一步一步怎么想」交给该 MCP，便于长链路任务不乱跳步；简单一问一答不必强行启用。

**环境注意**：`command` / `args` 里的路径需与本机一致；若迁移机器，请同步修改 `uv` 路径与 MCP 仓库目录。

---

### 4.3 `interactive-feedback-mcp`（本地 uv + 交互）

```json
"interactive-feedback-mcp": {
  "command": "C:\\Users\\Administrator\\.local\\bin\\uv",
  "args": [
    "--directory",
    "D:\\Program Files (x86)\\MCP\\interactive-feedback-mcp",
    "run",
    "server.py"
  ],
  "timeout": 600,
  "autoApprove": [
    "interactive_feedback"
  ]
}
```

| 项目 | 说明 |
|------|------|
| **类型** | 本地进程；同样通过 **uv** 运行项目内 `server.py` |
| **`timeout`: 600** | 最长等待约 10 分钟，适合 **需要用户在侧栏/弹窗中填写反馈** 的流程 |
| **`autoApprove`** | 对工具 `interactive_feedback` 自动批准，减少每次点确认 |

**工具参数要点**（以 schema 为准）：

- `project_directory`：项目根目录的 **完整路径**  
- `summary`：一行摘要，说明当前改动或待确认事项  

**使用建议**：在任务收尾、需要确认方向或验收标准时，通过该工具发起交互；与团队规范配合时，可同时约定「何种 summary 格式」便于复盘。

---

## 5. 通用使用方式（在 Cursor 里）

1. **确认 MCP 已连接**  
   在 Cursor 设置或 MCP 相关面板中查看服务器是否为在线 / 无报错（不同版本 UI 名称可能为「MCP」「Features」等）。

2. **用自然语言触发**  
   无需手写 JSON；说明任务类型即可，例如：  
   - 「用 Context7 查一下 Godot 4.x 里 XXX API」  
   - 「这件事步骤多，用 sequential thinking 分步推一下」  
   - 「结束前用交互反馈收集我的确认」

3. **排查常见问题**  
   - **本地命令失败**：检查 `uv` 是否在指定路径、`--directory` 是否指向正确 MCP 目录、Python 依赖是否已由该项目的 `pyproject.toml` / `uv sync` 装好。  
   - **远程 Context7 不可用**：检查网络、代理、防火墙及 Cursor 是否允许访问该 URL。  
   - **工具总被要求授权**：检查是否配置了 `autoApprove`，或 Cursor 的全局 MCP 批准策略。

---

## 6. 扩展与维护建议

- **新增服务器**：在 `mcpServers` 下增加新键；stdio 型保证 `command` 可执行、`args` 能启动 MCP 入口脚本。  
- **敏感环境**：对能执行系统命令或访问网络的 MCP 要谨慎使用 `autoApprove`。  
- **版本与文档**：Cursor 对 MCP 传输方式（如 SSE、Streamable HTTP）的细节可能随版本更新，以 [Cursor 官方文档](https://docs.cursor.com) 为准。

---

## 附录：当前配置的完整参考（示例）

以下为文档编写时所依据的配置快照，便于对照；若你本机已修改，以实际 `mcp.json` 为准。

```json
{
  "mcpServers": {
    "context7": {
      "url": "https://mcp.context7.com/mcp"
    },
    "sequential-thinking": {
      "command": "C:\\Users\\Administrator\\.local\\bin\\uv",
      "args": [
        "--directory",
        "D:\\Program Files (x86)\\MCP\\mcp-sequential-thinking",
        "run",
        "run_server.py"
      ]
    },
    "interactive-feedback-mcp": {
      "command": "C:\\Users\\Administrator\\.local\\bin\\uv",
      "args": [
        "--directory",
        "D:\\Program Files (x86)\\MCP\\interactive-feedback-mcp",
        "run",
        "server.py"
      ],
      "timeout": 600,
      "autoApprove": [
        "interactive_feedback"
      ]
    }
  }
}
```
