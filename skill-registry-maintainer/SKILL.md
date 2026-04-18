---
name: skill-registry-maintainer
description: >
  本地 skill 仓库维护器。按需加载，用于扫描仓库、重建 index.json、检查配置、发现重复和缺失元数据。
  仅在用户明确提出维护 skill 仓库时使用。
trigger:
  - 用户说"更新索引"
  - 用户说"扫描仓库"
  - 用户说"重建 index.json"
  - 用户说"检查 skill-loader 配置"
  - 用户说"整理/维护 skill 仓库"
---

# Skill Registry Maintainer

本 skill 只负责维护本地 skill 仓库，不负责项目级安装。

它不是常驻 skill，而是按需加载的维护工具。

## 职责范围

### 负责

- 读取并校验 `skill-loader` 配置
- 扫描本地 skill 仓库目录
- 重建 `index.json`
- 检查重复条目
- 检查缺失的 `name` 或 `description`
- 检查索引路径和目录结构是否一致

### 不负责

- 响应普通加载请求
- 为当前项目安装 skill
- 代替 `skill-loader` 做日常路由

## 共享配置

沿用 `skill-loader` 的配置文件：

```text
~/.config/skill-loader/config.json
```

项目内建议保留两份文件：

- `config.example.json`：提交到仓库，作为模板
- `config.json`：本地实际配置，不提交到仓库

关键配置项：

- `registry.path`
- `registry.indexFile`
- `aliases`

## 仓库约定

默认扫描以下结构：

```text
<registry.path>/
├── official/
├── community/
└── custom/
```

每个 skill 目录下应至少包含：

```text
SKILL.md
```

## 维护流程

### Step 1：读取配置

读取生效配置，确认：

- 仓库目录存在
- 索引输出路径可写
- aliases 可解析

如果配置无效：

> 检测到 skill-loader 配置异常。  
> 请先修正 `registry.path` 或 `indexFile`。

### Step 2：扫描仓库

递归扫描以下目录中的 `SKILL.md`：

- `official/`
- `community/`
- `custom/`

忽略无 `SKILL.md` 的目录。

### Step 3：提取元数据

优先从 frontmatter 提取：

- `name`
- `description`

生成索引条目时补充：

- `path`：相对 `registry.path` 的路径
- `source`：根据一级目录推断，如 `official`、`community`、`custom`
- `keywords`：按以下顺序生成
  1. frontmatter 中的 `keywords`（如果有）
  2. skill 的 `name`
  3. 命中的 aliases 反向补充词

## 索引输出格式

输出到：

```text
<registry.path>/<indexFile>
```

推荐结构：

```json
{
  "version": "1.0.0",
  "updatedAt": "YYYY-MM-DD",
  "skills": []
}
```

每个条目至少包含：

```json
{
  "name": "docx",
  "source": "official",
  "path": "official/anthropics/docx",
  "keywords": ["docx"],
  "description": "Word 文档创建与编辑"
}
```

## 常见维护指令

### “更新索引”

行为：

1. 扫描仓库
2. 重建 `index.json`
3. 报告新增、删除、重复、缺失项

### “扫描仓库”

行为：

- 只检查目录与 `SKILL.md` 是否齐全
- 不强制改写索引，除非用户明确要求

### “检查配置”

行为：

- 展示当前生效配置
- 检查路径是否存在
- 检查索引文件是否可读写

### “检查索引质量”

行为：

- 查重
- 查缺失 description
- 查 path 与目录不一致的条目

## 维护原则

- 维护与加载解耦
- 扫描结果优先基于事实，不猜测缺失字段
- 仅在用户明确要求时改写 `index.json`
- 尽量保留已有人工维护的关键词，不擅自覆盖
- 报告问题时给出可执行修复建议
