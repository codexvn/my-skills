---
name: fastmcp-gateway-usage
description: fastmcp-gateway 网关用法（必读技能）。只要环境中存在 fastmcp-gateway 的元工具（discover_tools、get_tool_schema、execute_tool、refresh_registry 中的任何一个），就必须先完整阅读本技能，并且每次新会话开始时都要调用 discover_tools() 获取当前全部可用 MCP 服务器与工具的实时清单，再继续任务。不得依赖记忆、不得使用固定列表——网关聚合的服务器会随时增减，只有 discover_tools() 的返回值才是当前事实。此外：调用任何 MCP 工具、排查工具调用失败、给网关新增或变更上游、重启网关服务时也必须读本技能。
---

# fastmcp-gateway 网关用法

**第一条规则：每次新会话开始，只要网关的元工具可用，先调用 `discover_tools()` 获取当前的完整清单，再处理任何具体任务。** 清单以 `discover_tools()` 的实时返回为准，不要凭记忆或固定列表判断有哪些工具——网关聚合的服务器会随时增减。

本机 MCP 工具链分两层，方向固定：MCP 客户端先连 fastmcp-gateway 网关，网关再聚合 MCPJungle，MCPJungle 下面才是各个 MCP 服务器。网关只向客户端暴露四个元工具，用「按需发现」替代「一次加载全部 schema」，避免上百个工具的定义撑爆上下文。

## 端点与挂载

网关以 Servy Windows 服务常驻，服务名为 `fastmcp-gateway`，HTTP 端点是 `http://127.0.0.1:44265/mcp`（Streamable HTTP 协议）。客户端挂载统一写成 `{"type": "http", "url": "http://127.0.0.1:44265/mcp"}`；不要用 stdio 方式拉起网关进程。

## 四个元工具的工作流

1. `discover_tools()`：不带参数时列出全部 domain（即上游服务器）和各自的工具数量；带 `domain="wiki"` 时列出该域每个工具的签名；传 `format="schema"` 改为返回 JSON 摘要。使用任何工具前都应先发现、后调用，不要凭记忆猜工具名。
2. `get_tool_schema("wiki__confluence_search")`：当签名不够用（需要参数说明、嵌套对象的必填字段或取值范围）时，取该工具的完整 JSON Schema。名字必须精确匹配，名字写错时返回 `tool_not_found`，并附最多三个近似名供纠正。
3. `execute_tool("wiki__confluence_search", {"query": "...", "limit": 1})`：把调用路由到正确的上游服务器执行，成功返回 `{"tool": ..., "result": ...}`，失败返回结构化错误，错误码会说明下一步该怎么处理。
4. `refresh_registry()`：重新查询全部上游并返回每个域的增删摘要，也能救回启动时不可达的上游。已加载上游的 schema 发生变更时会被完整性门禁拒绝并维持旧工具，操作者需要按返回的 digest 调用 `POST /registry/servers/refresh?expected_digest=<digest>`（请求体 `{"domain": "..."}`）确认后才生效。

工具的命名规则是「服务器名__工具名」，中间是双下划线。

## CLI 调用（PowerShell）

通用检查用 `mcp-inspector`（npm 全局安装，任意 MCP 服务器通吃，HTTP 调用实测约 0.9 秒）：

```powershell
mcp-inspector --cli http://127.0.0.1:44265/mcp --method tools/list
mcp-inspector --cli http://127.0.0.1:44265/mcp --method tools/call --tool-name execute_tool --tool-arg tool_name=wiki__confluence_search --tool-arg arguments='{"query":"MCP","limit":1}'
```

按网关语义调用用 `fm.cmd` 启动器（元工具参数名为 `tool_name` 与 `arguments`）：

```powershell
D:\Tools\fastmcp-service\fm.cmd list http://127.0.0.1:44265/mcp
D:\Tools\fastmcp-service\fm.cmd call http://127.0.0.1:44265/mcp discover_tools
D:\Tools\fastmcp-service\fm.cmd call http://127.0.0.1:44265/mcp execute_tool -a '{"tool_name":"wiki__confluence_search","arguments":{"query":"MCP","limit":1}}'
```

## 安装与升级

本机运行的是**本地 fork**，不是 PyPI 原版：仓库主副本在 `D:\Tools\fastmcp-gateway`（`v0.34.0-fix` 分支，该目录要保持 `git status` 干净用于向上游推送，不得在其中放置任何自定义文件）。fork 相对上游 0.34.0 放行了两类被原版静默拒绝的内容：工具名中的连字符，以及嵌套超过五层的参数 schema。

部署用 venv 在 `D:\Tools\fastmcp-gateway\.venv`，装的是 fork 构建的 wheel（`D:\Tools\fastmcp-service\wheels\fastmcp_gateway-0.34.0-py3-none-any.whl`）。升级时在干净的仓库上重新构建 wheel、在 venv 内 `python -m pip install --force-reinstall --no-deps <wheel>` 覆盖安装，然后重启服务。

通用安装（其他机器或全新环境）用 `pip install fastmcp-gateway`，CLI 以 `python -m fastmcp_gateway` 启动，必需的环境变量是 `GATEWAY_UPSTREAMS='{"domain": "http://host/mcp"}'`。注意 PyPI 原版带上面说的连字符与深嵌套限制。

## 配置与新增上游

本机的全部网关配置都经 Servy 服务的 `EnvironmentVariables` 注入，包括 `GATEWAY_HOST=127.0.0.1`、`GATEWAY_PORT=44265`、`GATEWAY_UPSTREAMS` 和 `GATEWAY_DOMAIN_DESCRIPTIONS`；配置文件是 `D:\Tools\Servy\configs\fastmcp-gateway.servy.json`。修改环境变量后必须重新 import 配置再 restart，直接重启不会重读配置。

上游需要认证时用 `GATEWAY_UPSTREAM_HEADERS='{"domain": {"Authorization": "Bearer ..."}}'` 按域设置请求头；没有设置的域沿用请求头透传。

要新增 MCP 服务器，正确路径是先在 MCPJungle 侧注册（见 mcpjungle-usage 技能），然后**重启网关服务**让它重新 populate——网关不会自动吸纳 MCPJungle 的清单变更。

## 服务管理与红线

```powershell
gsudo servy-cli restart -n fastmcp-gateway
```

日志在 `D:\Tools\Servy\logs\fastmcp-gateway\`。

红线：**不得把 fastmcp-gateway 自己注册进 MCPJungle**。网关的上游就是 MCPJungle，注册会形成自引用，双方在注册时互相等待对方的工具清单，可能死锁；而且四个元工具平铺进聚合端点会破坏渐进发现的用途。2026-10-20 曾误注册过一次，已注销。
