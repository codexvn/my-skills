---
name: servy-usage
description: Servy Windows 服务包装器的用法（把应用托管为原生服务、增删服务、升级 Servy）。何时使用：需要把 MCPJungle、fastmcp-gateway、SSEInspector 或新应用做成开机自启的 Windows 服务、查询/重启/卸载服务、升级 Servy 本体、或排查服务起不来的问题时。
---

# Servy 用法

Servy 是 Windows 服务包装器，把 MCPJungle、fastmcp-gateway、SSEInspector 等应用托管为**原生 Windows 服务**：自动启动、崩溃健康恢复、日志轮转。资料目录 `D:\Tools\Servy`（不是程序本体）。

## 部署结构

| 位置 | 内容 |
|---|---|
| `C:\Program Files\Servy` | 程序本体（Servy 桌面程序、Servy Manager、servy-cli、PowerShell 模块），已入系统 PATH |
| `C:\ProgramData\Servy` | 运行数据：服务宿主 exe（Servy.Service*.exe、Servy.Restarter.exe、handle64.exe）、Servy.db、AES 密钥、自身日志 |
| `D:\Tools\Servy` | 服务定义（configs）、被管理应用日志（logs）、配置导出（exports）、升级前备份（backups） |

**安装方式约束**：Servy 10.x 安全加固拒绝任何穿越 junction/符号链接的可执行路径，**不要用 Scoop 等 junction 布局的包管理器安装**（2026-10-01 已因此从 Scoop 迁到官方安装器）。

## 服务生命周期（servy-cli）

```powershell
servy-cli start -n <服务名>
servy-cli status -n <服务名>
servy-cli stop -n <服务名>
servy-cli restart -n <服务名>
servy-cli uninstall -n <服务名>   # 不删 configs 源文件
```

当前托管：`MCPJungle`、`fastmcp-gateway`、`SSEInspector`。被管理应用的标准输出日志按各配置的 `StdoutPath`/`StderrPath` 写入 `D:\Tools\Servy\logs\<服务名>\`。

## 新增一个被托管应用

1. 在 `configs\` 复制现有 `<服务名>.servy.json` 修改，核心字段：
   - `Name` / `DisplayName` / `Description`：服务名（全局唯一、不含空格）与显示信息；
   - `ExecutablePath` / `StartupDirectory` / `Parameters`：被托管程序、工作目录、启动参数；
   - `StartupType`：2 = 自动启动；`Priority`：2 = 正常；
   - `StdoutPath` / `StderrPath`：日志，统一放 `D:\Tools\Servy\logs\<服务名>\`；
   - `EnableSizeRotation` / `RotationSize` / `MaxRotations`：日志轮转；
   - `EnableHealthMonitoring` 等健康恢复项参照现有文件；
   - `EnvironmentVariables`：分号分隔字符串注入环境变量（fastmcp-gateway 的全部配置就走这里）。
2. 管理员权限注册并安装（宿主 exe 自动部署到 `C:\ProgramData\Servy`）：

   ```powershell
   servy-cli import --config json --path D:\Tools\Servy\configs\<服务名>.servy.json --install
   ```

3. `servy-cli start -n <服务名>` 启动并验证。

**改配置（含环境变量）后必须重新 import 再 restart**，直接 restart 不会重读配置文件。

## 升级 Servy 本体

1. [GitHub Releases](https://github.com/aelassas/servy/releases) 下载 `servy-x.y-x64-installer.exe`（自包含 x64 版，与当前构建保持一致，不用 net48 版）；
2. 先备份 `C:\ProgramData\Servy\db` 与 `C:\ProgramData\Servy\security` 到 `D:\Tools\Servy\backups\pre-<版本>-<日期>\`；
3. 直接运行安装器覆盖安装（无需卸载；服务注册与数据库自动保留，装完被管服务自动启动）；
4. 验证：`servy-cli status -n MCPJungle` 等返回 Running、各网关可访问；
5. 跨大版本先读 Release Notes 确认无破坏性变更。

## 安全

配置中的令牌/密钥只允许存在于 `configs\` 受保护文件；导出文件先检查有无敏感值再决定保留位置；升级前备份至少留两份，新版稳定后再清理。
