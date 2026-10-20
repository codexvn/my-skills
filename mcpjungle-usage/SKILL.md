---
name: mcpjungle-usage
description: MCPJungle 服务器注册管理器的用法（注册、注销、导出、直连调用）。何时使用：需要新增、删除或导出 MCP 服务器注册，直连调用 MCPJungle 侧的工具（不经网关），排查某服务器的工具消失，或处理服务器命名问题时。
---

# MCPJungle 用法

MCPJungle 管理本机全部 MCP 服务器的注册，聚合 chrome-devtools、rider、wiki、amap-maps、ssh-mcp-server、cn-scraper 等服务器，作为 fastmcp-gateway 网关的上游。它监听 `http://127.0.0.1:10236/mcp`，以 Servy Windows 服务常驻，工具目录在 `D:\Tools\MCPJungle`（含 exe、configs、exports 与注册数据库）。

工具链方向固定：客户端 → fastmcp-gateway（44265）→ MCPJungle（10236）→ 各服务器。日常的工具调用应该走网关（见 fastmcp-gateway-usage 技能）；本技能只覆盖 MCPJungle 本身的管理。

## CLI 速查

以下命令都在 `D:\Tools\MCPJungle` 目录下执行。

```powershell
# 注册；同名服务已存在时加 --force 覆盖（先注销旧注册再注册新的）
.\mcpjungle.exe register --conf '.\configs\amap-maps.json'
.\mcpjungle.exe register --force --conf '.\configs\amap-maps.json'

# 注销，位置参数是服务名；该操作不删除 configs 里的源配置文件
.\mcpjungle.exe deregister amap-maps

# 导出注册表中的全部实体（注意：不是只导出一个服务）
$exportDir = '.\exports\current'; New-Item -ItemType Directory -Force -Path $exportDir | Out-Null
.\mcpjungle.exe export --dir $exportDir

# 直连调用 MCPJungle 侧的工具（不经网关），--input 接 JSON 字符串
.\mcpjungle.exe --registry http://127.0.0.1:10236 invoke wiki__confluence_search --input '{"query":"MCP","limit":1}'

# 服务状态
servy-cli status -n MCPJungle
```

## 配置文件格式

配置放在 `configs\` 目录，每个服务一个 JSON。远程 Streamable HTTP 服务的完整形态如下（凭证必须只写在受保护的运行配置里，不进 README、示例或日志）：

```json
{
  "name": "amap-maps",
  "transport": "streamable_http",
  "description": "高德地图 MCP Server",
  "url": "https://mcp.amap.com/mcp?key=YOUR_KEY",
  "session_mode": "stateful"
}
```

本地服务不用 `url`，改用 `command` 与 `args` 描述启动方式。新建配置时应沿用现有文件的字段命名。

## 新增服务流程

1. 先确认服务供应商、传输协议和鉴权方式，然后为服务起一个首字符小写的 kebab-case 名称（命名限制见下一节）；
2. 在 `configs\` 目录按现有文件的样子创建 JSON 配置；
3. 执行 `register --conf` 注册，并在服务列表里确认新服务已经出现；
4. 用一次无副作用的工具列表或能力发现请求验证连接——在确认参数含义之前，不要执行写入、发送、删除或外部交易类的操作；
5. 重启网关 `gsudo servy-cli restart -n fastmcp-gateway`，因为网关不会自动吸纳 MCPJungle 的清单变更。

## 命名限制

工具名由「服务器名__工具名」组成，网关在注册时会校验完整工具名，规则如下：

- 名称的**首字符必须小写**，这条在 fork 与上游原版里一致；大驼峰命名（如 `ChromeDevtools`）会被拒绝。
- 连字符和嵌套超过五层的参数 schema 只在**本地 fork**（`v0.34.0-fix` wheel）里放行。如果回退到 PyPI 上游原版 0.34.0，这两类内容会被**静默拒绝**：MCPJungle 侧一切正常，网关只在日志里留一条 WARNING，客户端的表现是工具凭空消失——这种故障排查成本很高，遇到「工具不见了」时先想到这一点。
- **不得把 fastmcp-gateway 自己注册进 MCPJungle**：网关的上游就是 MCPJungle，注册形成自引用，双方互相等待工具清单，可能死锁。

## 安全与变更后检查

MCPJungle 会在注册表和 `list`、`export`、备份等命令输出中**保存或展示凭证**，因此这些输出不得复制到公共目录、聊天记录或代码仓库。每次新增、导入、更新或删除服务后，应检查：服务名称没有重复且符合命名限制；`transport` 与服务实际提供的协议一致；远程地址使用 HTTPS 且没有拼写错误；导出目录、备份目录和日志中没有新增未脱敏的凭证。如果 Key 曾被写入聊天记录、截图或不受控的导出文件，应到供应商控制台重置。

## 已知状态

penpot 当前未注册，原因是它的上游服务（本机 4401 端口）没有运行；上游恢复后，用 `configs\penpot.json` 重新注册即可。
