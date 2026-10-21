---
name: pushing-design-to-wiki
description: 把 markdown 文档同步、推送到 Confluence wiki 的固定管线（markdown 宏 + mermaiddiagram 宏，storage 格式直推）。何时使用：用户要求把 markdown 文档「同步到 wiki」「推送 wiki」「更新 wiki 页面」，或 wiki 页面图表渲染报错、格式错乱需要重排时。
---

# 推送 markdown 文档到 Confluence wiki

直接把 markdown 交给 wiki 服务的格式转换器不可靠：mermaid 代码块会被转成渲染不受控的占位对象，嵌套列表也会被压平。本技能用 storage 格式直推解决：正文包 `markdown` 宏，mermaid 包 `mermaiddiagram` 宏，由页面端渲染。

## 使用步骤

**1. 构建 payload**：把脚本中 `SRC` 替换为本次要推送的 markdown 源文件路径后执行（目标 page_id 与 title 从用户指令或上下文确定，可按标题用 `confluence_search` 查得）：

```python
# -*- coding: utf-8 -*-
import io, re, uuid

SRC = r'<源 markdown 文件路径>'
DST = r'D:\Tools\MCPJungle\tmp\_push_storage.xml'

t = io.open(SRC, encoding='utf-8').read()
assert ']]>' not in t

def md_macro(x):
    return ('<ac:structured-macro ac:macro-id="%s" ac:name="markdown" ac:schema-version="1">'
            '<ac:plain-text-body><![CDATA[%s]]></ac:plain-text-body></ac:structured-macro>') % (uuid.uuid4(), x)

def mdia_macro(code):
    return ('<ac:structured-macro ac:macro-id="%s" ac:name="mermaiddiagram" ac:schema-version="1">'
            '<ac:parameter ac:name=""></ac:parameter>'
            '<ac:plain-text-body><![CDATA[%s]]></ac:plain-text-body></ac:structured-macro>') % (uuid.uuid4(), code)

toc = ('<ac:structured-macro ac:macro-id="%s" ac:name="toc">'
       '<ac:parameter ac:name="printable">true</ac:parameter>'
       '<ac:parameter ac:name="style">disc</ac:parameter>'
       '<ac:parameter ac:name="maxLevel">4</ac:parameter>'
       '</ac:structured-macro>') % uuid.uuid4()

parts = re.split(r'(```mermaid\n.*?\n```)', t, flags=re.S)
out = []
for p in parts:
    if p.startswith('```mermaid\n'):
        out.append(mdia_macro(p[len('```mermaid\n'):-len('\n```')]))
    elif p:
        out.append(md_macro(p))

io.open(DST, 'w', encoding='utf-8', newline='\n').write(toc + ''.join(out))
```

**2. 推送**：调用 `confluence_update_page`，参数 `content_file` 为脚本中 `DST` 的路径、`content_format=storage`、`page_id` 与 `title` 为本次目标页面。

**3. 收尾**：删除临时文件，报告新版本号。

## 硬性约束

- 图必须用 `mermaiddiagram` 宏（含空名参数 `<ac:parameter ac:name=""></ac:parameter>`），文本用 `markdown` 宏，CDATA 一律真实换行；禁止字面 `\n` 转义（会导致整页格式错乱），禁止改回旧 `mermaid-macro` 宏名（该宏对语法挑剔）。
- payload 写在 `D:\Tools\MCPJungle\tmp\`（本机部署事实：wiki MCP 服务对可读路径做白名单校验，只认该根目录；换机部署时以实际沙箱根目录下的 tmp 子目录为准）。
- 推送前比对远端版本号，异常增长先拉历史核查，防止覆盖他人在 wiki 上的编辑。
