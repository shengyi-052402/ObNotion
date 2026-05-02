# ObNotion

一个按“LLM 持续维护的个人 Wiki”方式组织的 Obsidian 仓库。

## 目录结构

- `Clippings/`：待处理的原始资料
- `raw/sources/`：整理后的来源笔记
- `raw/assets/`：下载的图片和附件
- `wiki/`：沉淀后的知识页
- `system/templates/`：模板文件
- `AGENTS.md`：给 Codex/其他代理的维护规则

## 核心文件

- `wiki/总览.md`：仓库总览
- `wiki/索引.md`：知识页目录
- `wiki/日志.md`：维护日志

## 使用流程

1. 把网页剪藏、文章、摘录或笔记放进 `Clippings/`。
2. 将其整理到 `raw/sources/`，并同步更新 `wiki/` 下的相关知识页。
3. 每次有实质更新时，同步维护 `wiki/索引.md` 和 `wiki/日志.md`。
