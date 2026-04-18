---
name: skill-loader
description: >
  项目级 skill 加载器。常驻全局，负责识别“加载”与“维护”两类意图。
  对加载意图：读取配置与本地索引，命中则安装到当前项目；未命中则回退远程搜索。
  对维护意图：不直接维护，而是转交给 skill-registry-maintainer。
  关键词：加载 skill、安装 skill、需要XX功能、有没有 skill 能...
trigger:
  - 用户提到"加载/安装某个 skill"
  - 用户表达"帮我做X"但当前项目缺少对应 skill
  - 用户问"有没有 skill 能..."
  - 用户说"更新索引/扫描仓库/维护 skill 仓库"时，识别为维护意图并转交 maintainer
---

# Skill Loader

项目级 skill 的常驻加载器。

它的职责只有两件事：

1. 识别用户当前是要“加载 skill”还是“维护 skill 仓库”
2. 在明确是加载意图时，完成项目级 skill 的发现与安装

## 职责边界

### 负责

- 识别加载意图
- 读取配置文件
- 读取本地 skill 仓库索引
- 匹配本地可用 skill
- 将命中的 skill 安装到当前项目
- 本地未命中时执行远程搜索 fallback

### 不负责

- 扫描仓库
- 重建 `index.json`
- 分类整理 skill
- 修复仓库元数据
- 批量同步 skill
- 修改配置内容

这些维护工作统一交给按需加载的 `skill-registry-maintainer`。

## 配置

skill-loader 优先读取用户配置：

```text
~/.config/skill-loader/config.json
```

Windows：

```text
C:\Users\<username>\.config\skill-loader\config.json
```

### 配置文件结构

```json
{
  "version": "1.0.0",
  "registry": {
    "path": "E:/skills-registry",
    "indexFile": "index.json"
  },
  "behavior": {
    "installScope": "project",
    "autoSearchRemote": true,
    "autoSearchWeb": false,
    "confirmBeforeInstall": false
  },
  "fallback": {
    "remoteRegistry": "skills.sh",
    "searchCommand": "npx skills find"
  },
  "aliases": {
    "word": "docx",
    "excel": "xlsx",
    "ppt": "pptx"
  }
}
```

### 配置项说明

| 配置项 | 说明 |
|--------|------|
| `registry.path` | 本地 skill 仓库根目录 |
| `registry.indexFile` | 索引文件名 |
| `behavior.installScope` | 安装范围，默认 `project` |
| `behavior.autoSearchRemote` | 本地未命中时是否自动搜索远程 |
| `behavior.autoSearchWeb` | 远程未命中后是否继续网页搜索 |
| `behavior.confirmBeforeInstall` | 安装前是否先征求确认 |
| `fallback.remoteRegistry` | 远程 registry 标识 |
| `fallback.searchCommand` | 远程搜索命令 |
| `aliases` | 功能关键词到 skill 名称的映射 |

### 配置加载优先级

1. `~/.config/skill-loader/config.json`
2. `./skill-loader/config.json`（项目本地实际配置，建议由 `config.example.json` 复制生成，不提交）
3. `./skill-loader/config.example.json`（示例配置模板）
4. 内置默认值

## 意图识别

### 加载意图

以下情况视为加载意图：

- “帮我加载 docx skill”
- “我要处理 Word 文档”
- “有没有做 PPT 的 skill”
- “给这个项目装一个前端测试 skill”

### 维护意图

以下情况视为维护意图：

- “更新索引”
- “扫描仓库”
- “重建 index.json”
- “检查 skill 仓库配置”
- “整理一下 registry”

如果识别为维护意图，不要自己执行维护逻辑，直接转交：

> 这是 skill 仓库维护操作，不由 skill-loader 直接处理。  
> 请加载或调用 `skill-registry-maintainer` 继续执行。

## 加载流程

### Step 1：读取配置

优先读取用户配置文件，获取：

- `registry.path`
- `registry.indexFile`
- `behavior.confirmBeforeInstall`
- `behavior.autoSearchRemote`
- `aliases`

如果配置文件不存在：

> 检测到 skill-loader 配置文件不存在，正在使用默认配置。  
> 如需自定义，请创建 `~/.config/skill-loader/config.json`。

### Step 2：解析需求

从用户输入中提取目标 skill 名称或功能关键词。

示例：

- “Word 文档” → `word` → `docx`
- “Excel 表格” → `excel` → `xlsx`
- “PPT” → `ppt` → `pptx`

优先使用 `aliases` 做一次归一化。

### Step 3：读取本地索引

索引路径：

```text
<registry.path>/<registry.indexFile>
```

如果索引不存在：

> 本地 skill 仓库索引不存在。  
> 请使用 `skill-registry-maintainer` 扫描仓库并重建索引。

### Step 4：匹配本地 skill

匹配优先级：

1. `name` 精确匹配
2. `aliases` 映射后匹配
3. `keywords` 包含匹配
4. `description` 模糊匹配

结果处理：

| 结果 | 行为 |
|------|------|
| 命中 1 个 | 直接进入安装 |
| 命中多个 | 列出候选，让用户选择 |
| 未命中 | 进入远程 fallback |

## 项目级安装

安装命令：

```bash
npx skills add <registry.path>/<skill.path> -y
```

规则：

- 默认项目级安装，不加 `-g`
- 若 `confirmBeforeInstall=true`，先询问用户确认
- 安装成功后明确告知已装到当前项目

## 远程 fallback

仅在以下条件满足时触发：

- 本地索引未命中
- `behavior.autoSearchRemote=true`

搜索命令：

```bash
npx skills find <关键词> -y
```

如果远程找到候选：

> 本地仓库没有找到匹配 skill，但远程 registry 中存在候选。  
> 可以继续安装远程 skill，或先收录到本地仓库再安装。

如果远程也没有命中：

> 本地仓库和远程 registry 都没有找到匹配 skill。  
> 可以尝试更宽泛的关键词，或手动创建新 skill。

## 设计原则

- 全局常驻的只有“加载能力”
- “维护能力”必须按需加载，避免日常上下文膨胀
- 本地仓库优先，远程搜索兜底
- 默认项目级安装，不污染全局
- 配置驱动，不硬编码用户环境
