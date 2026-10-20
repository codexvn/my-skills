# AGENTS.md

本仓库是个人 Agent Skills 集合，通过 CC Switch 的技能仓库功能分发给各机器上的编码代理使用。

## 技能编写约定

- 每个技能一个独立目录，目录内必须存在 `SKILL.md`；目录名与 frontmatter 的 `name` 一致，使用 kebab-case。
- frontmatter 只需要 `name` 和 `description` 两个字段；`description` 要写清这个技能做什么、以及在什么场景下应该被使用（触发条件），因为代理靠它判断是否加载。
- 正文使用正式的中文书面语：句子成分完整，说明性文字不写成电报式短语；不用隐喻代替明确描述（例如不要把服务器集合称作「舰队」）；论断附带原因或出处。
- 命令和路径必须经过实际验证后才能写入；本机专属的绝对路径（如 `D:\Tools\...`）可以写，但要标明它们是本机部署事实，不是通用安装位置。
- 敏感信息（API Key、令牌、密码、内网地址）不得写入任何文件；引用带凭证的配置时只写文件路径，不写内容。

## 如何对接到 CC Switch

前提条件：仓库必须是 **公开** 的。CC Switch 的技能仓库功能通过匿名的 GitHub zip 归档下载获取内容（`https://github.com/{owner}/{repo}/archive/refs/heads/{branch}.zip`），私有仓库对该请求返回 404，SSH 协议也不在这条链路里；如果仓库是私有的，只能改公开，或改用下文的软链接方式对接本地目录。

对接步骤：

1. 打开 CC Switch 的技能页，进入「管理技能仓库」，选择「添加技能仓库」，填写 `https://github.com/codexvn/my-skills`（默认分支 main）。
2. 添加后 CC Switch 会下载仓库归档、扫描所有包含 `SKILL.md` 的目录，并把技能列在可用列表里。
3. 安装需要的技能，并选择对哪些应用启用（Claude Code、Codex 等）。
4. 安装的技能落在本地技能标准目录（`~\.agents\skills\`，或按设置使用 `~\.cc-switch\skills\`），CC Switch 再从这里同步到各启用应用的技能目录。

也可以不走仓库下载，用**软链接**分发本仓库的技能。需要注意各代理读取的是**各自的技能目录**，不存在所有代理共用的通用地址：Claude Code 读 `~\.claude\skills\`，Codex 读 `~\.codex\skills\`；`~\.agents\skills\` 是 CC Switch 的标准技能存储位置，代理本身不读它。因此完整的链接链是两级：仓库技能目录先链接到 `~\.agents\skills\`，再从那里链接到每个代理自己的技能目录（CC Switch 安装技能时内部就是这样做的）。仓库保持唯一副本，技能更新只需在本仓库执行 `git pull`，两级链接自动跟随最新提交。

在本仓库根目录执行一次以下命令即可完成两级链接（Windows 用 Junction，目录级链接且不需要管理员权限；macOS/Linux 把两处 `New-Item -ItemType Junction` 换成 `ln -s`）：

```powershell
$store = "$env:USERPROFILE\.agents\skills"
New-Item -ItemType Directory -Force -Path $store | Out-Null
$skills = Get-ChildItem -Directory | Where-Object { Test-Path (Join-Path $_.FullName 'SKILL.md') }
foreach ($s in $skills) {
    $storeLink = Join-Path $store $s.Name
    if (-not (Test-Path $storeLink)) {
        New-Item -ItemType Junction -Path $storeLink -Target $s.FullName | Out-Null
    }
    foreach ($agentDir in @("$env:USERPROFILE\.claude\skills", "$env:USERPROFILE\.codex\skills")) {
        if (-not (Test-Path $agentDir)) { continue }
        $agentLink = Join-Path $agentDir $s.Name
        if (-not (Test-Path $agentLink)) {
            New-Item -ItemType Junction -Path $agentLink -Target $storeLink | Out-Null
        }
    }
}
```

新增技能后重跑同一段命令即可补建新链接（已存在的链接自动跳过）。使用其他代理（如 Gemini CLI）时，把它的技能目录加进脚本里的 `$agentDir` 列表即可。

## 更新流程

在本仓库修改或新增技能并推送后，到 CC Switch 的技能页执行「检查更新」：CC Switch 通过 `~\.agents\.skill-lock.json` 锁文件记录每个技能的仓库归属和安装版本，检查到远端有新提交时会提示更新，确认后重新下载并覆盖本地副本，再同步到各应用。

新增技能后无需重新添加仓库：只要技能已推送，在技能页刷新可用列表即可看到新条目。
